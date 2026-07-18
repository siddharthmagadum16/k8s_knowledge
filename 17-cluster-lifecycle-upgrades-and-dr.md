# Cluster Lifecycle: Upgrades and Disaster Recovery

## 1. Version Skew Policy

The version skew policy is not a suggestion — the apiserver will refuse to work correctly (or silently misbehave) if you violate it. Here is the actual policy as of current Kubernetes releases:

```
kube-apiserver:      the reference version (e.g. v1.30)
kube-controller-manager,
kube-scheduler,
cloud-controller-manager:  must be <= apiserver version, same minor OK, no more than 1 minor behind
kubelet:             can be up to 3 minor versions behind apiserver (relaxed from 2 in older policy)
kube-proxy:          same rule as kubelet, must match the kubelet on that node
kubectl:             within 1 minor version above or below apiserver
```

In an HA control plane (multiple apiservers behind a load balancer), during an upgrade you will transiently have apiservers at N and N+1 talking to the same etcd. The hard rule: **all apiservers in the cluster must be within 1 minor version of each other**. You cannot run apiserver v1.28 and v1.30 simultaneously behind the same LB — skip-upgrading a minor version across the control plane is not supported, even transiently.

```
Allowed transient state during rolling control-plane upgrade:
  apiserver-1: v1.29        apiserver-2: v1.29        apiserver-3: v1.29
       |                          |                          |
       v (upgrade in progress)
  apiserver-1: v1.30        apiserver-2: v1.29        apiserver-3: v1.29   <- OK, within 1 minor
       |                          |
       v
  apiserver-1: v1.30        apiserver-2: v1.30        apiserver-3: v1.29   <- OK
       |
       v
  apiserver-1: v1.30        apiserver-2: v1.30        apiserver-3: v1.30   <- done

NEVER: apiserver-1: v1.31   apiserver-2: v1.29   <- 2 minors apart, unsupported, undefined behavior
```

### Why control plane must go first — concretely

The apiserver is the schema authority. It defines which API versions and fields exist, validates objects against OpenAPI schemas, and runs admission control. kubelets, controllers, and kube-proxy are all *clients* of the apiserver — they read and write objects through it, and they embed assumptions about what fields and API groups are available.

The direction of compatibility is: **newer apiserver understands older clients, but older apiserver does not understand newer clients**, and clients must never assume more features than the apiserver they're talking to actually offers. That is why control plane goes first — the apiserver has to already understand everything a slightly-behind kubelet might send it, and it does, by design (N vs N-3 kubelet support). If you flip the order and upgrade kubelets first, you'll have kubelets sending API calls / expecting behavior from an apiserver version that doesn't exist yet on the control plane, and depending on the delta, some of that breaks outright.

**Concrete failure example: API removal.** Kubernetes deprecates and eventually removes API versions on a fixed schedule (e.g., `policy/v1beta1` PodDisruptionBudget removed in v1.25, `extensions/v1beta1` Ingress removed in v1.22, `batch/v1beta1` CronJob removed in v1.25). If you upgrade the apiserver to the version that drops an old API group, and you still have kubelets, controllers, or **your own client workloads / operators / CI pipelines** built against the old API version, those writes/reads start failing with:

```
error: resource mapping not found for name: "x" namespace: "y" from "manifest.yaml":
no matches for kind "CronJob" in version "batch/v1beta1"
ensure CRDs are installed first
```

This isn't specific to kubelet-vs-apiserver skew directly (kubelet itself doesn't usually call deprecated APIs), but it's the same underlying mechanism that makes "upgrade control plane last" catastrophic: anything downstream of the apiserver — Helm charts, operators reconciling CRs, admission webhooks registered against an old `admissionregistration.k8s.io/v1beta1` — breaks the moment the apiserver stops serving the old version, regardless of when you do it. That's why the standard guidance is: bump the apiserver only after auditing what's still using soon-to-be-removed APIs (`kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis`, or `kubent`/`pluto` tools), and only then move nodes.

**Concrete kubelet-specific failure if kubelets get ahead of apiserver:** a kubelet newer than the apiserver may report node status fields, use kubelet API fields, or expect PATCH/field-manager semantics the apiserver doesn't recognize/support yet, and new kubelet features gated behind a specific apiserver-side API version simply won't function — the kubelet will error registering the node or silently degrade a feature (e.g., a new resource type in node status, a new admission-relevant field). This is precisely why skew policy caps kubelet from going *ahead* of apiserver at all — it's disallowed, full stop, whereas trailing behind (up to 3 minors) is the explicitly supported direction.

---

## 2. Upgrade Strategy

### 2.1 kubeadm upgrade flow (self-managed clusters)

Sequence, one control-plane node at a time, then workers one at a time:

```bash
# Step 0 — on the FIRST control-plane node, check what's available
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.30.4-1.1
apt-mark hold kubeadm

kubeadm upgrade plan
```

`kubeadm upgrade plan` diffs your current cluster state against the target version and tells you which components will move and whether there are known issues (e.g., deprecated flags, kubelet config drift). Read the output — it will list things like:

```
COMPONENT                 CURRENT       TARGET
kube-apiserver            v1.29.6       v1.30.4
kube-controller-manager   v1.29.6       v1.30.4
kube-scheduler            v1.29.6       v1.30.4
kube-proxy                v1.29.6       v1.30.4
CoreDNS                   v1.11.1       v1.11.3
etcd                      3.5.12-0      3.5.15-0
```

```bash
# Step 1 — apply on the FIRST control-plane node only
kubeadm upgrade apply v1.30.4
```

This upgrades the control plane static pod manifests (apiserver, controller-manager, scheduler), etcd if bundled, and CoreDNS/kube-proxy DaemonSets cluster-wide via the addon phase. It does NOT touch kubelet on this node yet — that's a separate step.

```bash
# Step 2 — on EVERY OTHER control-plane node (not the first)
kubeadm upgrade node
```

`kubeadm upgrade node` fetches the already-updated cluster config (uploaded to the `kubeadm-config` ConfigMap by step 1) and applies the equivalent static pod manifest changes locally. It does not re-run the full plan/apply logic — it just syncs this node to what the cluster already decided.

```bash
# Step 3 — on EACH node (control-plane node itself, then workers), one at a time:

# 3a. Drain first
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# 3b. Upgrade kubelet + kubectl packages
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.30.4-1.1 kubectl=1.30.4-1.1
apt-mark hold kubelet kubectl

# 3c. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 3d. Uncordon
kubectl uncordon <node>
```

**apt-mark hold gotchas:**

- Package managers hold kubelet/kubeadm/kubectl specifically because unattended `apt-get upgrade` / `unattended-upgrades` would otherwise silently bump your Kubernetes version outside of your control and outside the skew policy — this is one of the most common causes of "my cluster broke overnight" incidents in self-managed environments.
- You must `apt-mark unhold` before installing the new version, or `apt-get install` will refuse (or worse, some automation scripts force it with `--allow-downgrades`/`dpkg -i` and skip the hold entirely, meaning the *next* `apt-get upgrade` moves it further than intended).
- Forgetting to re-`hold` after upgrading is the classic mistake: next time cron-driven unattended-upgrades runs, it drags kubelet to whatever is latest in the repo, potentially jumping 2+ minor versions past your control plane, violating skew policy, and you find out only when nodes start reporting `NotReady` or the apiserver logs are full of `unknown field` warnings from a kubelet that's too new.
- Always pin exact versions (`kubelet=1.30.4-1.1`), never bare `apt-get install kubelet` during an upgrade — bare install grabs latest, not target.

### 2.2 Managed cluster upgrades (EKS / GKE / AKS)

Two fundamentally different node upgrade strategies, both control-plane-managed-for-you (the cloud handles apiserver/etcd upgrades; you only deal with node upgrades):

**Blue-green node pool replacement:**

```
1. Create new node pool at target version (nodes join, tainted/labeled distinctly, e.g. pool=blue vs pool=green)
2. Verify new pool is healthy: kubectl get nodes -l pool=green
3. Cordon all nodes in old pool:  kubectl cordon -l pool=blue
4. Drain old pool nodes one at a time (or in controlled batches):
     kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
5. Confirm workloads rescheduled onto new pool and are healthy
6. Delete old node pool (or scale to 0) once confident
```

Tradeoffs:
- Pros: instant rollback — if the new pool misbehaves, just cordon it and uncordon the old pool (which still exists, unscaled). No half-upgraded node state to reason about.
- Cons: you pay for both pools simultaneously during the transition (potentially double compute cost for the overlap window). You must carefully handle LB/ingress: pods draining from old nodes need their endpoints deregistered from the Service/LoadBalancer *before* the node disappears, or you get connection resets — this is why `drain` (graceful eviction respecting `terminationGracePeriodSeconds` and readiness-gate deregistration delay) matters, not just deleting nodes outright.
- Best for: production clusters where downtime tolerance is near zero and cost of a doubled node-hour window is acceptable.

**In-place / rolling node upgrade (managed node group rolling replacement — EKS managed node groups, GKE node pool auto-upgrade, AKS node image upgrade):**

```
1. Cloud provider spins up one new-version node
2. Cordons + drains one old node
3. Terminates old node
4. Repeats, one (or maxSurge/maxUnavailable-configured batch of) node(s) at a time
```

Tradeoffs:
- Pros: cheaper — no sustained double-capacity, only a small surge overlap.
- Cons: riskier mid-upgrade — if a node fails to come up healthy at the new version halfway through, you now have a cluster in a mixed-version state with no simple single-switch rollback; you have to manually intervene per-node. Also harder to bail out cleanly since the old pool is being destroyed as you go, not kept around.
- Configure `maxUnavailable`/`maxSurge` conservatively (e.g., surge 1, unavailable 0) for stateful or latency-sensitive workloads.

Recommendation: use blue-green for anything with strict SLAs or where you're unsure about workload compatibility with the new node OS/kubelet version (e.g., containerd major version bump, cgroup v1->v2 migration). Use in-place rolling for cost-sensitive, stateless, horizontally-redundant fleets where a bad batch is cheap to detect and pause.

### 2.3 Draining nodes safely

```bash
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

What `drain` actually does, step by step:
1. Cordons the node (`kubectl cordon` equivalent) — sets `spec.unschedulable: true` so the scheduler stops placing new pods there. Existing pods are untouched by this step.
2. For every pod on the node (excluding DaemonSet-managed pods, which `--ignore-daemonsets` skips since they're recreated by the DaemonSet controller anyway and can't be "moved"), it issues a `POST` to the pod's `eviction` subresource — **not** a direct delete.
3. The apiserver's eviction handler checks any matching PodDisruptionBudget before allowing this. If eviction would violate the PDB, the request is rejected with HTTP 429 (Too Many Requests), and `kubectl drain` retries with backoff.
4. `--delete-emptydir-data` is required if any pod on the node uses an `emptyDir` volume — without it, drain refuses to evict such pods (since emptyDir data is node-local and will be lost), forcing you to explicitly acknowledge data loss for that ephemeral volume.

**When drain hangs:** if a PDB is structured so eviction can never succeed (see PDB section below), `kubectl drain` will sit there retrying indefinitely, printing:

```
error when evicting pods/"web-7d9f8b6-x2kqp" -n default (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```

- `--timeout=<duration>`: how long `kubectl drain` itself will keep retrying before giving up and returning an error to your shell. It does NOT force anything — after timeout, the node is still cordoned, pods that couldn't be evicted are still running, and you're left to intervene manually (fix the PDB, scale up replicas elsewhere, or force).
- `--force`: allows drain to proceed even for pods **not managed by any controller** (bare/naked pods) — those have no controller to recreate them, so `--force` deletes them outright, permanently, with no replacement scheduled elsewhere. This is unrelated to PDB bypass — `--force` does not bypass PDBs. There is no supported `kubectl drain` flag to bypass a PDB; if you truly must override a PDB blocking a critical drain, you either scale the deployment, temporarily relax/delete the PDB, or fall back to directly deleting the pod (`kubectl delete pod --grace-period=<n>`), which bypasses the eviction API's PDB check entirely because it isn't going through eviction — but this is a manual override that intentionally violates the safety guarantee the PDB was providing, and should be a deliberate, documented incident-response action, not routine practice.
- The real risk with force-deleting or bypassing eviction: for pods with local state (a single-replica StatefulSet writing to local-path storage, a leader-elected process mid-write, a database pod without a synced replica), forcing removal can cause data loss or corruption because the pod is killed without the orchestration that a graceful, budget-respecting eviction was meant to guarantee.

---

## 3. PodDisruptionBudgets in Depth

### 3.1 minAvailable vs maxUnavailable

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2        # at least 2 pods matching selector must stay Ready at all times
  # OR
  # maxUnavailable: 1    # at most 1 pod matching selector may be down at a time
  selector:
    matchLabels:
      app: web
```

- `minAvailable`: absolute number or percentage of pods that must remain healthy. Eviction is blocked if it would drop healthy count below this.
- `maxUnavailable`: absolute number or percentage of pods allowed to be unhealthy/gone simultaneously. Eviction is blocked if the number already-unavailable plus this one would exceed the cap.
- You can only set one, not both, per PDB object.

### 3.2 How enforcement actually works

Eviction is not a regular delete — it's a `POST /api/v1/namespaces/<ns>/pods/<pod>/eviction`. The apiserver's eviction admission logic:

```
1. Look up the PDB(s) whose selector matches this pod.
2. Read PDB.status.currentHealthy, .desiredHealthy, .disruptionsAllowed
   (these are computed and kept current by the disruption controller,
   a controller-manager loop watching pod readiness for the selector).
3. If disruptionsAllowed > 0: allow eviction, decrement the allowed count,
   respond 200/201 to the caller (kubectl drain, descheduler, cluster-autoscaler).
4. If disruptionsAllowed == 0: reject with HTTP 429, response body includes
   "Cannot evict pod as it would violate the pod's disruption budget."
```

This means the PDB is enforced synchronously, in the request path of whatever tool is trying to evict — it isn't a background reconciler that "eventually" stops you; the eviction call itself fails atomically.

### 3.3 Critical distinction: voluntary vs involuntary disruption

PDBs are a contract for **voluntary disruptions only** — actions initiated by cluster operators/tooling going through the eviction API:
- `kubectl drain` during node maintenance/upgrade
- descheduler evicting pods for rebalancing
- cluster-autoscaler evicting pods during scale-down
- manual `kubectl drain`/eviction by an operator

PDBs provide **no protection** against involuntary disruption:
- a node's hardware fails or kernel panics and the node dies outright
- a spot/preemptible instance gets reclaimed by the cloud provider with 30s-2min notice
- the kubelet itself crashes or the node loses network connectivity and is marked `NotReady`, pods get terminated as unreachable
- an OOM-killer event on the node kills the container

None of these go through the eviction API — the pod (and possibly the whole node) simply disappears, and the PDB has no mechanism to prevent or even delay it. The disruption controller only ever intercepts calls to the eviction *subresource*; a node dying doesn't call that subresource, it just stops reporting heartbeats.

```
PDB scope:
┌─────────────────────────────────────────────────────┐
│  Voluntary (blocked by PDB if it violates budget)    │
│  - kubectl drain / cordon+evict                      │
│  - cluster-autoscaler scale-down eviction             │
│  - descheduler rebalancing                            │
├─────────────────────────────────────────────────────┤
│  Involuntary (PDB has ZERO effect)                    │
│  - node hardware failure / kernel panic               │
│  - spot instance reclamation                           │
│  - kubelet crash / node NotReady eviction              │
│  - OOM kill                                            │
└─────────────────────────────────────────────────────┘
```

**The false-confidence trap:** a team sets `minAvailable: 100%` (or `minAvailable` equal to replica count) thinking "this guarantees zero downtime, ever." It only guarantees that *your own tooling* (drain, autoscaler) won't be allowed to take a pod down voluntarily. A real node failure taking out every replica simultaneously (e.g., all 3 replicas happened to land on the same node due to a scheduling bug or lack of anti-affinity, or an AZ-wide outage takes multiple nodes at once) is completely unaffected by the PDB and you still go down. PDBs are a scheduling-safety mechanism for planned maintenance, not a high-availability guarantee — actual HA comes from replica count + anti-affinity/topology spread constraints + multi-AZ placement, independent of PDBs.

### 3.4 The undrainable-deployment gotcha

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-worker
spec:
  replicas: 3
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: legacy-worker-pdb
spec:
  minAvailable: 3     # <-- BUG: equals total replica count
  selector:
    matchLabels:
      app: legacy-worker
```

With `minAvailable: 3` and exactly 3 replicas, `disruptionsAllowed` is permanently `0` — evicting even one pod would drop healthy count to 2, below the required 3. Every eviction attempt against any pod in this deployment is rejected forever, regardless of cluster health. The practical symptom: `kubectl drain` on any node running one of these pods hangs indefinitely (or until `--timeout`), and cluster-autoscaler will never be able to scale down a node hosting one of these pods — it just logs the block and gives up on that node until conditions change.

The single-replica version of this bug is the most common in practice: `replicas: 1` with `minAvailable: 1` on a PDB (sometimes added "for safety" without understanding the implication) — the one pod can never be voluntarily evicted, ever, so any node holding it is permanently pinned for cluster-autoscaler and permanently blocks graceful drains.

Fix: `minAvailable` (or the maxUnavailable equivalent) must leave at least 1 unit of slack relative to replica count for any voluntary disruption to ever be possible — e.g., `maxUnavailable: 1` with `replicas: 3`, or bump replicas so `minAvailable` isn't the ceiling.

---

## 4. etcd Disaster Recovery Runbook

### 4.1 Quorum math and quorum-loss recovery

etcd requires a **majority** of members to be alive and reachable to accept writes: `(N/2) + 1`.

```
Cluster size    Tolerated failures    Quorum needed
     1                 0                   1
     3                 1                   2
     5                 2                   3
     7                 3                   4
```

In a standard 3-node etcd cluster, losing 2 of 3 members means only 1 remains — below the quorum of 2. The cluster stops accepting writes entirely (reads may still be served by the surviving member in some cases, but the Kubernetes control plane effectively goes read-only/degraded: existing pods keep running, but nothing new can be scheduled, no object can be updated, `kubectl apply` hangs or errors).

**Recovery when quorum is lost (2 of 3 etcd nodes are dead/unrecoverable):**

```bash
# 1. On the surviving member, confirm its own state and check member list
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

etcdctl --endpoints=https://127.0.0.1:2379 ... endpoint status -w table
```

```bash
# 2. Stop etcd on the surviving node (it's a static pod — move the manifest out
#    so kubelet stops managing it, then stop the container manually if needed)
mv /etc/kubernetes/manifests/etcd.yaml /tmp/etcd.yaml.bak
# wait for the static pod to actually disappear:
crictl ps | grep etcd
```

```bash
# 3. Force a new single-member cluster from this node's existing data directory.
#    This tells etcd: "stop trying to reach the old peers, you are now the
#    sole authoritative member of a brand-new cluster, keep your existing data."
ETCDCTL_API=3 etcd \
  --force-new-cluster \
  --data-dir=/var/lib/etcd \
  --name=<this-node-name> \
  --initial-cluster=<this-node-name>=https://<this-node-ip>:2380 \
  --initial-advertise-peer-urls=https://<this-node-ip>:2380 \
  --listen-peer-urls=https://<this-node-ip>:2380 \
  --listen-client-urls=https://127.0.0.1:2379,https://<this-node-ip>:2379 \
  --advertise-client-urls=https://<this-node-ip>:2379
# run this once interactively/foreground to confirm it starts cleanly, then
# fold the --force-new-cluster flag into the static pod manifest temporarily,
# start it via kubelet, verify health, then REMOVE --force-new-cluster
# before the next restart (it must only be used once).
```

```bash
# 4. Verify single-member cluster is healthy
etcdctl endpoint health
etcdctl member list -w table
# should show exactly 1 member now

# 5. Re-add the other two members ONE AT A TIME, standing up fresh nodes
#    (new data-dir, no stale data from the dead nodes)
etcdctl member add etcd-2 --peer-urls=https://<etcd-2-ip>:2380
# then start etcd on that node pointed at the now-updated initial-cluster
# string that etcdctl member add prints out, with --initial-cluster-state=existing

etcdctl member add etcd-3 --peer-urls=https://<etcd-3-ip>:2380
# same on the third node
```

```bash
# 6. After both are back and quorum is restored to 3, verify
etcdctl member list -w table
etcdctl endpoint health --cluster
kubectl get nodes
kubectl get componentstatuses   # deprecated but still informative on some versions
kubectl get pods -n kube-system
```

Update `--advertise-client-urls` / static pod manifests on any node whose IP changed (e.g., you replaced a dead node with a new instance at a different IP) — a stale advertise URL baked into etcd's own member list will cause other members and the apiserver to keep trying the old address.

### 4.2 Restoring from a snapshot into a fresh cluster

Routine backup:

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# always verify the snapshot is actually valid, don't assume:
etcdctl snapshot status /backup/etcd-snapshot-20260718-020000.db -w table
```

Full disaster restore (e.g., all 3 etcd nodes lost, or you're intentionally rolling back to a known-good point):

```bash
# 1. On EACH control-plane node, restore the snapshot into a fresh data-dir.
#    Must be run identically (same snapshot file) on all nodes that will
#    form the new cluster, each restoring to its own local path.
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20260718-020000.db \
  --name=etcd-1 \
  --initial-cluster=etcd-1=https://10.0.1.11:2380,etcd-2=https://10.0.1.12:2380,etcd-3=https://10.0.1.13:2380 \
  --initial-advertise-peer-urls=https://10.0.1.11:2380 \
  --data-dir=/var/lib/etcd-restored

# repeat on etcd-2 and etcd-3 with --name and --initial-advertise-peer-urls
# adjusted to that node's own identity, but the SAME --initial-cluster string
# and SAME snapshot file, on all nodes.
```

```bash
# 2. Point the static pod manifest at the new data directory
#    Edit /etc/kubernetes/manifests/etcd.yaml on each node:
#      --data-dir=/var/lib/etcd-restored   (was /var/lib/etcd)
#    and update the hostPath volume mount to match.

# 3. Restart kubelet to pick up the manifest change (static pods are
#    reconciled by kubelet watching the manifest directory)
systemctl restart kubelet

# 4. Watch etcd come up
crictl ps | grep etcd
crictl logs <etcd-container-id>
```

```bash
# 5. Verify
etcdctl endpoint health --cluster \
  --endpoints=https://10.0.1.11:2379,https://10.0.1.12:2379,https://10.0.1.13:2379

kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Expect to see controllers reconcile briefly (they'll notice objects that existed after the snapshot are now gone, and objects deleted after the snapshot have "come back") — this is the expected, unavoidable data-loss window inherent to point-in-time restore: **anything created/changed between the snapshot and the failure is gone**, and this is exactly why RPO (recovery point objective) matters — how often you snapshot determines how much you can lose.

### 4.3 Split-brain: why Raft prevents it structurally, and the human-error path around that

etcd's Raft consensus fundamentally cannot split-brain the way a naive active-active system can: a Raft cluster only commits a write (and only elects a leader) with agreement from a strict majority of members. Two partitions of the same cluster can never both have a majority simultaneously (majority is defined against the *total* configured member count, not just who's reachable), so at most one side of any partition can make progress. The other side stalls, unable to write, by design.

**The scenario that manually recreates split-brain despite this:** you don't get split-brain from Raft failing — you get it from an operator running `--force-new-cluster` (or a restore) incorrectly while some of the "dead" members are not actually dead.

```
WRONG sequence that causes split-brain:

  etcd-1 (alive, thinks it's fine)  etcd-2 (network partitioned, thinks it's fine)  etcd-3 (down)

  Operator assumes etcd-2 and etcd-3 are both dead (etcd-2 is actually just
  partitioned, not dead) and runs --force-new-cluster on etcd-1.

  Result: etcd-1 now believes it is a brand-new authoritative single-member
  cluster and starts accepting writes independently.

  Meanwhile etcd-2, still up and still able to talk to... nobody, may itself
  either stall (correct Raft behavior, no quorum) OR, if the operator ALSO
  force-recovers etcd-2 separately (e.g. runs recovery procedures on multiple
  nodes in parallel without confirming coordination), you now have TWO
  independently "authoritative" single/multi-member clusters, each accepting
  writes, each believing it is the one true etcd for this Kubernetes cluster.

  The apiserver(s) pointed at different endpoints now diverge silently —
  this is a human-induced split-brain, not a Raft failure.
```

How to avoid it:
1. **Before running any force-recovery or restore procedure, fully stop etcd on every member you believe is affected** — do not leave any "maybe still alive" member running while you force-create a new cluster elsewhere. Confirm via `crictl ps`/`systemctl status` on every node, not just via etcd client calls (which can themselves be affected by the same network partition you're trying to route around).
2. Never run `--force-new-cluster` or `snapshot restore` on more than one member concurrently as independent, uncoordinated operations — the "join the new cluster" steps for remaining members must always be *additions to* the one node that was force-recovered, never a second independent force-recovery.
3. After any recovery, immediately verify `etcdctl member list` returns the exact expected membership on every apiserver's configured etcd endpoint — if two different apiservers or two different etcdctl invocations against different endpoints show different member lists or different cluster IDs, stop immediately, you have a split-brain in progress, and you must pick one side as canonical and hard-stop the other.
4. Treat `--force-new-cluster` as a last-resort, single-use, single-node operation — remove it from the static pod manifest immediately after confirming health, so an accidental kubelet-triggered restart doesn't re-run it against an already-healthy multi-member cluster (which would itself fork a new cluster identity).

---

## 5. Cluster Autoscaler Internals

### 5.1 Scale-up path

```
1. A pod goes Pending. Scheduler tries to place it, fails all nodes,
   emits a FailedScheduling event:
     "0/8 nodes are available: 8 Insufficient cpu."
2. cluster-autoscaler's watch loop picks up the Pending pod (polls on an
   interval, default ~10s scan-interval).
3. For each configured node group / ASG / managed node pool, CA simulates:
   "if I added one more node of this node group's instance type/template,
   would this pod (and other pending pods) become schedulable?"
   This simulation uses the same predicates the real scheduler uses
   (taints/tolerations, node affinity, resource requests, topology spread).
4. CA picks the node group that satisfies the most pending pods most
   cheaply (heuristic, configurable via --expander: random, most-pods,
   least-waste, price, priority).
5. CA calls the cloud provider API to increase desired capacity
   (e.g., AWS ASG SetDesiredCapacity, GCP MIG resize, Azure VMSS scale).
6. New node joins, kubelet registers, scheduler places the pending pod(s).
```

### 5.2 Scale-down path

```
1. CA continuously computes per-node utilization: max(sum(cpu requests),
   sum(memory requests)) / node allocatable, for both cpu and memory.
2. If utilization stays BELOW threshold (default 50%) for BOTH cpu and
   memory, continuously, for the unneeded-duration (default 10 minutes),
   the node becomes a scale-down candidate.
3. CRITICAL GATE: before removing the node, CA verifies every pod
   currently on it CAN be rescheduled elsewhere in the cluster right now.
   If even one pod on an otherwise-idle node fails this check, the ENTIRE
   node is excluded from scale-down, no matter how idle the rest of it is.
```

### 5.3 What blocks scale-down (the pod-level veto list)

A node will NOT be scaled down if it has any pod matching:

- **Bare pods** — not owned by a ReplicaSet/Deployment/StatefulSet/Job — because CA has nothing to guarantee will recreate them elsewhere; deleting the node deletes the pod permanently.
- **Pods using local storage** — specifically `emptyDir` — unless the pod explicitly opts in via the annotation `cluster-autoscaler.kubernetes.io/safe-to-evict-local-volumes`.
- **Pods whose eviction would violate a PDB** — same eviction-API check as drain.
- **Pods with restrictive affinity/anti-affinity** that can't be satisfied by any other current node (CA's simulation checks this — if the only node matching a required node-affinity term is this one, or a pod anti-affinity rule means it can't co-locate with pods elsewhere, the node is pinned).
- **kube-system pods without a PDB** — by default, CA treats kube-system namespace pods as non-evictable unless they have a PDB explicitly permitting it. This is a common surprise: a CoreDNS or metrics-server pod with no PDB sitting on an otherwise-empty node can pin that node indefinitely.
- **Pods annotated** `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` — an explicit opt-out, often set on things like local monitoring agents, log shippers holding local buffers, or anything the team doesn't want CA touching.

### 5.4 Concrete diagnosable scenario

Symptom: a node sits at 18-20% CPU/memory utilization for hours, cluster-autoscaler never removes it, and you're paying for idle capacity.

Root cause (common in practice): a single lone pod on that node belongs to a Deployment with `replicas: 1` and a PDB with `minAvailable: 1`. Per the section above, that PDB makes the pod's eviction always rejected (0 disruptions allowed, ever) — CA's scale-down simulation tries to evict it, gets the same 429 that `kubectl drain` would, and gives up on that node for this scan cycle, repeating forever.

**Diagnosis steps:**

```bash
# 1. Find the idle node
kubectl top nodes
# node-7   180m (9%)   1200Mi (16%)

# 2. See what's actually running there
kubectl get pods -A --field-selector spec.nodeName=node-7 -o wide

# 3. Check cluster-autoscaler's own reasoning — it logs per-node scale-down
#    eligibility explicitly
kubectl logs -n kube-system deployment/cluster-autoscaler | grep -i node-7
# "Node node-7 - can't be removed: pod default/reporting-service-xxxx is not
#  safe to evict: pod's disruption budget would be violated"

# 4. Confirm via events
kubectl describe node node-7 | tail -30
kubectl get pdb -A
kubectl describe pdb reporting-service-pdb -n default
#   Allowed disruptions: 0
```

**Fix:** either scale the Deployment to >=2 replicas and adjust the PDB to leave slack (e.g., `replicas: 2` with `maxUnavailable: 1`), or, if a true singleton is required, accept that the node hosting it can never be autoscaled away and pin it deliberately (dedicated small node group, or a `safe-to-evict: false` annotation to make the intent explicit rather than accidental) rather than leaving it as an unexplained cost leak.

---

## 6. Backup Strategy Beyond etcd

### 6.1 Why etcd snapshots alone are not sufficient

etcd holds the Kubernetes **control plane's view of the world** — object specs, statuses, ConfigMaps, Secrets, the fact that a PVC exists and is Bound to a PV. It does **not** hold the actual bytes sitting on the PV — the actual database files, uploaded assets, application data living on an EBS volume, a Ceph RBD image, an NFS share, or a cloud disk.

If you restore only an etcd snapshot after a disaster:
- Kubernetes objects reappear (Deployments, PVCs, PV bindings) exactly as they were at snapshot time.
- But the underlying storage volumes referenced by those PVs may have been deleted, replaced, or diverged in the interim (e.g., the disaster that took out etcd also took out the storage backend, or new data was written to disks after the etcd snapshot was taken but etcd doesn't know about it because it never tracked the *content*, only the *existence/binding* of the volume).
- Result: PVC objects bind to PVs whose backing disks no longer exist, are stale, or point at storage that has since diverged from what the restored etcd metadata assumes — you get a cluster that "looks" restored (`kubectl get pods` shows Running) but pods crash-loop or, worse, silently start operating against wrong/stale data.

This is why etcd RPO and PV-data RPO must be planned **together** — a consistent recovery point needs both the object metadata and the actual data to correspond to roughly the same point in time, or you get an inconsistent world-view: the cluster's metadata believes one thing exists while the storage layer reflects another.

### 6.2 Velero

Velero is the standard tool for the layer etcd snapshots don't cover:

```bash
# Install (once), configured with a cloud object storage backend (S3/GCS/Azure Blob)
# and a VolumeSnapshotter/CSI plugin matching your storage backend.

# Ad-hoc backup of a namespace
velero backup create checkout-ns-backup --include-namespaces=checkout

# Backup scoped by label selector
velero backup create critical-backup --selector app.kubernetes.io/tier=critical

# Scheduled backup (cron-style)
velero schedule create daily-full-backup --schedule="0 2 * * *" --ttl=720h0m0s
```

What Velero actually captures:
- **Kubernetes object manifests** — every resource matching the backup scope (Deployments, Services, ConfigMaps, Secrets, RBAC, CRDs, PVC/PV objects) serialized and stored in object storage (S3 etc.).
- **PV data**, optionally, via one of two mechanisms:
  - **CSI snapshot API** (`VolumeSnapshot`/`VolumeSnapshotClass`) — if your storage driver supports CSI snapshots (EBS CSI, GCE PD CSI, most modern cloud/SAN drivers do), Velero triggers a native volume snapshot at backup time.
  - **File-system-level backup (Restic/Kopia integration)** — for storage backends without CSI snapshot support, Velero can run a pod-level filesystem backup, copying the actual file contents to object storage.
  - Without either enabled, Velero backs up only the PVC/PV *objects*, not their data — a common misconfiguration where teams believe they have full backups but only have metadata.

```bash
# Restore into the SAME cluster (disaster recovery within a cluster)
velero restore create --from-backup checkout-ns-backup

# Restore into a DIFFERENT cluster (migration / DR to a standby cluster —
# requires the target cluster to have Velero installed and pointed at the
# same object storage bucket)
velero restore create --from-backup checkout-ns-backup --namespace-mappings checkout:checkout-restored
```

Namespace/label scoping matters for both cost and blast radius: you rarely want "back up everything" as your only strategy — scope backups to what actually needs point-in-time recovery (stateful application namespaces), and rely on GitOps/IaC (your manifests in git, Terraform for infra) to reconstruct stateless/config-only namespaces instead of paying to snapshot things that are already reproducible from source.

### 6.3 Restore testing discipline

An untested backup is not a backup — it is an unverified assumption. Practical discipline:

1. **Scheduled restore drills**, not just scheduled backups: periodically (monthly, or after any significant storage/CSI driver change) actually run `velero restore create` into a scratch cluster or an isolated scratch namespace, from a real production backup.
2. **Verify at the application level, not the pod level.** `kubectl get pods` showing `Running` proves the container started — it does not prove the database is queryable, the data is the expected recent version, or the application's health endpoint returns 200 with correct data. Post-restore verification should include: connecting to the restored database and running a row-count/checksum sanity check, hitting the application's actual functional health/readiness path, and confirming timestamps of the most recent record match the expected RPO window.
3. **Measure and document actual observed RTO/RPO**, not assumed ones. Time the drill end-to-end: how long from "disaster declared" to "backup identified" to "restore command issued" to "application verified healthy" — that's your real RTO, and it is very often much longer than what's written in a DR doc that nobody has tested against. Compare the timestamp of the restored data against when the disaster notionally occurred — that delta is your real RPO, driven by your actual backup/snapshot frequency, not the frequency you intended to configure.
4. Keep the drill results (duration, gaps found, fixes applied) as the actual source of truth for what your organization's DR posture is — a documented RTO of "15 minutes" that has never once been achieved in a real drill is a liability, not a capability.
