# etcd and Control Plane Internals

## 1. etcd Raft Internals

etcd is a replicated state machine built on Raft. Every object in the cluster — Pods, Secrets, CRDs, Leases — is a key in etcd's flat keyspace, versioned with an MVCC (multi-version concurrency control) store. Kubernetes itself has no idea what "consensus" means; it delegates all of that to etcd and only talks to it through gRPC.

### 1.1 Leader election and log replication

Raft nodes are always in one of three states: `Follower`, `Candidate`, `Leader`.

```
        timeout, no heartbeat            majority votes granted
Follower ------------------------> Candidate ------------------------> Leader
   ^                                    |                                 |
   |            higher term seen        |                                 |
   +------------------------------------+---------------------------------+
                                  (steps down)
```

- All members start as followers.
- If a follower doesn't hear a heartbeat from the leader within its **election timeout** (randomized, typically 1000-5000ms in etcd, jittered per node to avoid split votes), it becomes a candidate, increments its term, votes for itself, and requests votes from peers.
- A candidate becomes leader only if it gets votes from a **majority** of the cluster (quorum).
- Once elected, the leader sends periodic heartbeats (empty `AppendEntries` RPCs) at the **heartbeat interval** (default 100ms) to maintain authority and prevent followers from timing out.

Write path once a leader exists:

1. Client (kube-apiserver) sends a write via gRPC to whichever etcd member it's connected to.
2. If that member isn't the leader, it transparently forwards to the leader (etcd handles this; the apiserver doesn't need to know who the leader is).
3. Leader appends the entry to its local WAL (write-ahead log) and replicates `AppendEntries` to all followers in parallel.
4. **The write is only committed — and only acknowledged back to the client — once a majority of members (including the leader) have persisted the entry to their WAL.** This is the critical detail: etcd does not wait for *all* nodes, only a majority. A slow or partitioned minority never blocks writes.
5. Once committed, the entry is applied to the in-memory MVCC store (bbolt-backed on disk) and becomes readable.

This means every write incurs at least one network round trip to a majority of members — this is why etcd latency is extremely sensitive to disk fsync latency and network RTT between members. Cross-AZ or cross-region etcd clusters are a common source of high `apiserver` write latency because every write blocks on the slowest member in the majority set.

Check current leader and Raft state:

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --cluster -w table
```

```
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.0.1.10:2379     | 8211f1d0f64f3269 |  3.5.9  |   58 MB |      true |      false |        14 |     982341 |             982341 |        |
| https://10.0.1.11:2379     | 91bc3c398fb3c146 |  3.5.9  |   58 MB |     false |      false |        14 |     982341 |             982341 |        |
| https://10.0.1.12:2379     | fd422379fda50e48 |  3.5.9  |   58 MB |     false |      false |        14 |     982341 |             982341 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

`RAFT TERM` increasing between calls without a corresponding leader change in logs is a strong signal of network flakiness triggering repeated elections — check for packet loss / high latency between etcd peer endpoints (`:2380`), not just client ports (`:2379`).

### 1.2 Why cluster size must be odd, and the quorum math

Quorum = `floor(n/2) + 1`. Fault tolerance (nodes you can lose and still have quorum) = `n - quorum`.

| Cluster size (n) | Quorum required | Nodes you can lose | Notes |
|---|---|---|---|
| 1 | 1 | 0 | No fault tolerance at all |
| 2 | 2 | 0 | Worse than 1 — any node loss halts writes, and you paid for 2x hardware |
| 3 | 2 | 1 | Standard production minimum |
| 4 | 3 | 1 | **Same fault tolerance as 3**, but costs more disk/network/CPU for replication — never do this |
| 5 | 3 | 2 | Better fault tolerance than 3, common for large/critical clusters |
| 6 | 4 | 2 | Same tolerance as 5, wasted resources — never do this |
| 7 | 4 | 3 | Higher tolerance, but write latency degrades (more members to replicate to before majority ack) |

The takeaway every junior misses: **going from 3 to 4 buys you nothing** — you still only tolerate 1 failure, but now you need agreement from 3 nodes instead of 2, so writes get *slower*, not safer. Always keep etcd cluster size odd. Production standard is 3 (small/medium clusters) or 5 (large, multi-AZ, or clusters that need to tolerate simultaneous AZ + node failure).

### 1.3 Quorum loss

Quorum loss = you no longer have a majority of members alive and reachable. Effects:

- The cluster **cannot commit new writes** — no leader can be (re)elected, or if a leader exists it can't get AppendEntries acknowledged by a majority.
- **Reads may still work** in a degraded, potentially stale way if you allow serializable (non-linearizable) reads, but by default `kube-apiserver` uses linearizable reads against etcd, which also require quorum. Practically: the whole cluster becomes **read AND write unavailable** for anything going through the apiserver — `kubectl get pods` hangs or times out.
- `kubectl` symptoms: requests time out, apiserver logs show `etcdserver: request timed out` or `context deadline exceeded`, and `/healthz` on the apiserver starts failing on the etcd check.

Example: 3-node cluster, 2 nodes die (disk corruption, VM deleted, etc). You have 1 surviving member — no quorum possible ever again with just that member. This is quorum loss.

### 1.4 Recovering from quorum loss

**Case A: some members are dead but their data disks / VMs can be brought back or replaced (preferred path — no data loss)**

If you can remove the dead members and add fresh ones one at a time, keeping quorum among the survivors at every step:

```bash
# from a surviving, healthy member
etcdctl member list -w table

# remove the dead member by its member ID
etcdctl member remove 91bc3c398fb3c146

# add a replacement member (must do this BEFORE starting the new etcd process)
etcdctl member add etcd-2 --peer-urls=https://10.0.1.13:2380

# then start etcd on the new node with --initial-cluster-state=existing
# and the exact --initial-cluster list etcdctl member add prints out
```

This only works while you still have quorum among the *remaining* members after each removal. If losing one more node would drop you below quorum, do removals/adds one at a time, verifying health between each step.

**Case B: quorum is already lost (majority of members gone, e.g., 2 of 3 disks destroyed) — no clean path, must force**

Two options, in order of preference:

1. **Restore from the most recent snapshot onto a brand-new single-node cluster**, then re-add members to grow back to 3/5 (see backup/restore section below). This is safe and deterministic — you get exactly the state as of the last snapshot.
2. **`--force-new-cluster`** on the single surviving member if no recent snapshot exists and you need to salvage whatever data that one surviving member has:

```bash
etcd --force-new-cluster \
  --data-dir=/var/lib/etcd \
  --name=etcd-0 \
  --initial-cluster=etcd-0=https://10.0.1.10:2380 \
  --listen-peer-urls=https://10.0.1.10:2380 \
  --listen-client-urls=https://10.0.1.10:2379 \
  --advertise-client-urls=https://10.0.1.10:2379
```

`--force-new-cluster` **unilaterally declares the local member the leader of a brand-new single-member cluster**, discarding the Raft membership list and forcing a new cluster ID. Dangers:

- Any writes that were in-flight/uncommitted on the other (now-dead) members are permanently lost, and there is no way to know what was lost.
- If you run `--force-new-cluster` on more than one surviving member independently (e.g., trying it on two nodes at once thinking it's "safer"), you create **two divergent clusters with the same data lineage but different cluster IDs** — a split-brain that is extremely painful to unwind, because both will think they're authoritative.
- After forcing, you must immediately re-add members properly and take a fresh snapshot — treat the forced node as radioactive until scaled back to a real quorum.

This flag exists purely for emergency, last-resort recovery, not as a routine tool. Never use it if a good snapshot exists — always prefer snapshot restore.

---

## 2. etcd Backup and Restore

### 2.1 Taking a snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

Notes:

- Use the `healthcheck-client` cert (or `apiserver-etcd-client` cert) — not the peer cert — for client operations like snapshotting.
- `snapshot save` streams a consistent point-in-time copy over the client connection; it doesn't require stopping etcd or the apiserver.
- On a kubeadm cluster, endpoints/certs are visible in `/etc/kubernetes/manifests/etcd.yaml` if you're unsure which paths to use.

### 2.2 Verifying a snapshot

Never trust a backup you haven't checked:

```bash
etcdctl snapshot status /backup/etcd-snapshot-20260718020000.db -w table
```

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 8a3f2c1d |    58213 |       1847 |      58 MB |
+----------+----------+------------+------------+
```

A zero-byte file, a `TOTAL KEYS` of near-zero, or an outright checksum error here means the backup is useless — catch this in your backup job, not during a real incident.

### 2.3 Restoring a snapshot

The gotchas that catch almost everyone the first time:

- `snapshot restore` **does not restore in place**. It builds a brand-new data directory from the snapshot. You must point etcd at that new directory afterward.
- You must supply `--name`, `--initial-cluster`, and `--initial-advertise-peer-urls` matching your intended (possibly new) cluster topology — restore fabricates a fresh cluster membership, it does not "remember" the old one.
- Restoring on each member independently with **consistent `--initial-cluster` and a shared `--initial-cluster-token`** is required, or members won't agree they belong to the same cluster.

```bash
etcdctl snapshot restore /backup/etcd-snapshot-20260718020000.db \
  --data-dir=/var/lib/etcd-restored \
  --name=etcd-0 \
  --initial-cluster="etcd-0=https://10.0.1.10:2380,etcd-1=https://10.0.1.11:2380,etcd-2=https://10.0.1.12:2380" \
  --initial-cluster-token=etcd-cluster-restored-1 \
  --initial-advertise-peer-urls=https://10.0.1.10:2380
```

Repeat on each node with its own `--name` / `--initial-advertise-peer-urls`, but identical `--initial-cluster` and `--initial-cluster-token`.

**kubeadm-managed clusters specifically:**

1. Stop the kubelet's static pod for etcd (or the whole kubelet, since static pods get recreated automatically) so it doesn't fight you:
   ```bash
   mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
   ```
2. Restore the snapshot to a new data dir (e.g., `/var/lib/etcd-restored`), as above, once per control-plane node.
3. Update `/etc/kubernetes/manifests/etcd.yaml` (or `/tmp/etcd.yaml.bak` before moving it back) so `--data-dir` and the volume `hostPath` point at the new restored directory, and any `--initial-cluster-token` args match.
4. Move the manifest back into `/etc/kubernetes/manifests/` — kubelet will pick it up and start the static pod against the restored data.
5. Verify: `etcdctl endpoint health`, then `kubectl get nodes`, `kubectl get pods -A` to confirm the apiserver is reading real data again.
6. **This is a whole-cluster operation.** A single-member restore-in-isolation followed by adding old peers back will not work cleanly because the restored member has a new cluster ID that the untouched peers don't recognize — you generally restore all members from the same snapshot together, or restore one to a single-node cluster and then `member add` fresh peers.

### 2.4 Backup automation

Never store the snapshot on the same node/disk as etcd's data — that protects you from application-level corruption but not from disk/VM/AZ loss, which is the failure mode backups are actually for.

CronJob pattern (runs on a schedule, ships to remote object storage):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-snapshot
  namespace: kube-system
spec:
  schedule: "0 * * * *"     # hourly
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
          - name: etcd-snapshot
            image: registry.k8s.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d%H%M%S).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
                --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
              # then push /backup/*.db to S3/GCS/etc, and prune local + remote retention
              aws s3 cp /backup/etcd-*.db s3://company-etcd-backups/$(hostname)/ 
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            emptyDir: {}
```

In practice most teams run this as a systemd timer or cron job directly on control-plane nodes (simpler cert/hostPath handling than in-cluster, and it survives even if the cluster's own scheduling is broken) shipping to S3/GCS/MinIO with lifecycle rules for retention (e.g., hourly for 48h, daily for 30 days).

### 2.5 Disaster recovery runbook: total loss of all etcd nodes

Scenario: all 3 control-plane nodes suffered disk failure (bad SAN firmware update, simultaneous AZ outage, whatever). Worker nodes and running Pods are still up (kubelets keep running workloads independently of the control plane being reachable), but nothing can be scheduled, scaled, or reconciled, and no new state can be read/written.

1. **Do not touch worker nodes.** Running Pods keep serving traffic via kube-proxy/CNI even with the control plane fully down — this buys time.
2. Confirm the actual blast radius: `etcdctl endpoint health` against each control-plane node's etcd; check `systemctl status kubelet` and container runtime on each; check disk health (`smartctl`, cloud provider console) before assuming full loss.
3. Locate the most recent verified-good snapshot (from the automated job above) and confirm its `snapshot status` output looks sane (non-zero keys, expected size ballpark vs historical).
4. Provision fresh control-plane node(s) — same Kubernetes/etcd version as the lost cluster. Version mismatch on restore is a common self-inflicted second incident.
5. Restore the snapshot to a **single new node** first (simplest, lowest-risk path — grow to 3 afterward):
   ```bash
   etcdctl snapshot restore /backup/etcd-latest.db \
     --data-dir=/var/lib/etcd \
     --name=etcd-0 \
     --initial-cluster=etcd-0=https://<new-node-ip>:2380 \
     --initial-cluster-token=etcd-cluster-dr-restore \
     --initial-advertise-peer-urls=https://<new-node-ip>:2380
   ```
6. Re-point the static pod manifest / systemd unit at the restored data-dir, start etcd, confirm `etcdctl endpoint health` is green and `member list` shows exactly the one member.
7. Bring up kube-apiserver, kube-controller-manager, kube-scheduler pointed at this single-node etcd. Confirm `kubectl get nodes`, `kubectl get pods -A` return the pre-incident state (as of last snapshot — anything written after the snapshot is gone, note this for stakeholders).
8. `etcdctl member add` two more nodes one at a time, bringing new etcd processes up with `--initial-cluster-state=existing`, verifying `member list` and `endpoint status --cluster` show healthy quorum after each addition.
9. Reconcile drift: anything created/deleted between the snapshot time and the incident is gone from etcd but may still exist as running Pods on workers (or vice versa — objects that existed in the snapshot but whose backing infra was since torn down). Expect kubelets to report back and controllers to reconcile most of this automatically, but manually verify anything stateful (PVCs, in-flight Jobs, external DNS/LB records created by controllers).
10. Post-incident: rotate any credentials that were exposed during the recovery process, verify backup cadence covered enough recent history (if last snapshot was 2 hours stale, question whether hourly is frequent enough), and write the postmortem.

---

## 3. API Server Request Flow

Every request to `kube-apiserver` — from `kubectl`, a controller, a webhook, another apiserver in aggregation — passes through the same ordered pipeline before it ever touches etcd.

```
 ┌────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌────────────┐     ┌──────────┐
 │   Client   │────▶│ Authentication │────▶│  Authorization    │────▶│  Admission  │────▶│   etcd   │
 │ (kubectl,  │     │ (who are you?)│     │ (are you allowed?)│     │  Control    │     │  (write) │
 │ controller)│     └───────────────┘     └──────────────────┘     └────────────┘     └──────────┘
 └────────────┘            │                       │                      │                  │
                            │ fail                  │ fail                 │ fail             │ conflict
                            ▼                       ▼                      ▼                  ▼
                       401 Unauthorized       403 Forbidden        4xx admission          409 Conflict
                                                                     denied message      (resourceVersion
                                                                                            mismatch)
```

### 3.1 Authentication

Multiple authenticators are chained; the first one to positively identify the request wins (they're tried in order until one succeeds or all fail).

- **Client certificates**: the cert's `CN` (Common Name) becomes the Kubernetes **username**, and `O` (Organization) fields become **group memberships**. This is why kubeadm-generated admin certs have `O=system:masters` — that group is bound to `cluster-admin` by a built-in ClusterRoleBinding, which is why possessing that cert is equivalent to full admin regardless of any RBAC you write later.
- **Bearer tokens**: static token files (deprecated), service account tokens (JWTs, validated against the API server's configured issuer/JWKS or the legacy in-cluster CA), or bootstrap tokens.
- **OIDC**: apiserver validates a JWT ID token against `--oidc-issuer-url`, `--oidc-client-id`, checking signature against the issuer's JWKS endpoint, then maps `--oidc-username-claim` (often `email` or `sub`) and `--oidc-groups-claim` to Kubernetes username/groups. Note the apiserver never talks to the OIDC provider at request time for token validation beyond fetching/caching JWKS — the actual login flow happens client-side (`kubectl` plugins like `kubelogin`), the apiserver only verifies the token it's handed.
- **Webhook token authentication**: apiserver POSTs a `TokenReview` to an external service, which returns whether the token is valid and what user/groups it maps to. Used for integrating with custom SSO/token systems.

Failure mode: if **no** authenticator can identify the request (bad/expired cert, garbage token, no credentials at all on a request that needs them), the result is **`401 Unauthorized`**. From `kubectl`:

```
error: You must be logged in to the server (Unauthorized)
```

### 3.2 Authorization

Once identity (user + groups) is established, authorization decides if that identity may perform the specific verb/resource/namespace combination. Modes are also chained (`--authorization-mode=Node,RBAC` is standard), each queried in order; **the request is allowed as soon as any authorizer says yes**, denied only if all say no (or one explicitly denies, for webhook mode).

- **RBAC** (the one that matters day to day): evaluates all `Role`/`ClusterRole` bindings for the user's identity and every group it belongs to, unioning permissions. RBAC has no explicit "deny" — only allow rules exist, so a user's effective permission is the union of every Role/ClusterRole bound to them or their groups, cluster-wide bindings included. There is no way to carve out an exception via RBAC alone (e.g., "allow all pods except in namespace X" needs a separate mechanism like an admission webhook, not RBAC).
- **ABAC**: policy file with JSON lines evaluated per request, largely legacy — requires apiserver restart to change policy, essentially unused in modern clusters in favor of RBAC.
- **Webhook authorizer**: apiserver POSTs a `SubjectAccessReview` to an external service (common in platforms doing centralized policy, e.g., OPA-based authorization services) which returns allow/deny.
- **Node authorizer**: special-purpose, restricts each kubelet to only reading/writing objects related to itself (its own Node object, Pods scheduled to it, etc.) — this is what stops a compromised kubelet from reading arbitrary Secrets cluster-wide.

Failure mode: **`403 Forbidden`**, with a message naming the exact missing permission:

```
Error from server (Forbidden): pods is forbidden: User "jdoe" cannot list resource "pods"
in API group "" in the namespace "production"
```

The distinction that matters for on-call triage: **401 means "I don't know who you are"** (cert expired, token garbage, no credential supplied) — the fix is on the credential/identity side. **403 means "I know exactly who you are, and you're not allowed to do this"** — the fix is an RBAC binding, not a credential. Conflating these wastes debugging time — check `kubectl auth can-i <verb> <resource>` before assuming it's a cert problem.

```bash
kubectl auth can-i delete deployments --as=jdoe -n production
kubectl auth can-i delete deployments --as=system:serviceaccount:production:ci-bot -n production
```

### 3.3 Admission control

Only requests that pass authn+authz reach admission — this is the stage that can **mutate** the object (defaulting, injection) and/or **validate** it against policy, after authorization but before persistence.

Ordering, precisely:

1. **Built-in admission controllers run interspersed** in a fixed compiled-in order set by `--enable-admission-plugins` (e.g., `NamespaceLifecycle`, `LimitRanger`, `ServiceAccount`, `NodeRestriction`, `ResourceQuota`, `DefaultStorageClass`, `PodSecurity`, etc.) — this order is not configurable per-cluster beyond enabling/disabling plugins.
2. **All mutating webhooks run before all validating webhooks**, always — this is a hard architectural rule, not a config option. The rationale: mutations (defaulting, sidecar injection, label injection) need to happen before anything validates the final shape of the object, otherwise validation would validate a stale, pre-mutation object.
3. Within the set of mutating webhooks, and separately within the set of validating webhooks, **multiple webhooks matching the same request are called sequentially, but the relative order across webhooks from different `MutatingWebhookConfiguration`/`ValidatingWebhookConfiguration` objects is not strictly guaranteed** beyond alphabetical-by-name in older versions — never build logic that depends on webhook A running before webhook B unless they're phases of the same admission chain (mutating vs validating) by design.
4. Mutating webhooks can modify the object; each mutation is re-validated for schema correctness before moving to the next webhook.
5. After all mutating webhooks, the (possibly heavily modified) object goes through the built-in validating admission controllers, then all validating webhooks, none of which may mutate anything at this stage.

Failure mode: **admission-denied requests return an HTTP status set by the webhook** (commonly 400 or 422, sometimes deliberately 403), with the message coming verbatim from the webhook's `AdmissionReview.response.status.message`:

```
Error from server (Forbidden): error when creating "deploy.yaml": admission webhook
"validation.gatekeeper.sh" denied the request: [container-must-have-limits] container
"app" has no resource limits set
```

Note the webhook name in the message — that's your entry point for finding which `ValidatingWebhookConfiguration`/policy is responsible.

---

## 4. Admission Controllers in Depth

### 4.1 Important built-in controllers

- **`NamespaceLifecycle`**: prevents creating objects in a namespace that's being terminated, and prevents deleting the built-in `default`/`kube-system`/`kube-public` namespaces.
- **`LimitRanger`**: enforces `LimitRange` objects — applies default requests/limits to containers that don't specify them, and rejects containers whose requests/limits fall outside the configured min/max.
- **`ResourceQuota`**: enforces `ResourceQuota` objects per namespace — tracks aggregate resource consumption (CPU/memory/object counts) and rejects requests that would exceed the quota. Runs late (after most other mutating steps) so it counts the final, fully-defaulted resource values.
- **`DefaultStorageClass`**: if a `PersistentVolumeClaim` has no `storageClassName`, this stamps in whichever `StorageClass` is marked default (`storageclass.kubernetes.io/is-default-class: "true"`). If two StorageClasses are both marked default, this controller picks the most recently created one and PVC creation proceeds with a possibly-unexpected class — a common footgun after adding a second default by accident.
- **`PodNodeSelector`**: enforces namespace-level node selector constraints via annotation, restricting which nodes Pods in a namespace can be scheduled to, independent of what the Pod spec itself requests.
- **`NodeRestriction`**: restricts kubelets (authenticated as `system:node:<name>`) to only modifying their own Node object and Pods bound to them — critical security boundary, not just a convenience.
- **`PodSecurity`** (replaced PodSecurityPolicy): enforces Pod Security Standards (`privileged`/`baseline`/`restricted`) at the namespace label level (`pod-security.kubernetes.io/enforce`).
- **`ServiceAccount`**: auto-mounts the default service account and its token into Pods that don't specify one.

### 4.2 Dynamic admission webhooks

Structure of a `ValidatingWebhookConfiguration`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gatekeeper-validating-webhook
webhooks:
- name: validation.gatekeeper.sh
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/admit
    caBundle: <base64 CA cert>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
    scope: "Namespaced"
  matchPolicy: Equivalent      # match this rule against equivalent API versions too (e.g. apps/v1beta1 requests treated as apps/v1)
  sideEffects: None            # must declare whether calling this webhook has side effects outside the admission response
  timeoutSeconds: 5
  admissionReviewVersions: ["v1"]
  failurePolicy: Fail
```

Key fields explained:

- **`matchPolicy: Equivalent`** vs `Exact`: `Equivalent` means the rule also matches requests hitting a different-but-convertible API version of the same resource — important when clients hit an older API version than the one you wrote the rule against.
- **`sideEffects`**: `None` (safest, allows dry-run to skip calling the webhook), `NoneOnDryRun`, or `Some` — kube-apiserver refuses `--dry-run` requests against webhooks that declare side effects unless they explicitly support `NoneOnDryRun`.
- **`timeoutSeconds`**: max 30s (usually configured much lower, 2-10s) — if the webhook doesn't respond in time, this is treated as a failure and `failurePolicy` decides what happens next.

### 4.3 `failurePolicy: Fail` vs `Ignore` — the real production landmine

- **`Ignore`**: if the webhook is unreachable or times out, the request proceeds as if the webhook didn't exist. Safe default for non-critical policy webhooks, but means your policy can be silently bypassed during an outage of the webhook itself.
- **`Fail`**: if the webhook is unreachable or times out, the **request is rejected**. This is what you want for security-critical enforcement (you don't want a mutating-webhook outage to let unvalidated Pods slip through) — but it has a sharp edge.

**The classic self-inflicted outage**: a team deploys OPA Gatekeeper (or Istio's sidecar-injector) with `failurePolicy: Fail` and a `rules` block matching `pods` cluster-wide with no `namespaceSelector` exclusion. Then:

- The Gatekeeper/webhook deployment itself gets rolled out, hits a bad image, `CrashLoopBackOff`, or all its replicas get evicted during a node drain.
- Now **every single Pod creation in the entire cluster** — including the replacement Pod for the webhook's own deployment — blocks on a webhook call that can never succeed, because the webhook backend is down.
- kube-apiserver returns admission-denied (timeout) for **every** matching Pod create/update, cluster-wide: new deployments stall, HPA-driven scale-ups fail, even the webhook's own new Pod can't schedule to fix itself — a total self-inflicted deadlock.

Mitigations used in real production setups:

1. **Exclude the webhook's own namespace** (and typically `kube-system`) from its own `rules`/`namespaceSelector`, so the webhook can always heal itself:
   ```yaml
   namespaceSelector:
     matchExpressions:
     - key: kubernetes.io/metadata.name
       operator: NotIn
       values: ["gatekeeper-system", "kube-system"]
   ```
2. Run the webhook backend with enough replicas + PodDisruptionBudget that a single node drain/upgrade can't take it fully down.
3. Keep `timeoutSeconds` low (2-5s) so a hung webhook fails fast rather than making every kubectl apply hang for 30s.
4. Have a documented break-glass procedure: `kubectl delete validatingwebhookconfiguration <name>` (or patch `failurePolicy` to `Ignore`) as an emergency unblock — requires `cluster-admin`, which is why access to delete webhook configurations should be tightly held and audited, since it's both a critical safety valve and a way to disable all policy enforcement at once.

---

## 5. Controller Manager and Reconciliation

### 5.1 Level-triggered, not edge-triggered

Kubernetes controllers are deliberately **level-triggered**: a reconcile loop, when invoked, looks at **current desired state vs current observed state** and drives toward the desired state — it does not care what specific event caused it to run, and does not try to process a queue of historical events in order.

Contrast:

- **Edge-triggered** ("a Pod just died, decrement replica count and create one more") is fragile — if the event is missed (controller restart, watch disconnect, network blip), the system permanently diverges from desired state with no self-correction.
- **Level-triggered** ("desired replicas = 3, observed running Pods = 2, therefore create 1") self-heals regardless of *why* it's out of sync — missed events, controller restarts, stale caches all get corrected on the next reconcile, because the loop always re-derives the delta from current state rather than replaying history.

This is why writing your own controller/operator and accidentally coding edge-triggered logic (e.g., reacting only to `ADDED` events and ignoring periodic resyncs) is a classic bug — it works in testing and silently drifts in production after any missed watch event.

### 5.2 Informers, List-Watch, and local cache

Controllers never make live API calls to etcd (via the apiserver) for every reconcile — that would fall over immediately at any scale. Instead:

```
                  ┌─────────────────────────────────────────────────────┐
                  │                     kube-apiserver                    │
                  └───────────────┬───────────────────────┬───────────────┘
                       1. List (full sync)         2. Watch (stream of deltas)
                                  │                        │
                                  ▼                        ▼
                       ┌─────────────────────────────────────────┐
                       │              Reflector                    │
                       │  (writes Add/Update/Delete into a queue)  │
                       └───────────────────┬───────────────────────┘
                                           ▼
                       ┌─────────────────────────────────────────┐
                       │            DeltaFIFO queue                │
                       └───────────────────┬───────────────────────┘
                                           ▼
                       ┌─────────────────────────────────────────┐
                       │     Indexer / local thread-safe cache      │
                       │   (this is what controllers actually read) │
                       └───────────────────┬───────────────────────┘
                                           ▼
                       ┌─────────────────────────────────────────┐
                       │     Controller's reconcile function        │
                       │  (reads from Indexer, writes back via API) │
                       └─────────────────────────────────────────────┘
```

- **Informer** = Reflector (does the List then Watch against the apiserver) + local cache (Indexer) + event handlers that enqueue work items.
- On startup, an informer does a full **List** to seed the cache, then switches to **Watch** to receive incremental deltas (using the `resourceVersion` returned by the List as the watch's starting point) — this is the "list-watch" pattern.
- A periodic **resync** (default often 30s in client-go, configurable) re-lists from the local cache (not the apiserver) and re-enqueues every object, purely to catch bugs in controller logic that might have dropped an item — it's a safety net for level-triggering, not a re-fetch from etcd.
- Every controller reads from its own local Indexer cache, never live from the apiserver, for the "what's the current state" side of a reconcile — this is what allows dozens of controllers to run against one apiserver without linearly multiplying etcd read load. Writes (creating/updating objects) still go live to the apiserver, of course.
- If a Watch connection drops (apiserver restart, network blip, watch timeout — apiserver deliberately terminates long-lived watches periodically to rebalance load), the Reflector detects it, does a fresh List to resynchronize (or resumes from last known `resourceVersion` if within etcd's compaction window — see `410 Gone` below), and continues.

### 5.3 resourceVersion and optimistic concurrency

Every object carries a `resourceVersion` (etcd's mod-revision under the hood). Updates must include the `resourceVersion` the client last read:

```bash
kubectl get deployment web -o jsonpath='{.metadata.resourceVersion}'
```

If another writer updated the object in between your read and your write, your update is rejected:

```
Error from server (Conflict): Operation cannot be fulfilled on deployments.apps "web":
the object has been modified; please apply your changes to the latest version and try again
```

This is **`409 Conflict`** — pure optimistic concurrency control, no locking. Controllers handle this routinely: `client-go`'s `RetryOnConflict` helper re-fetches the latest object, re-applies the intended change, and retries, typically with exponential backoff. A controller that doesn't handle 409s gracefully (e.g., logs and gives up instead of retrying) will silently stop reconciling under any contention — a subtle bug to watch for in custom operators.

A related but distinct failure: **`410 Gone`** on a Watch — happens when a controller's cache falls behind etcd's compaction (etcd periodically compacts old MVCC revisions to bound disk usage) and the requested `resourceVersion` no longer exists. The client must do a fresh List to recover; a well-written informer does this automatically, but a hand-rolled watch loop that doesn't handle 410 will spin forever retrying a `resourceVersion` that will never become valid again.

---

## 6. Leader Election for HA Control Plane Components

`kube-controller-manager` and `kube-scheduler` are typically run with multiple replicas (one per control-plane node) for HA, but **only one replica may be active at a time** — running multiple active schedulers or controller-managers simultaneously would cause double-processing and race conditions (e.g., two controller-managers both trying to garbage-collect the same object, or two schedulers double-binding the same Pod).

### 6.1 Mechanics

Modern versions use a `Lease` object (`coordination.k8s.io/v1`) in `kube-system`; older versions used annotations on a `ConfigMap` or `Endpoints` object (`control-plane.alpha.kubernetes.io/leader` annotation) — functionally identical purpose, Lease is just a lighter-weight, purpose-built object.

```bash
kubectl get lease -n kube-system
```

```
NAME                     HOLDER                                              AGE
kube-controller-manager  control-plane-node-2_a1b2c3d4-5678-90ab-cdef-1234   14d
kube-scheduler           control-plane-node-1_9f8e7d6c-5432-10fe-dcba-9876   14d
```

```bash
kubectl get lease kube-scheduler -n kube-system -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  holderIdentity: control-plane-node-1_9f8e7d6c-5432-10fe-dcba-9876
  leaseDurationSeconds: 15
  acquireTime: "2026-07-04T02:11:03.000000Z"
  renewTime: "2026-07-18T09:44:12.481922Z"
  leaseTransitions: 3
```

`holderIdentity` tells you exactly which replica (by pod/node identity suffix + a UUID) currently owns leadership. `leaseTransitions` is a cheap way to spot instability — a number climbing rapidly means leadership is flapping (crashlooping component, network partition between a replica and etcd/apiserver, or resource starvation causing renew calls to miss their deadline).

### 6.2 Tuning: duration, deadline, retry period

Three parameters control failover behavior (defaults shown, configurable via `--leader-elect-lease-duration`, `--leader-elect-renew-deadline`, `--leader-elect-retry-period` on both components):

- **`leaseDuration`** (default 15s): how long a lease is valid without renewal before another candidate may claim it.
- **`renewDeadline`** (default 10s): how long the current leader will keep retrying to renew before giving up and stepping down voluntarily.
- **`retryPeriod`** (default 2s): how often non-leader candidates poll to try to acquire the lease.

Practical implication: **failover time is roughly bounded by `leaseDuration`** — if the active leader dies ungracefully (SIGKILL, node power loss, network partition), other replicas must wait for the existing lease to expire (up to `leaseDuration` seconds since the last successful renewal) before one of them can acquire it. Lowering `leaseDuration` gives faster failover but increases the risk of unnecessary leader flapping under transient latency/GC pauses against etcd — a leader that hits a slow GC pause or a momentary etcd latency spike longer than `renewDeadline` will lose leadership even though it's still alive, causing a needless transition. Most production tuning leaves these at defaults unless there's a specific, measured reason (e.g., very large clusters with observed etcd write latency close to the renew deadline, where defaults cause flapping and should be loosened, not tightened).

### 6.3 What actually happens during a crash

1. Leader (say `kube-scheduler` on node-1) crashes.
2. Its lease stops being renewed. Standby replicas on other nodes are polling every `retryPeriod` (2s) but can't acquire until the lease's `renewTime + leaseDuration` has passed.
3. Once expired, the first standby to successfully `Update` the Lease object (an atomic conditional update against etcd, which is itself just another optimistic-concurrency write — first one wins) becomes the new leader.
4. During this gap — from crash to new leader acquiring the lease — **no scheduling happens at all** (for kube-scheduler) or **no reconciliation happens at all** (for kube-controller-manager). Existing Pods keep running (kubelets are independent), but new Pods sit `Pending`, and controllers stop reacting to any changes cluster-wide until a new leader takes over. This window is typically low-teens seconds with default settings — small in absolute terms, but worth knowing about explicitly when someone asks "why did this Pod sit Pending for 12 seconds after that node died" during an incident review.
