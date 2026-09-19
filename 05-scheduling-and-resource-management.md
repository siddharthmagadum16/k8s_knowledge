# Scheduling and Resource Management

## 1. Requests vs Limits

Every container in a Pod can declare `resources.requests` and `resources.limits` for `cpu` and `memory` (and extended resources like `nvidia.com/gpu`). These serve two entirely different purposes — one is a **scheduling input**, the other is a **runtime enforcement ceiling**.

```yaml
resources:
  requests:
    cpu: "500m"        # 0.5 CPU core — used by the scheduler to find a fitting node
    memory: "256Mi"
  limits:
    cpu: "1"            # hard cap enforced at runtime via cgroups
    memory: "512Mi"
```

### 1.1 Requests: what the scheduler uses

- `requests` is the amount of a resource the container is guaranteed to get, and it's what the **scheduler** sums up per node to decide if a Pod fits: it adds up the requests of all pods already on a node and checks whether the node's allocatable capacity has room for the new Pod's requests. A node is never allowed to be over-committed on requests — this is a hard scheduling constraint.
- Setting requests too low relative to actual usage causes overcommitment risk (many pods scheduled onto a node that later can't actually give each of them what they need under load); setting them too high wastes capacity (nodes look "full" to the scheduler while actually idle).
- If `requests` is omitted, it typically defaults to being equal to `limits` (if limits is set) or to an admission-controller/`LimitRange` default, or effectively `0` if nothing is set at all (dangerous — a Pod with zero requested resources is treated as needing almost nothing for scheduling purposes, yet can still consume real resources up to its limit or unboundedly if no limit is set either).

### 1.2 Limits: what the kernel/kubelet enforces

- `limits` is enforced by the **container runtime via Linux cgroups**, not by the scheduler. The scheduler doesn't care about limits when placing Pods (aside from validating limits ≥ requests) — limits only matter once the container is actually running.
- **CPU limit enforcement**: CPU is a *compressible* resource. Exceeding the CPU limit does not kill the container — the kernel's CFS (Completely Fair Scheduler) bandwidth controller **throttles** it: the container is allowed to use up to its limit within each scheduling period (default 100ms) and then has its CPU access paused for the remainder of that period once it hits the quota, resuming next period. This shows up as increased latency/response time, not crashes — visible via the container's `cpu.stat` cgroup file (`nr_throttled`, `throttled_time`), a very common and under-diagnosed production issue where p99 latency spikes correlate exactly with hitting the CPU limit.
- **Memory limit enforcement**: memory is *incompressible* — you can't "pause" memory usage. Exceeding the memory limit results in the kernel's **OOM killer** terminating the container outright (`OOMKilled`, exit code 137). The kubelet then restarts it per `restartPolicy`, and repeated OOMKills manifest as `CrashLoopBackOff`.
- There is no such thing as CPU "OOMKill" and no such thing as memory "throttling" — the enforcement mechanisms for the two resource types are fundamentally different, which is why sizing memory limits carefully matters far more (getting it wrong kills the process) than CPU limits (getting it wrong just slows it down).

### 1.3 Practical sizing guidance

- Set `requests` close to real steady-state usage (informed by actual metrics — Prometheus/`kubectl top`), not guesses, so scheduling reflects reality and bin-packing works efficiently.
- Set memory `limits` with real headroom above expected peak usage — an OOMKill is a hard failure; being too conservative here directly causes outages under load spikes (e.g., traffic surge, GC pause build-up, cache growth).
- CPU limits are more debatable — some teams deliberately omit CPU limits (only set requests) to let a container burst above its request when the node has spare capacity, accepting that under real node contention it degrades gracefully via the CFS's fair-share weighting rather than a hard throttle. This has QoS-class implications (below).

## 2. Quality of Service (QoS) Classes

Kubernetes derives a **QoS class** per Pod automatically from how requests/limits are set — you don't set this field directly, it's computed and visible at `pod.status.qosClass`.

### 2.1 Guaranteed

- **Every** container in the Pod has both `requests` and `limits` set for **both** CPU and memory, and for each container, `requests == limits` exactly.
- Gets the strongest resource guarantee — this Pod is essentially "reserved" that exact amount and won't be starved or throttled unexpectedly relative to its own allocation, and is the **last** to be evicted under node resource pressure.
- Typical for critical stateful workloads (databases) where predictable performance matters more than bursting flexibility.

```yaml
resources:
  requests: { cpu: "1", memory: "1Gi" }
  limits:   { cpu: "1", memory: "1Gi" }
```

### 2.2 Burstable

- At least one container has a `requests` or `limits` set for CPU or memory, but the Pod doesn't qualify as `Guaranteed` (e.g., requests < limits, or only some containers/resources specify values).
- Gets a proportional guarantee based on its `requests` but can burst up to its `limits` when spare node capacity exists. Middle priority for eviction.
- The most common class in practice — most application workloads.

```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```

### 2.3 BestEffort

- **No** `requests` or `limits` set at all, for any container, for any resource.
- No guarantee whatsoever — first to be killed under node memory pressure, gets leftover CPU only after Guaranteed/Burstable pods are satisfied.
- Appropriate only for genuinely disposable, non-critical batch work — in practice, admission controls (`LimitRange` with defaults, policy engines like Kyverno/OPA Gatekeeper requiring resources to be set) are commonly used to prevent BestEffort pods from ending up in production namespaces by accident.

### 2.4 Eviction priority under node pressure

When a node runs low on memory (or disk), the kubelet proactively evicts Pods (before the kernel OOM killer has to act indiscriminately) in this order:
1. **BestEffort** pods first.
2. **Burstable** pods whose usage exceeds their `requests` (i.e., the ones actually consuming beyond their guaranteed share) next.
3. **Guaranteed** pods (and Burstable pods within their requests) last, and only if the node is still critically short.

Within the same QoS tier, pods are further ranked by how far over their requested usage they are, and by Pod priority (`priorityClassName`) if configured — higher-priority pods are preferred to survive over lower-priority ones at the same QoS tier.

## 3. Default Scheduler Basics

Covered architecturally in the introduction doc; here's the practical mechanics for steering placement.

### 3.1 Filtering then scoring, recap

- **Filtering**: eliminate nodes that structurally cannot run the Pod — insufficient allocatable CPU/memory/ephemeral-storage versus the Pod's `requests`, node taints without a matching Pod toleration, `nodeSelector`/required node affinity not satisfied, requested hostPort already bound on that node, volume topology mismatches (e.g., a zonal PV that isn't reachable from that node).
- **Scoring**: rank the surviving feasible nodes — e.g., `NodeResourcesFit` scoring (prefer most-requested vs least-requested packing depending on scheduler config, commonly favoring bin-packing to consolidate load and free whole nodes for scale-down), `InterPodAffinity` (favor/avoid co-locating with certain other pods), `ImageLocality` (prefer nodes that already have the image cached, faster start), `PodTopologySpread` (spread replicas across zones/nodes evenly).

### 3.2 nodeSelector

The simplest placement constraint — an exact-match label requirement:

```yaml
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/arch: amd64
```

Node must carry `disktype=ssd` **and** `kubernetes.io/arch=amd64` labels (nodes are labeled via `kubectl label node <node> disktype=ssd`, or automatically for well-known labels like arch/zone/instance-type by the kubelet/cloud-controller-manager). No partial match, no "prefer" semantics — `nodeSelector` is strict AND-of-exact-matches; for anything more expressive (OR conditions, "prefer but don't require", set-based operators like `In`/`NotIn`/`Exists`), you need `nodeAffinity` (a superset capability, not covered exhaustively here, but worth knowing it exists as `spec.affinity.nodeAffinity` with `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`).

### 3.3 Node capacity fields

```bash
kubectl describe node worker-1
```

Shows:
- `Capacity`: total physical resources on the node (all CPU/memory the kernel reports).
- `Allocatable`: capacity minus reservations for the OS and Kubernetes system daemons (`--system-reserved`, `--kube-reserved` kubelet flags) — this is the number the **scheduler actually uses** for fitting Pods, not raw Capacity. A node with 8 vCPU / 32Gi capacity might only have ~7.5 vCPU / 29Gi allocatable after reservations.
- `Allocated resources`: sum of `requests` (and separately `limits`) already claimed by Pods scheduled there — this is what determines remaining headroom for new Pods.

## 4. ResourceQuotas and LimitRanges

Both are namespace-scoped admission-time guardrails, aimed at different problems: **ResourceQuota** caps aggregate consumption across a namespace; **LimitRange** sets per-object defaults/bounds within a namespace.

### 4.1 ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    persistentvolumeclaims: "10"
    requests.storage: 500Gi
    count/deployments.apps: "20"
```

- Enforced as an **admission-time check**: if creating/updating a Pod (or PVC, or a `count/`-tracked object) would push the namespace's aggregate usage past any `hard` limit, the API server rejects the request outright (`403 Forbidden` — "exceeded quota").
- Once **any** ResourceQuota exists in a namespace covering `requests.cpu`/`requests.memory` (or `limits.*`), **every Pod created in that namespace must explicitly specify those requests/limits** — Pods without them are rejected, because the quota controller can't count against an undefined value. This is a common way teams force explicit resource declarations across a namespace without a separate policy engine.
- `count/<resource>.<group>` quota types let you cap the *number* of arbitrary objects (Deployments, Services, Secrets, custom resources), not just compute — useful to prevent namespace sprawl/object-count abuse.
- Check current usage: `kubectl describe resourcequota team-a-quota -n team-a`.

### 4.2 LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "2Gi"
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 100Gi
```

- `default`/`defaultRequest`: if a container omits `limits`/`requests`, the LimitRange **mutates the Pod at admission time** to inject these values automatically — this is exactly how a namespace avoids accidental `BestEffort` pods even when developers forget to specify resources.
- `min`/`max`: hard bounds — a container requesting less than `min` or more than `max` is rejected outright at admission.
- Also supports a `ratio` (max limit-to-request ratio, capping how much a container is allowed to burst relative to its guaranteed request) and separate limit types for `Pod` (aggregate across all its containers) and `PersistentVolumeClaim` (min/max storage size).
- LimitRange and ResourceQuota are complementary and commonly deployed together: LimitRange ensures every Pod has *sane* individual defaults/bounds, ResourceQuota ensures the *namespace as a whole* can't exceed its allotted share of the cluster.

## 5. Horizontal Pod Autoscaler (HPA)

HPA automatically adjusts a workload's `replicas` count based on observed metrics, closing the loop between load and capacity without a human changing `replicas` by hand.

### 5.1 metrics-server dependency

- The HPA controller (part of kube-controller-manager) needs a source of resource utilization metrics. For basic CPU/memory-based scaling, this comes from **metrics-server** — a lightweight cluster add-on that scrapes each kubelet's `/metrics/resource` (Summary API) endpoint and exposes aggregated Pod/Node resource usage via the **Metrics API** (`metrics.k8s.io`), which HPA queries.
- metrics-server is **not installed by default** on most self-managed clusters (though most managed offerings include it) — without it, `kubectl top nodes`/`kubectl top pods` fail and CPU/memory-based HPAs cannot function at all (they'll show `<unknown>` for current metrics and never scale).
- For custom application-level metrics (queue depth, requests-per-second, custom business metrics), HPA instead queries the **Custom Metrics API** or **External Metrics API**, typically served by an adapter like the Prometheus Adapter, translating PromQL queries into the API shape HPA expects.

### 5.2 Basic CPU-utilization HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70     # target: average 70% of requested CPU across all pods
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min of sustained low usage before scaling down
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
```

- **`averageUtilization: 70` for CPU is a percentage of the Pod's `requests.cpu`**, not of node capacity or an absolute value — this is precisely why setting accurate `requests` is a prerequisite for HPA to behave sensibly: if `requests.cpu` is set too low, 70% of a tiny number triggers scale-out almost immediately under any load; too high, and the HPA never reacts until things are already in trouble.
- Desired replica calculation, conceptually: `desiredReplicas = ceil(currentReplicas * (currentMetricValue / targetMetricValue))`. The controller re-evaluates on a periodic sync (`--horizontal-pod-autoscaler-sync-period`, default 15s).
- **`behavior`** stanzas (stabilization windows, scale-up/down policies) prevent flapping — without a scale-down stabilization window, a brief dip in load can trigger an immediate scale-down followed by an immediate scale-up on the next spike, causing thrashing (repeated Pod churn, cold-start latency on every cycle). Scale-up is intentionally allowed to react fast (protect availability); scale-down is intentionally conservative (protect against flapping/premature capacity removal).
  - **`stabilizationWindowSeconds`**: before acting, look back over this many seconds of *past desired-replica-count calculations* and pick the value that resists the direction of change (the highest recent value for scale-down, the lowest for scale-up) instead of reacting to just the latest instant. `scaleDown: 300` above means "only scale down based on the least-aggressive (highest) replica count computed in the last 5 minutes" — so one brief dip doesn't trigger an immediate drop. `scaleUp: 0` means no lookback at all — react to the very latest metric reading immediately.
  - **`policies`** cap how much change is allowed per time window, once scaling has been triggered — a rate limiter, applied *after* the stabilization window decides *whether* to act at all:
    - `type: Percent` (used in `scaleDown` above): the change is capped as a percentage of current replica count. `value: 25, periodSeconds: 60` = remove at most 25% of current replicas every 60 seconds (e.g., 20 replicas → drops to a minimum of 15 in one step, then re-evaluates on the next period, not straight to whatever the ideal count is).
    - `type: Pods` (used in `scaleUp` above): the change is capped as an absolute pod count instead of a percentage. `value: 4, periodSeconds: 60` = add at most 4 pods every 60 seconds, regardless of current replica count.
    - If multiple policies are listed under the same direction, HPA picks the one that allows the **largest** change by default (`selectPolicy: Max`, the default) — or the smallest via `selectPolicy: Min`, giving fine control over how conservative or aggressive scaling is allowed to be.
    - This is why the example is asymmetric on purpose: scale-up uses a fast, no-wait, absolute-pod-count policy (react immediately, add capacity fast to protect availability during a spike); scale-down uses a slow, 5-minute-averaged, percentage-based policy (shed capacity gradually, only once low usage is clearly sustained, to avoid flapping).
- Multiple `metrics` entries can be specified simultaneously (e.g., CPU **and** a custom requests-per-second metric) — HPA computes desired replicas for each independently and takes the **maximum** across all of them, so any single metric under pressure can drive scale-out.
- HPA only ever adjusts `spec.replicas` — it has no concept of node capacity; if the cluster's nodes don't have room to schedule the new replicas, they'll simply sit `Pending` until either the Cluster Autoscaler (a separate, node-level autoscaling component watching for unschedulable Pods and provisioning new nodes) adds capacity, or existing Pods free up room.
- Cannot be combined with manual `kubectl scale` on the same Deployment in a stable way — HPA will simply override manual replica changes on its next reconciliation if its target metric implies a different count; treat HPA-managed Deployments' `replicas` field as owned by the HPA once one is attached.

### 5.3 Relationship to Vertical Pod Autoscaler (brief mention)

Where HPA changes the **number** of replicas, **VPA** (a separate, not-built-in-by-default component) adjusts the **requests/limits** of existing Pods based on observed usage history, recreating Pods with revised resource values. HPA and VPA can conflict if both target CPU/memory on the same workload simultaneously (VPA changing requests invalidates the utilization baseline HPA is scaling against) — common guidance is to use VPA only in "recommendation" mode alongside an active HPA, or use HPA on CPU while VPA (in `Auto`/`Recreate` mode) manages memory only, keeping their axes of control separate.
