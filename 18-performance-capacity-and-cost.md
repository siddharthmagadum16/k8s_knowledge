# Performance, Capacity, and Cost

## 1. Resource request/limit tuning methodology

### The "set it high to be safe" anti-pattern

The most common thing junior and mid-level engineers do when writing a Deployment manifest is guess at resource requests, then pad the guess "for safety":

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

The app in question, under real p99 load, uses 200m CPU and 300Mi memory. This looks harmless — the pod isn't crashing, latency is fine, nobody complains. But it is quietly expensive, for a reason most engineers never connect to the symptom they see (large clusters, poor autoscaler efficiency, high cloud bill):

**The scheduler places pods based on requests, not actual usage.** `kube-scheduler`'s `NodeResourcesFit` plugin checks whether a node has enough *allocatable* capacity left after subtracting the sum of *requests* of pods already bound to it. It never looks at real-time cgroup usage. A node with 8 vCPU allocatable can hold exactly 4 pods that each request 2 CPU — even if all four are sitting at 200m actual usage. The other 7.2 CPU of that node sits idle, invisible to the scheduler because it's already "spoken for."

Run the numbers on a real cluster: 50 replicas of a service requesting 2 CPU / 4Gi, each actually using 200m / 300Mi:

```
Requested (drives scheduling):   50 * 2 CPU  = 100 vCPU,  50 * 4Gi = 200Gi
Actual usage (drives nothing):   50 * 0.2CPU =  10 vCPU,  50 * 0.3Gi=  15Gi
Waste ratio:                     10x CPU, 13x memory
```

On m5.2xlarge nodes (8 vCPU / 32GiB allocatable ~7.4 vCPU/29GiB after system-reserved), memory is usually the binding constraint here: 200Gi of memory requests needs at least 7-8 such nodes just for this one service, when 1 node could physically run the real workload. Multiply this pattern across dozens of services in a platform and you get clusters that are 3-5x larger than necessary, cluster autoscaler that never scales down (because every node has *some* pod with high requests pinning it), and a bill that scales with imagined peak, not reality.

This is why "nothing is on fire" is not evidence that requests are tuned correctly. The failure mode of over-requesting is invisible in application metrics and only shows up in infra cost and node count.

### VPA in recommendation-only mode

The correct way to size requests is to measure, not guess. The Vertical Pod Autoscaler's `recommender` component does this for you: it watches historical container usage (via metrics-server samples, aggregated into decaying histograms per container) and computes percentile-based recommendations continuously.

Deploy VPA with `updateMode: "Off"` — this **only produces recommendations, it never touches running pods**:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: checkout-api-vpa
  namespace: payments
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: checkout-api
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "4"
          memory: 8Gi
```

Let it run against production traffic for **1-2 weeks minimum** — you need to capture peak business hours, batch jobs, deploys (which cause usage spikes), and at least one weekly cycle. Then read the recommendation:

```bash
kubectl describe vpa checkout-api-vpa -n payments
```

```
Recommendation:
  Container Recommendations:
    Container Name:  checkout-api
    Lower Bound:
      Cpu:     110m
      Memory:  380Mi
    Target:
      Cpu:     150m
      Memory:  410Mi
    Uncapped Target:
      Cpu:     150m
      Memory:  410Mi
    Upper Bound:
      Cpu:     640m
      Memory:  1200Mi
```

How to read this:

- **Target** — VPA's best-guess recommendation, roughly the observed p90-ish usage with a safety margin baked into VPA's internal algorithm. This is what you'd set as the **request**.
- **Lower Bound** — the floor below which VPA is fairly confident the container will get resource-starved if requests are set this low. Never set requests below this.
- **Upper Bound** — a conservative ceiling that accounts for usage growth/spikes; a reasonable candidate for the **limit** (or `2x target` if the upper bound looks too tight for your traffic variance).
- **Uncapped Target** — target ignoring `maxAllowed`/`minAllowed` policy clamps; check this differs from Target only if you've set policy bounds.

### The workflow (this is the actual senior-level practice)

1. Deploy VPA in `Off` mode next to every workload you suspect is misconfigured (start with anything requesting >500m CPU or >1Gi memory, since that's where the $ leverage is).
2. Wait 1-2 weeks, through at least one full business cycle.
3. Harvest `Target` and `Upper Bound` from `kubectl describe vpa`.
4. **Manually** edit the Deployment's `resources.requests`/`limits` to those values. This is a deliberate, reviewed, human action — not automation.
5. Either delete the VPA object, or keep it running in `Off` mode indefinitely as a passive "drift detector" — if Target creeps far from what you set, you get a signal to re-tune next quarter.

### Why not `updateMode: "Auto"`

`Auto` mode sounds like the obvious next step — let VPA apply its own recommendations. It cannot resize a running container's cgroup limits in place (in-place pod resize is a separate, newer, still-maturing feature gate — `InPlacePodVerticalScaling` — and even where enabled, VPA does not use it by default as of common production versions). Instead, Auto mode **evicts and recreates the pod** with new requests whenever the recommendation drifts meaningfully.

This is dangerous for:

- **Stateful workloads** — a Kafka broker, a Postgres replica, anything with local state or a slow warm-up (JVM JIT warm caches, connection pools) gets killed and has to rebuild state or reconnect, potentially causing a real availability blip.
- **Latency-sensitive workloads** — an eviction means a full pod reschedule cycle: image pull (if not cached), container start, readiness probe delay, traffic re-routing. During that window, capacity for that replica is zero — if you're running close to N-1 redundancy, a VPA-triggered eviction can cause a real latency spike or error rate blip, with no external trigger (no deploy, no incident) — which is exceptionally confusing to debug ("nothing changed, why did p99 spike at 3am" — answer: VPA silently evicted a pod because its own recommender's histogram crossed a threshold).
- **Anything behind a small replica count** — evicting 1 of 3 pods is a 33% capacity cut during the resize window; evicting 1 of 50 is a rounding error. Auto mode's blast radius scales inversely with replica count.

The safe, supported senior pattern: **VPA Off mode = advisory only, forever**. Treat its output as input to a human-reviewed PR that changes YAML, same as any other config change. Some clusters keep VPA in Auto mode only for genuinely stateless, high-replica-count, tolerant batch workloads where an occasional restart is a non-event — but that's an intentional per-workload decision, not a blanket default.

### Concrete before/after impact

Service: `checkout-api`, 50 replicas, originally requesting 1 CPU / 2Gi (Guaranteed-ish sizing chosen by "feels safe"). VPA reveals real p95 usage is 150m CPU / 400Mi memory.

```
Before:  50 * 1 CPU  = 50 vCPU requested   |  50 * 2Gi = 100Gi requested
After:   50 * 250m   = 12.5 vCPU requested |  50 * 600Mi = 30Gi requested   (target + ~50% headroom)

Freed capacity: 37.5 vCPU, 70Gi memory
```

On m5.2xlarge (8 vCPU/32GiB, ~7 vCPU/29GiB allocatable after reservations), that's roughly **5-6 fewer nodes** needed just to host this one service's requests — assuming Cluster Autoscaler can actually consolidate onto fewer nodes (see bin-packing section — this requires `MostAllocated` scoring or descheduling, not just freed requests). At roughly ₹12-15/hour per m5.2xlarge on-demand (varies by region/reserved pricing), 5 nodes freed is on the order of ₹4-5 lakh/month in a single region, for one service. This is the kind of change that senior engineers hunt for explicitly, because it's pure waste elimination with zero feature work.

---

## 2. HPA internals

### The actual scaling formula

The Horizontal Pod Autoscaler controller (in `kube-controller-manager` or the standalone HPA controller) computes desired replicas as:

```
desiredReplicas = ceil( currentReplicas * ( currentMetricValue / desiredMetricValue ) )
```

Worked example: HPA targets 50% average CPU utilization, currently 6 replicas, average CPU utilization currently measured at 80%:

```
desiredReplicas = ceil( 6 * (80 / 50) ) = ceil( 6 * 1.6 ) = ceil(9.6) = 10
```

Scale-down example: same HPA, utilization drops to 20%:

```
desiredReplicas = ceil( 6 * (20 / 50) ) = ceil( 6 * 0.4 ) = ceil(2.4) = 3
```

The controller polls metrics every `--horizontal-pod-autoscaler-sync-period` (default 15s), and applies a tolerance band (default 10%) so it doesn't oscillate on noise — utilization has to move outside `[desired - 10%, desired + 10%]` before any scaling action is even considered.

### Stabilization windows — why scale-down is slow and scale-up is fast by design

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api-hpa
  namespace: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 4
  maxReplicas: 40
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
```

- **`scaleDown.stabilizationWindowSeconds: 300`** (the default) — before scaling down, the controller looks back over the last 300 seconds of computed "desired replica" recommendations and takes the **maximum** value seen in that window, not the current value. So if utilization was high 2 minutes ago and just dropped, HPA still won't scale down for up to 5 more minutes. This exists specifically to prevent flapping: a traffic dip that's actually a brief lull (e.g. a batch job finishing, or a momentary dip in the middle of a diurnal peak) shouldn't trigger a scale-down immediately followed by a scale-up 60 seconds later, which would cause pod churn, cold-start latency, and connection resets.
- **`scaleUp` default stabilization is 0s** — the API server default (`behavior.scaleUp.stabilizationWindowSeconds` unset defaults to 0) means HPA reacts to load spikes as fast as the metrics pipeline allows (bounded by sync period + metrics latency, typically 15-30s), because under-provisioning during a spike costs you 5xx errors and latency SLO burn, while over-provisioning briefly just costs a little money. The asymmetry is intentional: **fail toward more capacity, not less.**

### Behavior policies in detail

Each `policies` entry says "at most this much change, over this time window":

- `type: Pods` — an absolute number of pods (e.g. `value: 4, periodSeconds: 60` = at most 4 pods added/removed per minute).
- `type: Percent` — a percentage of current replicas (e.g. `value: 10, periodSeconds: 60` = at most 10% of current replica count removed per minute — with 40 replicas, that's at most 4 pods/minute; with 5 replicas, `ceil(0.5)` = at least 1 pod, since it never rounds down to zero change when a change is warranted).
- `selectPolicy: Max` (used for scale-up above) — when multiple policies are listed, take whichever produces the *largest* change — be aggressive, reach `maxReplicas` fast if needed.
- `selectPolicy: Min` (used for scale-down above) — take whichever policy produces the *smallest* change — be conservative, ratchet down slowly.
- `selectPolicy: Disabled` — turns off scaling in that direction entirely (used to build a "scale up only, never scale down automatically" HPA for something you want to manually shed capacity from).

The config above limits scale-down to at most 10% of pods per 60 seconds — with 40 replicas running, at most 4 pods are removed per minute even if the metric says "you could go all the way down to 4 replicas immediately." This gives connection draining, load balancer deregistration, and any client-side retries time to settle between each decrement, rather than yanking 36 pods at once.

### Why HPA and VPA must not both actively drive off the same metric

If VPA runs in `Auto` mode adjusting CPU **requests** on a Deployment, and HPA scales replica **count** off `Utilization` on that same CPU resource, you get a feedback loop:

```
Utilization = actual_cpu_usage / requested_cpu   (this is literally the metric HPA reads)

1. VPA sees usage climbing, raises requests from 200m -> 400m per pod (Auto mode resizes pods).
2. Utilization = actual/requested drops sharply, purely because the DENOMINATOR doubled,
   not because load changed.
3. HPA reads the now-lower utilization and scales DOWN replica count.
4. Fewer replicas means each remaining replica now carries more real traffic,
   actual usage per pod climbs again.
5. VPA sees usage climbing again (now against a smaller, hotter fleet) and raises requests further.
6. Loop continues -- request size and replica count both oscillate, neither converges,
   and real capacity (replicas * per-pod capacity) swings wildly under constant background load.
```

Both controllers believe they are the one correctly reacting to "usage," but they're changing two different sides of the same ratio simultaneously, and neither one is aware of the other's action. This is a documented, known-bad combination — the Kubernetes autoscaling SIG explicitly calls out that VPA and HPA must not target the same metric on the same resource.

**The supported pattern:**

- **HPA drives on custom/external metrics** that are orthogonal to per-pod resource sizing — requests-per-second, queue depth (SQS/Kafka consumer lag), concurrent connections, latency percentile. These don't change denominator when VPA resizes a pod.
- **VPA runs in `Off` (recommendation only) mode** on CPU/memory, purely advisory, feeding into manual request tuning (section 1).
- If you must scale on CPU/memory and also want automatic vertical sizing, pick one axis: either HPA-on-CPU with VPA fully absent, or VPA-Auto with no HPA (fixed replica count), never both live on the same metric.

Example RPS-based HPA using a custom metric (Prometheus Adapter or KEDA):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 4
  maxReplicas: 40
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "50"
```

This coexists safely with a VPA object on the same Deployment as long as that VPA stays in `Off` mode.

---

## 3. Bin packing and node utilization

### Scheduler scoring: LeastAllocated vs MostAllocated

After filtering candidate nodes, the scheduler scores each remaining node with plugins, and `NodeResourcesFit` is the one relevant to packing density. It supports a `scoringStrategy`:

- **`LeastAllocated`** (long-time in-tree default) — scores nodes higher the *more free capacity* they have, i.e. prefers to spread pods across many lightly-loaded nodes. Good for resilience/latency headroom on a per-node basis, but bad for cost: it actively works against consolidation, keeping every node partially full so Cluster Autoscaler never finds an empty node it can safely remove.
- **`MostAllocated`** — scores nodes higher the *more already-utilized* they are (within remaining fit), i.e. prefers to pack new pods onto nodes that are already busy, leaving other nodes empty or near-empty. This is what you want if cost is the objective: nearly-empty nodes are exactly what Cluster Autoscaler's scale-down logic looks for (a node under a low-utilization threshold, typically 50% by default, with all pods evictable/reschedulable elsewhere, and no blocking PodDisruptionBudget) — CA can then safely cordon+drain+terminate it.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
            resources:
              - name: cpu
                weight: 1
              - name: memory
                weight: 1
```

Illustration of the difference with 3 nodes, each 8 vCPU allocatable, and a stream of pods each requesting 2 vCPU:

```
LeastAllocated result (spread):        MostAllocated result (pack):
Node A: [pod][pod][pod][pod]  8/8      Node A: [pod][pod][pod][pod]  8/8
Node B: [pod][pod][pod][pod]  8/8      Node B: [pod][pod]             4/8
Node C: [pod][pod][pod][pod]  8/8      Node C: (empty)                0/8

3 nodes needed, all partially loaded    2.5 nodes worth of pods, but node C
in the spread case -> CA cannot         is fully empty -> CA scales it down
scale any of them down                  to 0, cluster shrinks to 2 nodes
```

Cost-conscious platform teams running their own scheduler config (EKS/GKE/AKS managed control planes may restrict custom `KubeSchedulerConfiguration` — check whether your provider exposes this, or whether you need a self-hosted scheduler profile / secondary scheduler) generally prefer `MostAllocated` for exactly this reason: it turns "average node utilization" into a visible cost lever, whereas `LeastAllocated` naturally leaves nodes 60-70% full and unshrinkable.

Trade-off to know: `MostAllocated` reduces failure-domain independence per node (busy nodes have more blast radius if one dies) and slightly increases noisy-neighbor risk (see below), so it's not free — it's a deliberate cost/resilience trade you make consciously, usually justified because your PodDisruptionBudgets and multi-AZ replica spread already provide the resilience layer that used to be a side effect of `LeastAllocated`.

### Overcommit and QoS-driven OOM behavior

Setting `limits > requests` (Burstable QoS) is a legitimate, common cost lever: you request only what's typically needed, and let the pod burst into whatever's actually free on the node when it needs more, without paying for headroom nobody's usually using. The scheduler only reserves the request; unused capacity on the node up to the sum of limits is fair game for anyone.

CPU overcommit is relatively safe: the CFS (Completely Fair Scheduler) bandwidth controller enforces limits by *throttling* — a container that tries to use more than its CPU limit gets its scheduling quota paused for the rest of the period and resumes next period. Slower, not fatal. `container_cpu_cfs_throttled_periods_total` climbing tells you a pod is limit-bound; nobody dies.

**Memory overcommit has no equivalent soft mechanism.** Memory isn't compressible or time-shareable the way CPU is — there is no "pause and resume" for a memory allocation. If the aggregate actual memory usage of all pods on a node exceeds the node's physical memory, one of two things happens, both hard:

1. **kubelet's eviction manager** notices a node-level memory pressure condition (`memory.available` dropping below the `eviction-hard` threshold) and proactively evicts pods to relieve pressure, *before* the kernel OOM killer gets involved — but this is a coarse, whole-pod action, and picks by QoS class + usage-over-request, described below.
2. If pressure spikes faster than kubelet's eviction loop can react (a sudden allocation spike), the **kernel OOM killer** fires directly, picking a process by `oom_score_adj`.

Kubernetes sets `oom_score_adj` per pod based on QoS class specifically so that the kernel's blunt instrument kills in a sane order:

```
QoS class      oom_score_adj              Killed
Guaranteed     -997                       last  (requests == limits on every resource)
Burstable      2 - 999, scaled by         middle (killed once using more than its
               (usage/request) ratio                own memory REQUEST, roughly proportional
                                                     to how far over-request it is)
BestEffort     1000                       first (no requests set at all)
```

The practical failure scenario: a node runs 3 Burstable pods, each requesting 500Mi/limiting 2Gi. Under load, pod A balloons to 1.8Gi actual usage (within its own limit, technically "fine" from that pod's perspective), pushing total node memory usage past capacity. The kernel doesn't necessarily kill pod A — it kills whichever process has the *worst* combination of oom_score_adj and absolute memory usage, which is a function of how far over *its own request* each pod is, not which pod "caused" the pressure. **A well-behaved neighbor sitting near its request can get killed because a different pod on the same node blew past its request.** This is the single most common "why did my healthy-looking pod just restart with no application error" ticket, and the diagnosis is always: check `kubectl describe node` for `MemoryPressure` conditions and `kubectl get events` for `Evicted`/`OOMKilled` on that node around the timestamp, then check every pod's usage-vs-request on that node, not just the one that died.

Mitigation: keep Burstable limits within a sane multiple of requests (2x, not 10x) so worst-case aggregate burst on a node is bounded and predictable; reserve `Guaranteed` QoS (requests == limits) for anything where an unrelated neighbor's memory spike killing you is unacceptable (payment processing, anything stateful without fast failover).

---

## 4. Multi-tenancy models

### Soft multi-tenancy: namespace + ResourceQuota + LimitRange + NetworkPolicy

This is "one cluster, logically separated," and is the default posture for internal platform teams serving multiple product teams that trust each other reasonably but still want blast-radius containment.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-checkout
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 80Gi
    limits.cpu: "80"
    limits.memory: 160Gi
    pods: "100"
    persistentvolumeclaims: "20"
    services.loadbalancers: "2"
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-defaults
  namespace: team-checkout
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 10m
        memory: 16Mi
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-checkout
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace-and-dns
  namespace: team-checkout
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from: [{ podSelector: {} }]
  egress:
    - to: [{ podSelector: {} }]
    - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } } }]
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
```

`ResourceQuota` caps the aggregate requests/limits/object counts a namespace can consume — it's a hard API-server-enforced ceiling (pod creation is rejected with a 403 if it would breach quota), not advisory. `LimitRange` fills in sane per-container defaults so a developer who forgets to set requests doesn't get an unbounded BestEffort pod, and clamps min/max so nobody accidentally requests 64 CPU. `NetworkPolicy` with deny-by-default plus explicit allows stops one namespace's pods from being reachable by or reaching another namespace's pods at the network layer.

**What this does not isolate:** all of this still runs on the *same kernel, same kubelet, same node*. Things that are not namespaced in Linux/Kubernetes:

- **Disk I/O contention** — cgroup v1 has weak/no I/O isolation by default (cgroup v2 with `io` controller helps but is often not fully configured); a noisy neighbor doing heavy sequential writes can starve another pod's disk-bound workload even though CPU/memory look fine for both.
- **conntrack table exhaustion** — the node's netfilter connection tracking table is a single shared resource; one namespace's pod opening huge numbers of short-lived connections (a misbehaving retry loop, a port-scanning security tool, a load test) can exhaust `nf_conntrack_max` node-wide, causing *every other pod on that node* to start silently dropping/refusing new connections with no application-level error explaining why.
- **Node-level DNS / kube-proxy overload** — CoreDNS pods and the node's iptables/IPVS rules (installed by kube-proxy) are shared infrastructure; a namespace generating a DNS query storm degrades resolution latency for every other pod on that node or even cluster-wide if CoreDNS itself saturates.
- **Kernel-level resources** generally: PID namespace limits (`kernel.pid_max`), inotify watch limits, file descriptor limits at the node/kernel level.

This is why soft multi-tenancy is appropriate for trusted internal teams, and inappropriate for hostile-tenant or strict-compliance scenarios (e.g. running arbitrary customer workloads, or workloads under regulatory data segregation requirements) — a sufficiently motivated or sufficiently buggy tenant can degrade the whole node regardless of quota/NetworkPolicy correctness.

### Hard multi-tenancy: separate clusters, or vcluster

**Separate clusters per tenant** gives real isolation: separate control plane, separate etcd, separate node pools, separate blast radius for both application bugs and platform bugs (an apiserver crash, an etcd disk full, a CNI misconfiguration only affects that tenant's cluster). Cost: N clusters means N control planes to patch, N sets of RBAC/policy to keep consistent, N times the platform-team toil for upgrades, and generally higher fixed overhead per tenant (even a "small" cluster has a floor cost for control plane + minimum node pool).

**vcluster** (and similar virtual-control-plane tools) sits in between: each tenant gets what looks like their own apiserver, their own etcd (or an etcd-compatible backing store), their own CRDs/RBAC/namespaces — but pods still get scheduled onto the *host cluster's* real nodes via a syncer that translates virtual-cluster objects into real ones in a designated host namespace. Tenants can run cluster-admin-level operations (install their own CRDs, run their own controllers, use their own Kubernetes version even) without touching the shared host control plane, while the platform team still only patches and upgrades the underlying host cluster's node pools and one real control plane.

```
Hard multi-tenancy comparison:

Separate clusters:        vcluster:                    Soft (namespace) multi-tenancy:
+------+  +------+        +----------------------+      +------------------------------+
|apiserver|apiserver|      | host apiserver       |      |     single apiserver         |
|etcd A |  |etcd B |       | +--------+ +-------+ |      |  ns-A   ns-B   ns-C  ...      |
|nodes A|  |nodes B|       | |vc-api A| |vc-api B| |      |  (shared nodes, shared kubelet,|
+------+  +------+        | +--------+ +-------+ |      |   shared kernel, shared CNI)  |
                          |     (shared nodes)    |      +------------------------------+
Full isolation,           Control-plane isolation,       Cheapest, lowest isolation,
highest overhead          nodes still shared             fastest to provision
```

Choose separate clusters when regulatory/compliance boundaries require physically distinct infrastructure, or when tenants need genuinely different Kubernetes versions/feature gates. Choose vcluster when tenants need "feels like their own cluster" (their own CRDs, their own controller installs, self-service namespace-equivalent admin) without the platform team paying for N real control planes and N real node pools. Choose plain namespaces when tenants are mutually trusted internal teams and the overhead of anything more is not justified by the (low) risk.

### Concrete noisy-neighbor mitigation

- **Requests + limits on everything** — no BestEffort pods in a shared multi-tenant namespace; enforce via LimitRange default + admission policy rejecting missing requests.
- **PriorityClasses** to control preemption order under node pressure:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: tenant-critical
value: 100000
globalDefault: false
description: "Reserved for tenant-designated critical workloads; preempts lower-priority pods when the node is full."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: tenant-batch
value: 1000
globalDefault: false
description: "Batch/best-effort tenant workloads; first to be preempted."
```

  A pending `tenant-critical` pod that can't be scheduled due to a full node will trigger the scheduler's preemption logic, evicting lower-`tenant-batch`-priority pods to make room — deliberate, controlled degradation of low-priority work rather than random contention.

- **Dedicated node pools with taints/tolerations** for genuinely sensitive workloads (payment processing, anything with strict latency SLOs) so they never share a kernel with unpredictable tenants at all:

```yaml
# Node pool taint (applied at node-pool provisioning, e.g. via Karpenter/eksctl/gcloud):
# taint: dedicated=payments:NoSchedule

# Pod toleration + node affinity:
tolerations:
  - key: dedicated
    operator: Equal
    value: payments
    effect: NoSchedule
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: dedicated
              operator: In
              values: ["payments"]
```

- **Topology spread constraints** so a single node or single AZ failure (or a single node's noisy-neighbor event) doesn't take out all replicas of a service at once:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: checkout-api
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: checkout-api
```

---

## 5. Cost optimization playbook

### Spot/preemptible node pools

Appropriate for: stateless services with horizontal replica counts >= 3-4, workloads whose individual unit of work is short and idempotent (batch jobs, CI runners, queue consumers that checkpoint), anything where losing an individual instance costs at most a retried request. Inappropriate for: single-replica anything, stateful workloads without fast leader re-election, workloads with long-running non-checkpointed work (a 45-minute video encode with no resume capability).

Handling interruption correctly is the actual engineering work here, not just "add a spot node pool":

- **AWS spot interruption notice** arrives via the instance metadata endpoint roughly 2 minutes before reclamation: `curl http://169.254.169.254/latest/meta-data/spot/instance-action` returns a termination time once AWS has decided to reclaim the instance. GCP preemptible/Spot VMs give a 30-second notice via a similar metadata mechanism — the deltas matter for how much you can gracefully drain.
- **A node termination handler DaemonSet** (AWS's `aws-node-termination-handler`, or equivalent) polls that metadata endpoint from every spot node, and on receiving a notice, proactively **cordons** the node (stops new scheduling) and **drains** it (evicts pods respecting PDBs) *before* the hard reclamation happens — rather than waiting for the instance to vanish and letting Kubernetes discover it's gone via missed kubelet heartbeats (NodeNotReady after ~40s, pod eviction after ~5 more minutes by default) — that reactive path is much slower and messier than a proactive drain.
- **`terminationGracePeriodSeconds`** should be tuned to be comfortably inside the interruption notice window minus buffer for drain scheduling overhead — e.g. if you get 120s notice, a grace period of 30-60s gives the app time to finish in-flight requests and close connections cleanly, while leaving margin for the termination handler's own reaction latency.
- **PodDisruptionBudgets** matter enormously on spot, because spot interruptions are correlated (a spot price/capacity event can reclaim many instances of the same type in the same AZ near-simultaneously) — without a PDB, a proactive drain (or several drains close together) can take a service below its minimum safe replica count all at once:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-api-pdb
spec:
  minAvailable: 75%
  selector:
    matchLabels:
      app: checkout-api
```

### Cluster Autoscaler vs Karpenter

**Cluster Autoscaler (CA)** works against pre-defined node groups (ASGs on AWS, MIGs on GCP, VMSS on Azure) — you decide ahead of time "I have a t3.large group, an m5.xlarge group, a spot-diversified group," each with min/max size, and CA's job is to pick *which existing group* to scale up when pods are unschedulable, and to identify underutilized nodes to scale down. It's mature, widely deployed, well understood, and works fine when your instance-type diversity needs are modest and stable.

**Karpenter** provisions nodes directly against a flexible constraint set (instance families, sizes, architectures, purchase options) with no pre-defined node groups at all — you give it a `NodePool`/`EC2NodeClass` describing *allowed* instance types/sizes/capacity types, and for each batch of pending unschedulable pods, it computes the actual bin-packing-optimal instance type and size to launch, then calls the cloud API directly to create exactly that instance.

Practical differences that matter:

- **Reaction speed** — CA has to go through an ASG's own scaling API and wait for the ASG to launch an instance of whatever type that group is pinned to; Karpenter calls the EC2 RunInstances-equivalent API directly with the specific instance type it already decided on, cutting out a layer of indirection and typically scaling out faster, especially noticeable under bursty, latency-sensitive scale-out.
- **Bin-packing quality** — CA picks from your pre-defined groups, so if your pending pods need e.g. 3.5 vCPU/6Gi and your only groups are sized for 4 vCPU/16Gi and 8 vCPU/32Gi, you get whichever group fits, with the leftover capacity wasted. Karpenter evaluates the actual pending pod resource shapes across its allowed instance type list and picks the tightest-fitting real instance type, batch by batch, so heterogeneous workloads get much closer to ideal packing without you hand-maintaining a matrix of node groups for every workload shape.
- **Operational simplicity vs flexibility trade** — CA's mental model (fixed groups) is simpler to reason about, audit, and constrain (e.g. compliance requiring only specific approved instance types is trivial — that's just the ASG's launch template). Karpenter's flexibility means broader instance-type exposure by default, which needs explicit constraint in `NodePool` requirements if you have compliance/instance-type restrictions, and it's a newer, faster-moving project with a different mental model for on-call staff to learn.

Migrate to Karpenter when: instance-type/size diversity of your actual workloads is high (mixed CPU/memory-bound services), scale-out latency under burst genuinely matters (customer-facing traffic spikes), or CA's node-group sprawl (many ASGs to cover different shapes) has become its own maintenance burden. Stick with CA when: workload shapes are homogeneous, your team already has CA operational muscle memory and no urgent driver to change, or strict, simple, auditable instance-type control via existing ASG tooling is a hard requirement.

### Rightsizing, idle cleanup, and cost attribution

Rightsizing is section 1's VPA workflow, applied on a schedule (quarterly is a reasonable cadence) rather than once.

**Finding idle workloads**: query ingress/egress traffic metrics or request-rate metrics per service and flag anything at effectively zero for an extended window —

```promql
sum by (namespace, service) (
  rate(istio_requests_total[7d])
) == 0
```

or, without a service mesh, using kube-state-metrics joined with actual pod network activity, or simply CPU usage near request floor for weeks continuously, which is a cheap proxy for "nobody's really using this." Cross-reference against a labeling convention (e.g. `env=staging`, `owner=team-x`) so cleanup candidates can be routed to the right team for a decommission decision rather than deleted unilaterally.

**Cost attribution** requires namespaces/workloads to carry consistent, enforced labels — `team=`, `cost-center=`, `env=` — because without them, joining Kubernetes resource consumption to cloud billing is guesswork. `kube-state-metrics` exposes object labels as Prometheus label dimensions (`kube_namespace_labels{label_team="checkout"}`), which tools like **OpenCost** or **Kubecost** join against actual node cost (on-demand/spot/reserved pricing per instance type, per cloud) and per-pod resource consumption to produce a real ₹/month figure per namespace, per team, per cost-center — not an estimate, an actual allocation of the cloud bill.

The label convention only works if it's *enforced*, not just documented — otherwise cost reports have a growing "unattributed" bucket as teams forget or new namespaces skip the convention. Enforce via admission policy so a namespace or workload simply cannot be created without the required labels:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cost-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-and-cost-center
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        message: "Namespace must carry 'team' and 'cost-center' labels for cost attribution."
        pattern:
          metadata:
            labels:
              team: "?*"
              cost-center: "?*"
```

(Equivalent enforcement is expressible in OPA Gatekeeper via a `ConstraintTemplate` + `K8sRequiredLabels` constraint — the point is the same: reject the object at admission time if the label is missing, rather than relying on convention and finding out during the monthly cost review that 30% of the bill is unattributed.)

---

## 6. Capacity planning

### Forecasting from historical trends

`predict_linear()` fits a linear regression over a time range and projects forward, which is exactly the right tool for "given the current trend, when do I run out." Predicting memory exhaustion within the next 4 hours from the last 6 hours of trend:

```promql
predict_linear(node_memory_MemAvailable_bytes{instance="ip-10-0-4-21"}[6h], 4 * 3600) < 0
```

Read literally: fit a line to `node_memory_MemAvailable_bytes` over the trailing 6 hours, extrapolate that line 4 hours (`4 * 3600` seconds) into the future, and alert if the projected value is below zero — i.e. "at the current rate of decline, this node's available memory hits zero within 4 hours." Wire this as an alert, not just a dashboard panel:

```yaml
- alert: NodeMemoryExhaustionPredicted
  expr: predict_linear(node_memory_MemAvailable_bytes[6h], 4 * 3600) < 0
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Node {{ $labels.instance }} predicted to exhaust memory within 4h at current trend"
```

The same pattern applies directly to PVC fill rate — critical for anything with local disk pressure (Prometheus's own TSDB volume, a Kafka broker's data volume, a database's data directory):

```promql
predict_linear(kubelet_volume_stats_available_bytes{persistentvolumeclaim="prometheus-data-0"}[6h], 24 * 3600) < 0
```

This predicts, from the last 6 hours of consumption trend, whether that PVC will hit zero available bytes within the next 24 hours — giving you a full day's lead time to expand the volume (if the storage class supports online resize) or clean up retained data, instead of discovering it when the write path starts failing.

Use a longer lookback window (`[3d]` or `[7d]`) for slower, noisier trends like general node-pool CPU/memory growth as the platform grows organically, and a shorter one (`[1h]`-`[6h]`) for acute, fast-moving situations like a memory leak or a runaway log volume — the lookback window is effectively how much you're smoothing out noise versus how quickly you want to react to a recent regime change.

### Headroom conventions

**Cluster-wide headroom**: a common convention is keeping 20-30% of total cluster capacity unallocated at steady state, sized specifically to absorb the loss of one AZ's worth of nodes without immediately breaching capacity. If a region has 3 AZs and you run N nodes evenly spread, losing one AZ removes ~33% of capacity instantly — 20-30% headroom means the *remaining* two AZs' nodes, plus Cluster Autoscaler reacting within its own latency window (typically 1-5 minutes for a scale-up to actually land ready-to-serve pods), can absorb the rescheduled load from the lost AZ without cascading into wider capacity failure while CA catches up. This is a business continuity number, not a performance number — it should be sized against "how long can we tolerate degraded capacity while CA/Karpenter catches up," not against average utilization.

**Per-node headroom**: every node reserves capacity below the label "allocatable" for the kubelet and OS itself, via `--system-reserved` and `--kube-reserved` kubelet flags, plus an `--eviction-hard` threshold that keeps a further buffer to react to memory pressure before the node becomes genuinely unresponsive:

```yaml
# kubelet config (or equivalent flags)
systemReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "2Gi"
kubeReserved:
  cpu: "250m"
  memory: "512Mi"
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
```

`allocatable = node capacity - system-reserved - kube-reserved - eviction-hard threshold`. On an 8 vCPU/32GiB node with the values above, allocatable memory for pods is roughly `32Gi - 1Gi - 512Mi - 500Mi ≈ 30Gi`, not the full 32Gi — this is why `kubectl describe node` shows a smaller `Allocatable` than `Capacity`, and why capacity planning math must always use `Allocatable`, never raw instance-type specs, or you'll be consistently off by several percent per node across a large fleet.

### Top-down vs bottom-up quota planning

**Top-down**: start from an approved total infrastructure budget (say, cluster-wide spend target of ₹40 lakh/month), convert to a total vCPU/memory budget at your blended on-demand+spot+reserved rate, then divide that budget into per-team `ResourceQuota` allocations by business priority (payments team gets 30% of the pool, recommendations gets 15%, and so on). Fast to set up, forces explicit prioritization conversations up front, but is disconnected from what workloads *actually* need — quotas end up arbitrary unless revisited against real usage.

**Bottom-up**: sum VPA-informed, rightsized per-workload requests (section 1's actual measured Target values) across every workload in a namespace, aggregate across namespaces to derive the real cluster/node-pool size needed, and set quotas to match measured reality plus a growth margin. Accurate and grounded in real usage, but only as good as your VPA data recency, and doesn't on its own answer "which team should get less if the total doesn't fit the budget" — that's still a business decision, not a math one.

**Recommended**: run both, reconciled on a regular cadence (quarterly is reasonable) — bottom-up VPA-informed sums tell you what things actually cost today and where the organic growth trend is heading; top-down budget tells you the ceiling leadership will fund. Where bottom-up demand exceeds top-down budget, that's the trigger for a real prioritization conversation (defer a project, rightsize harder, negotiate more budget) rather than either number silently winning by default. Where bottom-up demand is comfortably under budget, that headroom is exactly the buffer described above for AZ-loss tolerance and burst absorption — don't let it get silently eaten by scope creep across a dozen individually-reasonable-looking `ResourceQuota` bumps.
