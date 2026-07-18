# Troubleshooting and Debugging

## Systematic Debugging Methodology

The reason a fixed order matters: every step down this list is more expensive than the one before it — more latency, more blast radius, more cognitive load. `kubectl get pod` is a single API server read that costs milliseconds. `kubectl exec` requires the kubelet to proxy a stream to a running container and assumes the container has a shell. Node-level investigation means SSH or a privileged debug pod. Network-level investigation means packet captures. Skipping straight to `tcpdump` when the pod hasn't even scheduled yet wastes time and often misleads you, because you're looking at a layer where the actual fault doesn't exist.

The order, and what each step rules in/out:

### Step 1 — Pod status and placement

```bash
kubectl get pod -o wide -n <ns>
```

This single line tells you: is the pod scheduled (NODE column populated)? What phase is it in (`Pending`, `Running`, `CrashLoopBackOff`, `Completed`, `Error`, `ImagePullBackOff`)? How many restarts? Which node? Pod IP assigned?

If `NODE` is `<none>` — this is a scheduling problem, not a runtime problem. Stop here, don't look at logs, go straight to `kubectl describe pod` for scheduler events (see Pending section below).

If `RESTARTS` is climbing and phase oscillates between `Running` and `CrashLoopBackOff` — this is a runtime problem. Proceed to step 2.

### Step 2 — Events

```bash
kubectl describe pod <pod> -n <ns>
```

Read the `Events` section at the bottom first, not the spec at the top. Events are ordered chronologically and tell you the *causal chain*: `Scheduled` → `Pulling` → `Pulled` → `Created` → `Started` → then possibly `Killing`, `BackOff`, `Unhealthy`. The `Unhealthy` events specifically tell you if a probe is failing and which one (`Liveness probe failed: ...` vs `Readiness probe failed: ...`) — this distinction alone resolves a large fraction of "pod is crashing" tickets before you ever read application logs.

Also check `Last State` under `Containers` here — this is covered in depth in the CrashLoopBackOff section.

```bash
kubectl get events --sort-by=.lastTimestamp -A | tail -50
```

Cluster-wide events sorted by time are useful when the failure isn't isolated to one pod — e.g., you suspect a node problem, an admission webhook rejecting many pods at once, or a quota being hit. `describe pod` only shows events for that object; the wide view shows correlated events across objects (node conditions, PVC binding failures, replicaset scaling events) in the same window.

### Step 3 — Logs

```bash
kubectl logs <pod> -c <container> -n <ns>
kubectl logs <pod> -c <container> -n <ns> --previous
```

`--previous` is the one people forget. If the container has already restarted, plain `kubectl logs` shows the *current* attempt's log (which may be empty if it just started, or may not yet contain the crash). `--previous` pulls stdout/stderr from the terminated instance — this is where the actual stack trace or panic message lives. For multi-container pods, always specify `-c`; omitting it against a pod with 2+ containers errors out or picks an arbitrary one depending on kubectl version.

```bash
kubectl logs <pod> -n <ns> --all-containers --prefix --since=10m
```

Useful when you don't know which sidecar is the culprit — `--prefix` tags each line with the source container name.

### Step 4 — Exec into the container

```bash
kubectl exec -it <pod> -c <container> -n <ns> -- sh
```

Only makes sense once the container is actually staying up long enough to exec into (a `CrashLoopBackOff` container that dies in 2 seconds gives you no window). Use this to check: is the config file actually mounted where the app expects it, does the env var have the value you think it has, can the process resolve DNS from inside, is there actually free disk space in the container's writable layer. This step confirms or denies hypotheses formed from logs — it is not a substitute for reading logs first.

If the container has no shell (distroless, `scratch` base images), this step fails outright — jump to ephemeral debug containers (see Useful Debugging Commands section).

### Step 5 — Node-level

```bash
kubectl describe node <node>
kubectl get node <node> -o yaml | less
```

You go here when the pod-level story doesn't explain the symptom — e.g., the container looks healthy in logs but keeps getting evicted, or scheduling is refusing to place pods on this node, or multiple unrelated pods on the same node are all misbehaving simultaneously (a strong signal it's the node, not the workload). Check `Conditions`, `Allocatable` vs `Capacity`, and `Events` at the node level.

### Step 6 — Network-level

Last resort, because it requires the most tooling and the most assumptions to already be ruled out (pod is running, logs show the app is up and listening, DNS resolves, but the caller still can't reach it). This is where you check Service endpoints, NetworkPolicy, CNI health, and if truly necessary, packet captures with `tcpdump` from a debug pod on the same node/namespace.

The discipline is: **never jump to step N before ruling out step N-1 with actual evidence**, not assumption. "I think it's a network policy" said before checking `kubectl get endpoints` has burned more on-call hours than almost any other troubleshooting anti-pattern.

---

## CrashLoopBackOff — Exhaustive Root Cause Analysis

`CrashLoopBackOff` is not itself a root cause — it's the kubelet telling you it's applying exponential backoff (10s, 20s, 40s ... capped at 5min) before restarting a container that keeps exiting. The actual cause is one of a small number of categories, and the fastest way to disambiguate them is the **exit code** in `Last State`.

```bash
kubectl describe pod payments-7d8f9c6b45-x2kqz -n payments
```

```
Containers:
  payments-api:
    Container ID:  containerd://8f2a1b...
    Image:         registry.internal/payments-api:v2.4.1
    Image ID:      registry.internal/payments-api@sha256:9c3a...
    Port:          8080/TCP
    State:         Waiting
      Reason:      CrashLoopBackOff
    Last State:    Terminated
      Reason:      Error
      Exit Code:   1
      Started:     Fri, 18 Jul 2026 09:41:02 +0530
      Finished:    Fri, 18 Jul 2026 09:41:04 +0530
    Ready:         False
    Restart Count: 7
    Environment:
      DB_HOST:      <set to the key 'db_host' of config map 'payments-config'>  Optional: false
      DB_PASSWORD:  <set to the key 'db_password' in secret 'payments-secret'>  Optional: false
    Mounts:
      /etc/config from config-volume (rw)
Events:
  Type     Reason     Age                From     Message
  ----     ------     ----               ----     -------
  Normal   Pulled     3m (x7 over 12m)   kubelet  Container image already present on machine
  Normal   Created    3m (x7 over 12m)   kubelet  Created container payments-api
  Normal   Started    3m (x7 over 12m)   kubelet  Started container payments-api
  Warning  BackOff    2m (x15 over 11m)  kubelet  Back-off restarting failed container
```

`Finished` minus `Started` here is 2 seconds — the process is dying almost immediately, which already rules out "slow startup hitting a liveness probe" and points at an immediate crash on boot. Exit code 1 plus `--previous` logs would confirm.

### Reading exit codes

| Exit Code | Signal | Meaning | Where to look |
|---|---|---|---|
| 0 | — | Clean exit — unusual for a long-running pod, means the main process returned normally (often a misconfigured entrypoint/command, or the app was only ever meant to run once) | Check `command`/`args`, check if this is actually a Job masquerading as a Deployment |
| 1 | — | Generic application error, unhandled exception during startup or runtime | `kubectl logs --previous` — stack trace will be there |
| 137 | SIGKILL (128+9) | Either OOMKilled by the cgroup, or something sent a hard `kill -9` (rare, usually kubelet-initiated after a grace period timeout) | Check `Reason` field right above/next to Exit Code — `OOMKilled: true` confirms memory; if false, check for external kill (VPA, chaos tooling, manual) |
| 139 | SIGSEGV (128+11) | Segfault — native code fault, bad memory access. Common in Go/Rust programs calling into cgo/C libraries, or corrupted binaries, or CPU architecture mismatch (an amd64 binary run under emulation, or an illegal instruction on a CPU without a required extension) | Application-level crash dump / core dump if enabled; check if this started after a base image or dependency upgrade |
| 143 | SIGTERM (128+15) | Graceful shutdown was requested and the process honored it. This is **not inherently a bug** — it's what happens on every normal rolling update. It becomes a symptom of a bug when it happens in a loop outside of deploys, which usually means a **failing liveness probe** is triggering repeated kills | Check `Events` for `Unhealthy` / `Killing` messages referencing liveness probe; check probe timing math (below) |
| 1 (from OOM in some runtimes) | — | Some language runtimes (JVM primarily) catch the OOM condition internally and exit(1) with their own OutOfMemoryError before the kernel even gets a chance to SIGKILL — this looks like exit code 1 in `describe pod`, not 137, and can be confused with an application bug | Check `kubectl logs --previous` for `java.lang.OutOfMemoryError`, distinguish from a genuine heap sizing problem via `-Xmx` vs container memory limit mismatch |

### Root cause 1 — App crash on startup (bad config / missing env var / unhandled exception)

Symptom: exit code 1, `Started`/`Finished` gap of a few seconds, restart count climbing steadily. `kubectl logs --previous` shows an actual stack trace, a `KeyError`, `undefined is not a function`, a config parser error, or a "required environment variable X not set" message from the app's own bootstrap code.

This is the most common category and the fix is almost always outside Kubernetes — a bad deploy, a config value that wasn't updated for the new version, a required field added to a schema without a corresponding ConfigMap update. The Kubernetes-side diagnostic contribution here is limited to confirming the env vars/mounts are what you *expect* — cross-check `describe pod`'s `Environment` and `Mounts` blocks against what the app's own code expects.

### Root cause 2 — Missing or misconfigured Secret/ConfigMap mount

Symptom differs depending on whether the reference is optional:

```
Warning  Failed     8s (x4 over 45s)  kubelet  Error: couldn't find key db_password in Secret payments/payments-secret
```

or, if the Secret/ConfigMap object itself doesn't exist:

```
Warning  Failed     10s (x6 over 1m)  kubelet  Error: secret "payments-secret" not found
```

Note this shows up as a **pod-level event before the container even starts** (state `CreateContainerConfigError`, not `CrashLoopBackOff`) when it's an env-from-secret reference. It only manifests as an actual `CrashLoopBackOff` when the Secret exists and mounts fine, but the *value* is wrong (e.g., wrong password, expired token) and the app itself crashes trying to use it — that failure mode looks identical to root cause 1 from Kubernetes' perspective; you have to read app logs to tell them apart.

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[0].state}'
```

If you see `CreateContainerConfigError` instead of `CrashLoopBackOff`, you're in this category, not a runtime crash — go straight to checking the Secret/ConfigMap exists and has the right keys:

```bash
kubectl get secret payments-secret -n payments -o jsonpath='{.data}' | jq 'keys'
```

### Root cause 3 — Misconfigured liveness probe killing an otherwise-healthy app

This is the one that costs the most debugging hours because logs look *clean*. The app boots, logs "server started on :8080", then a few seconds later gets SIGTERM (exit 143), restarts, boots again, same thing. Nothing in the app logs looks like an error — because there isn't an application bug.

The math that causes this:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 1
```

Total grace period before the kubelet gives up and kills the container: `initialDelaySeconds + (failureThreshold * periodSeconds)` = 5 + 30 = 35 seconds. If the app — because of JVM class loading, a slow dependent connection pool warmup, Spring context initialization, or just a large container image with a cold page cache — actually takes 45 seconds to become ready, the liveness probe will have already failed 3 times and the kubelet kills the container at second 35, restarting the whole cycle. The app never gets a chance to fully start, forever.

Confirm via the `Unhealthy` events:

```
Warning  Unhealthy  1m (x3 over 1m)   kubelet  Liveness probe failed: Get "http://10.244.2.17:8080/healthz": dial tcp 10.244.2.17:8080: connect: connection refused
Normal   Killing    50s               kubelet  Container payments-api failed liveness probe, will be restarted
```

The fix is not to make the app boot faster (though that helps) — it's to separate the concerns Kubernetes gives you for exactly this: use a `startupProbe` with generous timing to cover slow boots, and let `livenessProbe` only kick in after startup succeeds:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2      # up to 60s to start before liveness even begins evaluating
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 1
```

Also distinguish liveness from readiness here: a failing **readiness** probe does not restart the container — it only pulls it out of Service endpoints. If you're seeing restarts, it's liveness (or the container process itself exiting). If you're seeing a pod stuck `Running` but never receiving traffic, it's readiness — check `kubectl get endpoints`, not `describe pod` restart counts.

### Root cause 4 — OOMKilled

Covered in full in the next section, but as a CrashLoopBackOff signature: exit code 137, `Reason: OOMKilled` appears explicitly next to it (this is the tell — 137 alone is ambiguous, `OOMKilled: true` is not).

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      Fri, 18 Jul 2026 09:12:44 +0530
  Finished:     Fri, 18 Jul 2026 09:14:51 +0530
```

Note the gap here is over 2 minutes, unlike the immediate-crash pattern — the process ran for a while, did work, and then was killed once it crossed the memory ceiling. That gap is itself diagnostic: instantaneous OOM (seconds after start) usually means the limit is simply too low for even baseline allocation (e.g., JVM heap + metaspace + thread stacks exceeding the container limit before a single request is served); OOM after minutes/hours under load usually means either a genuine leak or a limit sized for average load but not peak/burst load.

---

## OOMKilled Deep Dive

### The mechanism

A container's `resources.limits.memory` is translated by the container runtime into a cgroup memory controller limit — `memory.limit_in_bytes` on cgroup v1 hosts, `memory.max` on cgroup v2 hosts (most current distros — check with `cat /sys/fs/cgroup/cgroup.controllers` or `mount | grep cgroup` on the node). This is a hard ceiling enforced by the kernel, not by Kubernetes or the kubelet — Kubernetes just writes the number into the cgroup at container creation.

When the cgroup's memory usage (RSS + page cache attributable to it, roughly) hits this ceiling, the **kernel's OOM killer fires scoped to that cgroup**, not the whole node. It picks a process inside the cgroup (usually the largest allocator, weighted by `oom_score_adj`) and sends it SIGKILL. In a typical single-process container this kills PID 1, which tears down the whole container. This is why a memory-hungry container gets killed without affecting sibling containers on the same node, or even sibling containers in the same pod that are under separate cgroups within the pod cgroup hierarchy.

This is fundamentally different from **node-level memory pressure eviction**, which is a kubelet-driven, proactive, whole-pod action taken *before* the node actually runs out of memory — covered below.

### Reading the evidence

```bash
kubectl describe pod <pod> -n <ns>
```

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

That's the confirmation. If you only see `Exit Code: 137` with `Reason: Error` and no `OOMKilled`, it wasn't the cgroup OOM killer — something else sent SIGKILL (rare: a `kubectl delete pod --grace-period=0 --force`, or the container runtime itself killing a hung container after exceeding a stop timeout).

Also check the node's kernel log for corroborating detail the pod description doesn't give you (which process, how much it had allocated):

```bash
kubectl debug node/<nodename> -it --image=busybox -- chroot /host dmesg -T | grep -i 'killed process'
```

```
[Fri Jul 18 09:14:51 2026] Memory cgroup out of memory: Killed process 284913 (java) total-vm:4823012kB, anon-rss:2098176kB, file-rss:1024kB, shmem-rss:0kB, UID:1000 pgtables:5120kB oom_score_adj:994
```

`anon-rss:2098176kB` ≈ 2GB actually resident at the moment of the kill — compare this against the configured limit to see how close/far the process was from a "reasonable" ceiling versus a genuinely runaway allocation.

### Distinguishing a leak from an undersized limit

Single snapshot values don't tell you this — you need a trend.

```bash
kubectl top pod <pod> -n <ns> --containers
```

`kubectl top` only gives a point-in-time snapshot pulled from metrics-server, which itself only retains the most recent scrape — there is no history API here. For trend analysis you need one of:

**Prometheus / cAdvisor**, which is the correct tool for this: graph `container_memory_working_set_bytes{pod="<pod>"}` over the last several days. A leak shows a monotonic staircase that never comes back down even under low load and correlates with time-since-restart, not with request volume. A genuinely undersized limit shows usage tracking load — it climbs during traffic peaks, comes back down afterward, and gets killed specifically when peak traffic coincides with the ceiling (e.g., every day at the same batch-job hour).

```promql
container_memory_working_set_bytes{namespace="payments", pod=~"payments-api.*"}
```

**Direct cgroup inspection on the node** when you don't have Prometheus wired up or need ground truth during a live incident:

```bash
kubectl debug node/<nodename> -it --image=busybox -- chroot /host bash
# cgroup v2:
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/*<pod-uid>*/memory.max
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/*<pod-uid>*/memory.current
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/*<pod-uid>*/memory.events
# cgroup v1:
cat /sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/*/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/*/memory.max_usage_in_bytes
```

`memory.max_usage_in_bytes` (v1) is a running high-water mark since the cgroup was created — extremely useful for confirming "how close did this actually get to the limit historically" without needing a metrics pipeline. `memory.events` (v2) contains an `oom_kill` counter you can watch increment in real time.

The fix is different depending on which pattern you find: a real leak needs an app-side fix (or, tactically, a scheduled restart via a low `maxUnavailable` rollout or a liveness probe tuned to catch it — a band-aid, not a fix); an undersized limit for legitimate peak load needs the limit raised, ideally informed by the actual p99 working-set from the history above, not a guess.

### Cgroup OOM kill vs kubelet eviction (node memory pressure)

These are frequently conflated and they are not the same failure mode:

| | Cgroup OOM kill | Node-pressure eviction |
|---|---|---|
| Trigger | This container's own cgroup hit its `memory.limit`/`memory.max` | Node's overall available memory dropped below the kubelet's configured eviction threshold (`--eviction-hard=memory.available<100Mi` or similar) |
| Who acts | The kernel, instantly, scoped to one cgroup | The kubelet's eviction manager, polling periodically, whole-node scope |
| What gets killed | One process inside one container (usually kills the container) | An entire pod chosen by the eviction manager's ranking (QoS class first — `BestEffort` evicted before `Burstable` before `Guaranteed`; then usage-above-request within a class) |
| Pod status afterward | `CrashLoopBackOff` with restart count incrementing on the same pod object | Pod `Status: Failed`, `Reason: Evicted`, and it is **not restarted in place** — the ReplicaSet controller creates a brand-new pod object elsewhere |
| Evidence | `Last State: Terminated, Reason: OOMKilled` on the container | `kubectl get pod` shows `STATUS: Evicted`; `kubectl describe pod` shows `Status: Failed`, message like `The node was low on resource: memory. Container ... was using 512Mi, which exceeds its request of 128Mi` |

```bash
kubectl get pods -n <ns> --field-selector=status.phase=Failed
```

```
NAME                          READY   STATUS    RESTARTS   AGE
batch-worker-6c9d8f7b4-p2xkq  0/1     Evicted   0          2h
```

```bash
kubectl describe node <node> | grep -A5 Conditions
```

```
Conditions:
  Type             Status  ...  Reason                       Message
  ----             ------  ...  ------                       -------
  MemoryPressure   True    ...  KubeletHasInsufficientMemory  kubelet has insufficient memory available
```

If you see `MemoryPressure: True` on the node at the same time your pods are evicting, that's your confirmation it's a node-level condition, not a per-container limit problem — the fix is node capacity, workload rightsizing across the whole node, or `Guaranteed` QoS for pods that must survive pressure (requests == limits), not adjusting any single pod's own memory limit.

---

## ImagePullBackOff — Mapping Error Message to Fix

`ImagePullBackOff`/`ErrImagePull` events give you the *exact* underlying registry error in the `Events` section — this is one of the few failure categories where you almost never need logs or exec, because the event message is the root cause verbatim.

```bash
kubectl describe pod <pod> -n <ns>
```

**Wrong tag / typo:**
```
Warning  Failed     8s (x3 over 40s)  kubelet  Failed to pull image "registry.internal/payments-api:v2.4.99": rpc error: code = NotFound desc = failed to pull and unpack image "registry.internal/payments-api:v2.4.99": failed to resolve reference: registry.internal/payments-api:v2.4.99: not found
```
Fix: the tag genuinely doesn't exist in the registry — check CI/CD, check if the tag got deleted by a retention policy, check for a copy-paste typo in the manifest.

**Private registry auth failure:**
```
Warning  Failed     5s (x2 over 20s)  kubelet  Failed to pull image "registry.internal/payments-api:v2.4.1": rpc error: code = Unknown desc = failed to authorize: failed to fetch anonymous token: unexpected status: 401 Unauthorized
```
Fix: missing `imagePullSecrets` on the pod/ServiceAccount, wrong credentials in the secret, or an expired token (very common with cloud registries issuing short-lived tokens — ECR tokens expire every 12 hours, GCR/Artifact Registry service account keys can be revoked). Verify:
```bash
kubectl get sa <sa-name> -n <ns> -o jsonpath='{.imagePullSecrets}'
kubectl get secret <pull-secret> -n <ns> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

**Registry rate limiting (the classic Docker Hub incident):**
```
Warning  Failed     3s (x5 over 1m)  kubelet  Failed to pull image "nginx:latest": rpc error: code = Unknown desc = toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
This is a well-known real-world failure pattern: unauthenticated pulls from Docker Hub are rate-limited per anonymous IP (historically 100 pulls/6h), and a cluster with many nodes all pulling the same public base image through a shared NAT gateway IP exhausts this limit fast, especially during a mass rollout or a node pool scale-up event where every new node cold-pulls the same images simultaneously. Fix: authenticate pulls (even a free Docker Hub login raises the limit substantially), or better, mirror external images into your own private registry/pull-through cache so external rate limits never gate an internal deploy.

**Manifest not found for the platform/arch:**
```
Warning  Failed     4s (x2 over 15s)  kubelet  Failed to pull image "registry.internal/payments-api:v2.4.1": rpc error: code = NotFound desc = no match for platform in manifest: not found
```
Happens on mixed-architecture clusters (amd64 + arm64 nodes, common with Graviton/ARM node pools for cost) when the image was built and pushed for only one architecture but the pod landed on a node of the other architecture. Fix: build and push a multi-arch manifest list (`docker buildx build --platform linux/amd64,linux/arm64`), or constrain the pod to the correct node arch via `nodeSelector: kubernetes.io/arch: amd64` as a stopgap.

**Network/firewall blocking egress to the registry:**
```
Warning  Failed     6s (x4 over 1m)  kubelet  Failed to pull image "registry.internal/payments-api:v2.4.1": rpc error: code = Unknown desc = failed to resolve reference: failed to do request: Head "https://registry.internal/v2/payments-api/manifests/v2.4.1": dial tcp: lookup registry.internal on 10.96.0.10:53: read udp: i/o timeout
```
Note this one manifests as a DNS timeout, not an HTTP error — the request never even got a response from the registry. This points at egress network policy blocking the node (not the pod — image pulls happen via the container runtime on the node, outside any pod-scoped NetworkPolicy), a firewall rule change, or the registry endpoint itself being unreachable/down. Confirm from the node directly:
```bash
kubectl debug node/<nodename> -it --image=busybox -- chroot /host curl -v https://registry.internal/v2/
```

---

## Pending Pods — Quick Triage Checklist

Full scheduling internals belong in a dedicated scheduling doc; here's the triage cross-reference for when a pod won't leave `Pending`:

```bash
kubectl describe pod <pod> -n <ns> | grep -A10 Events
```

- **Insufficient resources**: `0/12 nodes are available: 12 Insufficient cpu` — no node has enough allocatable CPU/memory left for the requested amount. Check `kubectl describe node <candidate>` `Allocatable` vs sum of existing `Requests`.
- **Unschedulable taints**: `0/12 nodes are available: 12 node(s) had untolerated taint {dedicated: gpu}` — pod is missing the matching `tolerations`, or you intended it for a different node pool.
- **PVC stuck Pending**: pod stays `Pending` with event `pod has unbound immediate PersistentVolumeClaims` — check the PVC itself, not the pod:
```bash
kubectl get pvc <pvc> -n <ns>
kubectl describe pvc <pvc> -n <ns>
```
  Two distinct causes here: no `StorageClass`/provisioner exists to satisfy the claim (`no persistent volumes available for this claim and no storage class is set` — a cluster config gap), or the StorageClass uses `volumeBindingMode: WaitForFirstConsumer` by design, meaning the PVC will legitimately stay `Pending` **until a pod referencing it is scheduled** — this is normal and resolves itself the moment the pod is scheduled, don't chase it as a bug in isolation; check if the pod itself is also stuck for an unrelated scheduling reason first.

---

## Service Not Reachable — Full Checklist

Work this top to bottom; each step is cheaper than the next and rules out an entire category.

**1. Are there populated endpoints?**
```bash
kubectl get endpoints <svc> -n <ns>
```
```
NAME       ENDPOINTS   AGE
payments   <none>      3h
```
Empty endpoints is the single most common "service not reachable" cause and it's a two-second check. It means either no pods match the selector, or matching pods exist but are all failing readiness (a pod not `Ready` is never added to endpoints, even if it's `Running`).

**2. Does the selector actually match pod labels?**
```bash
kubectl get svc <svc> -n <ns> -o jsonpath='{.spec.selector}'
kubectl get pods -n <ns> --show-labels
```
Classic mismatch: Service selects `app: payments-api`, pods are labeled `app.kubernetes.io/name: payments-api` from a Helm chart upgrade that changed label conventions, or a stray `version: v2` selector left over from a canary that no longer matches the stable pod set.

**3. Is readiness actually passing on the backing pods, if endpoints are empty for that reason?**
```bash
kubectl get pods -n <ns> -l app=payments-api
```
```
NAME                          READY   STATUS    RESTARTS   AGE
payments-api-7d8f9c6b45-x2kqz 0/1     Running   0          5m
```
`0/1` Ready with `Running` status confirms the readiness probe is failing — go back to `describe pod` events for the `Unhealthy` readiness message.

**4. targetPort vs port vs the container's actual listening port**
```bash
kubectl get svc <svc> -n <ns> -o yaml | grep -A3 ports
```
`port` is what clients use to reach the Service; `targetPort` must match the port the container process is *actually bound to* inside the pod, not the `containerPort` declared in the pod spec (that field is documentation only, not enforced) and not the `port` field. A common bug: app listens on `3000`, Dockerfile `EXPOSE 8080`, pod spec `containerPort: 8080`, Service `targetPort: 8080` — everything *looks* consistent but the process never bound 8080, so connections get refused. Confirm directly:
```bash
kubectl exec -it <pod> -n <ns> -- sh -c "netstat -tlnp || ss -tlnp"
```

**5. NetworkPolicy blocking traffic — check both ends**
```bash
kubectl get networkpolicy -n <destination-ns> -o yaml
kubectl get networkpolicy -n <source-ns> -o yaml
```
NetworkPolicies are namespace-scoped and additive-restrictive: once *any* policy selects a pod for `Ingress`, all traffic not explicitly allowed by some policy is denied — a default-deny in the destination namespace with no matching allow rule for the source namespace/pod-selector silently drops traffic with no event, no log, nothing in `describe pod`. This is invisible from the Kubernetes API layer entirely; you only find it by reading policy YAML or testing connectivity directly. Also check **egress** policies on the *source* pod's namespace — a default-deny-egress policy with no matching egress rule to the destination blocks the request before it even leaves the source pod, independent of anything on the destination side.

**6. DNS resolution**
```bash
kubectl exec -it <pod> -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local
```
```
Server:    10.96.0.10
Address:   10.96.0.10:53

** server can't find payments.prod.svc.cluster.local: NXDOMAIN
```
`NXDOMAIN` here with a correct-looking name usually means either a typo'd namespace in the query, or CoreDNS itself is unhealthy — check `kubectl get pods -n kube-system -l k8s-app=kube-dns` and `kubectl logs -n kube-system -l k8s-app=kube-dns` for CoreDNS crash loops or upstream resolution failures, which affect the *entire cluster's* DNS, not just this one lookup — a strong signal to check other pods too before assuming this is isolated.

---

## Node NotReady

```bash
kubectl describe node <node> | grep -A20 Conditions
```

```
Conditions:
  Type                 Status  Reason                       Message
  ----                 ------  ------                       -------
  MemoryPressure       False   KubeletHasSufficientMemory    kubelet has sufficient memory available
  DiskPressure          True   KubeletHasDiskPressure         kubelet has disk pressure
  PIDPressure           False  KubeletHasSufficientPID        kubelet has sufficient PID available
  Ready                 False  KubeletNotReady                container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

Each condition maps to a distinct root cause and a distinct fix — read all four, don't stop at `Ready: False`:

- **`DiskPressure: True`** — node's filesystem (or imagefs, tracked separately) has crossed the kubelet's eviction threshold. Check what's consuming disk: unbounded container logs without rotation, orphaned image layers (`crictl rmi --prune`), or a genuinely full data volume. `df -h` on the node via a debug pod is the direct check.
- **`PIDPressure: True`** — process/thread table exhaustion, usually a fork bomb in a misbehaving container or a container leaking zombie processes because PID 1 in the container isn't reaping children (classic "app run directly as PID 1 without an init process" bug — fix with `tini` or `--init`).
- **`NetworkPluginNotReady`** — this is the sneaky one: the kubelet process itself is alive and responding, but it refuses to report `Ready` because the CNI hasn't initialized — usually a missing or malformed CNI config in `/etc/cni/net.d/` on that node, or the CNI daemonset pod (Calico/Cilium/etc.) itself crashing on that node. The node *looks* structurally fine (kubelet up, API server can talk to it) but is functionally useless because no pod networking can be set up on it — new pods scheduled there will hang in `ContainerCreating` forever. Check:
```bash
kubectl get pods -n kube-system -o wide | grep <node>
```
  looking specifically for the CNI daemonset pod on that node being `CrashLoopBackOff` or missing entirely.

**If the node doesn't respond to `kubectl describe node` conditions at all being fresh** (stale `LastHeartbeatTime`, conditions frozen from hours ago) — the kubelet process itself is likely down or the node has lost network connectivity to the API server entirely. This requires node access, not API access:
```bash
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 200 --no-pager
```
Common findings here: kubelet crashed on a bad config after a manual change, certificate rotation failure (expired client cert, kubelet can no longer authenticate to the API server — look for `x509: certificate has expired` in the journal), or the node ran out of disk space for the kubelet's own state directory.

---

## Useful Debugging Commands

### Node debug pod with host filesystem access

```bash
kubectl debug node/<nodename> -it --image=busybox
```

This schedules a privileged pod on the target node with the node's root filesystem bind-mounted at `/host`. Use it whenever you need to inspect node-level state (cgroups, `/var/log`, CNI config, container runtime state) but don't have SSH access, or SSH is disabled by policy in a managed environment (common on GKE Autopilot, EKS with restricted node access). Once inside:
```bash
chroot /host
cat /sys/fs/cgroup/.../memory.max
ls /etc/cni/net.d/
journalctl -u kubelet --no-pager | tail -100
```

### Ephemeral debug containers for shell-less images

```bash
kubectl debug <pod> -it --copy-to=<pod>-debug --container=<container> --image=busybox --share-processes -n <ns>
```

For distroless or `scratch`-based production images with no shell, `kubectl exec` fails outright (`exec: "sh": executable file not found in $PATH`). `--copy-to` creates a new pod that's a copy of the original but with the target container's image swapped for a debug image (or an additional ephemeral container added, depending on flag choice), letting you inspect the filesystem and, critically, with `--share-processes`, see the original container's process namespace — you can `ps aux` and see the actual app process, inspect its `/proc/<pid>/environ`, open file descriptors, and mapped memory, all from the debug container, without ever needing a shell inside the production image itself.

### CRI-level inspection when kubectl itself is unreliable

When the kubelet or API server is degraded enough that `kubectl` commands against a node are timing out or returning stale data, drop a layer below Kubernetes entirely and talk to the container runtime directly via `crictl` (requires node access):

```bash
crictl ps -a
crictl inspect <container-id>
crictl logs <container-id>
crictl pods
```

`crictl ps -a` shows containers the runtime knows about, including ones the kubelet may have lost track of, which is useful for confirming whether a discrepancy is a kubelet reporting bug versus an actual runtime-level container state. `crictl inspect` gives you the raw OCI runtime spec actually applied — including the exact cgroup limits, mounts, and env vars as the runtime received them, which is the ground truth when you suspect the kubelet translated the pod spec incorrectly.

### kubelet-level errors invisible to the API

```bash
journalctl -u kubelet -f
```

Some failures never make it into any object's `status` or `events` at all — a kubelet failing to even *create* a pod sandbox due to a CNI error before the pod object gets any meaningful status update, or repeated failed attempts to pull the kubelet's own config, or cgroup driver mismatches (`cgroupfs` vs `systemd` — a classic kubelet-vs-containerd config mismatch that causes every pod on that node to fail sandbox creation with an error that only ever appears in this journal, never in `kubectl describe pod`).

---

## Performance Debugging

### The limitation of `kubectl top`

```bash
kubectl top pod -n <ns>
kubectl top node
```

Both pull from metrics-server, which itself scrapes cAdvisor/kubelet stats on a short interval (default 15-60s depending on config) and retains **no history** — every call to `kubectl top` shows only the most recent scrape. There is no way to ask `kubectl top` "what was CPU usage 20 minutes ago" — for anything beyond "what's happening right now," you need a real metrics backend (Prometheus + cAdvisor metrics, or a managed equivalent) with retention and query capability (PromQL range queries, Grafana dashboards).

### CPU throttling below the limit — the senior-level insight

The counterintuitive case that trips up most people: `kubectl top pod` shows a pod using 40% of its CPU limit on average, yet the application is visibly slow and profiling shows time spent waiting, not computing. The average is misleading because **CFS (Completely Fair Scheduler) quota enforcement operates per 100ms period, not as a smooth amortized rate**.

When you set `resources.limits.cpu: 500m`, the kernel translates this into a CFS quota: `cpu.cfs_quota_us = 50000` and `cpu.cfs_period_us = 100000` (a 100ms period, standard default) — meaning the container's threads are collectively allowed 50ms of CPU execution time out of every 100ms wall-clock window. If the workload is bursty — say, it does bursts of computation handling a batch of requests, using 5ms of CPU in most 100ms windows but occasionally needing 80ms of CPU within a single 100ms window to handle a request spike — that single window throttles hard (the process is paused for the remainder of that period once it exhausts its 50ms quota), even though the *average* over a full second looks like only 40-50% utilization. The averaging in `kubectl top`'s snapshot (or even a 1-minute Prometheus rate) completely hides this, because it smooths out the per-period spikes that are the actual cause of latency.

This is measurable directly via the cgroup's own throttling accounting, which is period-level, not averaged:

```bash
kubectl debug node/<nodename> -it --image=busybox -- chroot /host bash
cat /sys/fs/cgroup/kubepods.slice/.../cpu.stat
```

```
usage_usec 8231044213
user_usec 6182031044
system_usec 2049013169
nr_periods 481200
nr_throttled 96812
throttled_usec 40213558102
```

`nr_throttled / nr_periods` here is roughly 20% — one in five scheduling periods, this container was throttled at least once. `throttled_usec` (40+ seconds cumulative) is real added latency the application experienced that never shows up as "CPU usage" in any point-in-time metric, because a throttled process reports 0% usage *while it's paused*, dragging the average down even further and making the problem look like the opposite of what it is.

In Prometheus, this is exposed as:

```promql
rate(container_cpu_cfs_throttled_periods_total{pod="<pod>"}[5m])
/
rate(container_cpu_cfs_periods_total{pod="<pod>"}[5m])
```

A non-trivial ratio here (even 5-10%) on a latency-sensitive service is a real finding worth acting on, independent of what the average CPU usage graph shows. The two standard fixes: raise the CPU limit (giving more headroom per period so bursts don't exhaust quota), or — often more effective for genuinely bursty workloads — remove the CPU limit entirely and rely on `requests` alone plus the node's CFS *shares* for relative scheduling priority, accepting that the pod can use spare node capacity during bursts instead of getting hard-throttled at an arbitrary ceiling. The tradeoff is noisy-neighbor risk on shared nodes, which is why this decision needs to be made per-workload, not as a blanket policy — latency-sensitive request-serving pods often benefit from no CPU limit at all, while batch/background jobs are exactly where you want a hard limit to protect co-located workloads.
