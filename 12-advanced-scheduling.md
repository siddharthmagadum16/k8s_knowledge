# Advanced Scheduling

## 1. Affinity and Anti-Affinity

The scheduler supports two independent axes: **node affinity** (which nodes a pod can land on, based on node labels) and **pod affinity/anti-affinity** (which nodes a pod can land on, based on labels of *other pods already running there*). Both come in `required` and `preferred` flavors, and both flavors carry the same suffix that trips people up:

```
requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution
```

`IgnoredDuringExecution` means the rule is evaluated only at scheduling time. Once a pod is bound to a node, Kubernetes never re-checks the rule. If you relabel a node, or relabel the pods that a `podAffinity` rule was matching against, running pods are **not evicted or rescheduled**. The affinity rule only affects the next scheduling decision for a new pod. This is the single most common misconception junior engineers have — they expect affinity to be a live constraint enforced continuously, like a network policy. It's not. It's a placement decision made once, at bind time.

There is a `RequiredDuringSchedulingRequiredDuringExecution` variant defined in the API types for future use, but as of current stable Kubernetes it is not implemented by the scheduler — don't rely on it.

### Node affinity

Node affinity replaces `nodeSelector` with a richer expression language (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: analytics-worker
  template:
    metadata:
      labels:
        app: analytics-worker
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["m6i.2xlarge", "m6i.4xlarge"]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: node-lifecycle
                operator: In
                values: ["spot"]
          - weight: 20
            preference:
              matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["ap-south-1a"]
      containers:
      - name: worker
        image: registry.internal/analytics-worker:1.4.2
```

`required` terms are OR'd across `nodeSelectorTerms` and AND'd within `matchExpressions` inside a single term. If the required block matches zero nodes, the pod stays `Pending` forever (until you fix labels or the block).

For `preferred`, each matching term contributes its `weight` (1-100) to a per-node sum. The scheduler adds up weights from every preferred rule that matches a given node, folds that into the overall priority score alongside other scoring plugins (`PodTopologySpread`, `InterPodAffinity`, `NodeResourcesBalancedAllocation`, etc.), and picks the highest-scoring node among those that passed the filter phase. A `preferred` rule is never a hard requirement — if no node matches any preferred term, the pod still schedules, just without preference applied.

### Pod affinity / anti-affinity and topologyKey

`topologyKey` is what makes pod affinity fundamentally different from node affinity: it defines the **domain of "togetherness"**. The scheduler looks at the topology key's value on the candidate node, then checks whether any node sharing that same value also hosts a pod matching your `labelSelector`.

- `topologyKey: kubernetes.io/hostname` → togetherness domain is a single node.
- `topologyKey: topology.kubernetes.io/zone` → togetherness domain is an entire AZ (any node with the same zone label counts as "together").

**Spread Deployment replicas across AZs (anti-affinity):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  replicas: 6
  selector:
    matchLabels:
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: checkout-api
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: checkout-api
        image: registry.internal/checkout-api:2.9.0
        resources:
          requests: {cpu: "500m", memory: "512Mi"}
          limits: {cpu: "1", memory: "1Gi"}
```

Caution with `required` anti-affinity plus replica count: if you have 3 zones and ask for 6 replicas with a hard "one per zone" rule via `topologyKey: kubernetes.io/hostname` (not zone) you'd fail after N nodes are full. With zone-level anti-affinity and only 3 zones, requiring strict "no two pods share a zone" caps you at 3 replicas — the other 3 will sit `Pending`. In production, prefer `preferred` with high weight, or use **topology spread constraints** (section 3) instead of `required` pod anti-affinity for this exact scenario — it expresses "spread evenly," not "never share," and handles replica counts > zone counts gracefully.

**Colocate a cache pod with an app pod on the same node (affinity):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sidecache
  template:
    metadata:
      labels:
        app: sidecache
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["checkout-api"]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: sidecache
        image: registry.internal/sidecache:0.8.1
```

This is a poor-man's sidecar pattern for teams not using native sidecar containers — it guarantees the cache pod lands on a node that already has (or will have, order matters — see below) a `checkout-api` pod. Note the scheduling-order problem: if `sidecache` is scheduled before any `checkout-api` pod exists on any node, the required rule matches zero nodes and it hangs `Pending`. Pod affinity has no concept of "wait for the other pod" — this is a real footgun when both Deployments are applied simultaneously via GitOps. Mitigate with `preferred` instead of `required`, or ensure creation ordering, or use a DaemonSet if you actually want "one per node" semantics (which handles the ordering problem naturally).

`topologyKey` for pod affinity/anti-affinity, unlike node affinity, has one more restriction: the **kube-scheduler and API server enforce that `requiredDuringScheduling` pod anti-affinity across namespaces requires the `CrossNamespacePodAffinity` behavior to be explicitly allowed** via a quota (`namespaceSelector` + a `ResourceQuota` scoped to `CrossNamespacePodAffinity`) in modern clusters, to stop one tenant from affecting scheduling of pods in another tenant's namespace. If you see pod affinity silently not matching pods in other namespaces, check `namespaceSelector` and this quota first.

---

## 2. Taints and Tolerations

Taints go on nodes; tolerations go on pods. A taint says "don't put things here unless they tolerate this." A toleration says "I can live with this taint" — it does **not** attract the pod to the tainted node, it only removes a repulsion. This is the second most common junior mistake: adding a toleration to a pod and expecting it to preferentially schedule onto the tainted (e.g. GPU) nodes. It won't — it becomes merely *eligible* to land there, same as every other untainted node in the cluster, unless you also add a `nodeSelector` or `nodeAffinity` pinning it there.

### Effects

| Effect | New pods without matching toleration | Pods already running without matching toleration |
|---|---|---|
| `NoSchedule` | Blocked | Untouched, keep running |
| `PreferNoSchedule` | Scheduler tries to avoid, but will place if no better option (soft) | Untouched |
| `NoExecute` | Blocked | **Evicted** (immediately, unless `tolerationSeconds` set) |

```yaml
# Taint a node
kubectl taint nodes gpu-node-01 dedicated=ml-training:NoSchedule
kubectl taint nodes gpu-node-01 dedicated=ml-training:NoExecute
```

`NoExecute` is the only effect that acts on pods already scheduled. It's used for two real scenarios:

1. **Deliberate workload isolation** — you taint a node pool and only pods with the matching toleration may even survive there.
2. **Automatic node-health handling** — the node controller taints nodes `NoExecute` when conditions like `node.kubernetes.io/not-ready` or `node.kubernetes.io/unreachable` appear (node stops sending heartbeats). Every pod in the cluster tolerates these two taints **by default** for 300 seconds — this is a toleration the API server injects automatically:

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

`tolerationSeconds` is the mechanism behind "give a flaky node 5 minutes before we start evicting its pods" — it's what stops a single missed heartbeat or a brief network blip from triggering a mass pod eviction storm. If you set `tolerationSeconds: 0` (or omit it entirely and only have `NoExecute` without duration) the pod is evicted the instant the taint is applied.

You can lower this default for latency-sensitive workloads that would rather fail fast and get rescheduled elsewhere, or raise it for stateful workloads where a restart is expensive:

```yaml
spec:
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30   # fail over fast — stateless, latency-sensitive
```

### Dedicated node pool pattern (correct version)

To get GPU workloads reliably onto GPU nodes and *keep everything else off*, you need both halves:

```bash
kubectl taint nodes gpu-node-01 nvidia.com/gpu=present:NoSchedule
kubectl label nodes gpu-node-01 node-type=gpu
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training-job
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training-job
  template:
    metadata:
      labels:
        app: ml-training-job
    spec:
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "present"
        effect: "NoSchedule"
      nodeSelector:
        node-type: gpu
      containers:
      - name: trainer
        image: registry.internal/ml-trainer:3.1.0
        resources:
          limits:
            nvidia.com/gpu: 1
```

Toleration permits scheduling onto the tainted node. `nodeSelector` (or `nodeAffinity`) is what actually attracts the pod there instead of any of the other untainted nodes in the cluster. Skipping the selector means your GPU pods might land on ordinary CPU nodes just fine (since they tolerate nothing special is blocking them) and never touch the GPU nodes at all unless the scheduler happens to pick one — leaving expensive GPU capacity idle while cheaper nodes fill up, or vice versa, essentially random from the operator's perspective.

Built-in taints worth knowing: `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable`, `node.kubernetes.io/out-of-disk`, `node.kubernetes.io/memory-pressure`, `node.kubernetes.io/disk-pressure`, `node.kubernetes.io/pid-pressure`, `node.kubernetes.io/network-unavailable`, `node.kubernetes.io/unschedulable` (set by `kubectl cordon`), and cloud-controller-managed `node.cloudprovider.kubernetes.io/uninitialized`.

---

## 3. Topology Spread Constraints

Pod (anti-)affinity answers "must/should this pod avoid or colocate with specific other pods." Topology spread constraints answer a different question: "keep the *count* of matching pods balanced across a topology domain, within a tolerance." This is usually the better primitive for HA spreading because it understands replica counts natively, instead of the binary "shares this domain or not" logic of anti-affinity.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
spec:
  replicas: 9
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: payments-api
        minDomains: 3
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: payments-api
      containers:
      - name: payments-api
        image: registry.internal/payments-api:4.2.1
        resources:
          requests: {cpu: "250m", memory: "256Mi"}
```

Field semantics:

- **`maxSkew`**: the maximum allowed difference between the pod count in the domain with the *most* matching pods and the domain with the *least*. `maxSkew: 1` with 9 replicas across 3 zones means each zone should end up with 3, and the scheduler will refuse (or deprioritize) placements that push any zone to 5 while another has 3 (skew of 2).
- **`topologyKey`**: the node label defining domains, same semantics as pod affinity's topologyKey.
- **`whenUnsatisfiable`**:
  - `DoNotSchedule` (hard) — pod stays `Pending` if placing it anywhere would violate `maxSkew`.
  - `ScheduleAnyway` (soft) — scheduler still prefers balanced placement via scoring, but will place the pod even if it worsens skew, rather than leave it `Pending`.
- **`labelSelector`**: which pods count toward the skew calculation. Omitting it (pre-1.25 behavior on some fields) or getting it wrong is a common bug — if the selector doesn't match your own pod's labels, the constraint effectively counts zero pods everywhere and does nothing useful.
- **`minDomains`**: the minimum number of topology domains you expect to exist. If the cluster only has 2 zones but `minDomains: 3`, the scheduler treats the missing third domain as having zero pods for skew math under `DoNotSchedule`, effectively forcing it to leave room / fail rather than silently over-pack the 2 real zones. This matters for clusters that autoscale node pools per zone — without `minDomains`, a zone with no nodes yet is just invisible to the calculation, not treated as "empty and needing pods."

### Combining with podAntiAffinity — precedence

You can set both `topologySpreadConstraints` and `podAntiAffinity` on the same pod spec. They are evaluated independently in the Filter phase — a candidate node must pass **all** filters (both the anti-affinity predicate and the topology spread predicate) to remain eligible; there's no "override" between them. A common production pattern is:

```
podAntiAffinity (preferred, hostname)        -> soft nudge away from exact same node
topologySpreadConstraints (required, zone)   -> hard guarantee of even zone distribution
topologySpreadConstraints (soft, hostname)   -> best-effort spread across hosts within a zone
```

Layering a hard zone-level topology spread with a soft hostname-level anti-affinity/spread gives you a guaranteed AZ-failure blast radius limit, while still trying for host-level diversity without risking `Pending` pods when a zone has fewer nodes than replicas. Don't stack two `required`/`DoNotSchedule` hard constraints on the same topology key from both mechanisms — you gain nothing and double your chance of an unsatisfiable combination leaving pods stuck.

---

## 4. Pod Priority and Preemption

### PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 100000
globalDefault: false
description: "Customer-facing production workloads"
preemptionPolicy: PreemptLowerPriority
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 1000
globalDefault: true
description: "Default class for anything not explicitly set — batch/best-effort"
preemptionPolicy: PreemptLowerPriority
```

`globalDefault: true` on exactly one PriorityClass applies that value to every pod that doesn't specify `priorityClassName` — worth auditing, because a cluster with no explicit default silently assigns priority `0` to everything, meaning nothing can ever preempt anything else and a single misbehaving high-resource-request Job can starve the whole node pool with no relief.

Kubernetes ships two reserved system classes you should never repurpose:

- `system-cluster-critical` (value ~2000000000) — cluster-critical addons (CoreDNS, etc.)
- `system-node-critical` (value ~2000001000, highest) — node-local critical daemons (kube-proxy, CNI plugins)

Attach priority to your workload:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  template:
    spec:
      priorityClassName: production-critical
      containers:
      - name: checkout-api
        image: registry.internal/checkout-api:2.9.0
```

### How preemption victim selection actually works

When a pod `P` with priority `X` can't be scheduled because the cluster is full, the scheduler runs a separate preemption cycle:

1. For each node, simulate removing pods with priority *lower than X* (never removes equal-or-higher-priority pods, and never removes pods that are themselves already in a preemption-caused terminating state elsewhere) until `P` would fit if bound there.
2. Among nodes where this is possible, the scheduler picks the node requiring **removal of the fewest pods**, and among ties, prefers **removing lower-priority pods over higher-priority ones**, and prefers not violating PodDisruptionBudgets **if an alternative that doesn't violate any PDB exists**.
3. Crucially: **PDBs are a soft signal here, not a hard stop.** If every viable eviction option violates some PDB, the scheduler will go ahead and violate one anyway rather than let the high-priority pod stay `Pending` indefinitely. Preemption is allowed to push a Deployment below its PDB's `minAvailable`.
4. Victim pods are deleted with their normal `terminationGracePeriodSeconds` (graceful shutdown still applies) — this is not a hard kill, just a scheduling-triggered eviction.
5. There is no guarantee `P` actually lands on the exact node whose pods were evicted by the time it's rescheduled — nominal node info (`.status.nominatedNodeName`) is a hint, not a binding contract; another pod can race in.

### The real production gotcha

A batch of newly-deployed pods carrying a high `priorityClassName` (often by accident — a copy-pasted manifest, or a Helm chart default that got bumped) arrives faster than the cluster autoscaler can add capacity. The scheduler starts preempting lower-priority pods to make room *immediately*, because preemption is far faster than waiting for a new node to boot (which can take 1-3 minutes on cloud providers). The result: production workloads at a lower (or unset/default-zero) priority start getting evicted to make room for what's often a batch job or a test deployment that happened to inherit a high-priority class, while the cluster autoscaler is still catching up.

Mitigations:
- Never let unrelated teams/namespaces share a broad, high-value `PriorityClass` — scope priority classes per team/tier via admission policy (e.g. `ValidatingAdmissionPolicy` or Kyverno) so namespaces are restricted to an allow-listed set of PriorityClass names.
- Keep a meaningful gap between tiers (e.g. 100000 vs 10000 vs 1000, not 100 vs 99) so you can insert intermediate tiers later without renumbering everything.
- Set `preemptionPolicy: Never` on classes that should queue rather than evict (e.g. best-effort analytics jobs that are high priority for scheduling order but should never evict something else — "run me first when there's room, but don't kick anyone out for me"):

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: analytics-priority-no-preempt
value: 50000
preemptionPolicy: Never
```

- Alert on preemption events: `kubectl get events --field-selector reason=Preempted -A`.

---

## 5. Custom Schedulers and Scheduler Extenders

The default scheduler (`default-scheduler`) is just a controller watching unscheduled pods and writing bindings. Nothing stops you from running additional scheduler binaries, or configuring one kube-scheduler process to run multiple named profiles.

### Running a second scheduler by name

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-batch-job
spec:
  schedulerName: volcano
  containers:
  - name: worker
    image: registry.internal/batch-worker:1.0.0
```

Any pod with `schedulerName` set to something other than `default-scheduler` is completely ignored by the default scheduler — it sits `Pending` until a controller/process watching that scheduler name picks it up. If you typo the name or the named scheduler isn't actually deployed, the pod hangs `Pending` forever with no useful event beyond the absence of `FailedScheduling` — because no scheduler is even looking at it. This is a common silent failure: check `schedulerName` early when a pod is inexplicably `Pending` with zero scheduling events at all.

This is the mechanism behind batch/ML schedulers like **Volcano** and **Kueue** (gang scheduling — all-or-nothing placement of a set of pods belonging to one job, which the default scheduler has no native concept of) and Kubeflow's `kube-batch`. They deploy as a separate Deployment running their own scheduling loop against the same API server, and pods opt in via `schedulerName`.

### Scheduler extenders vs scheduler framework plugins

**Extenders** (older, still widely used for quick integrations) are external HTTP(S) webhook services the built-in scheduler calls out to at specific points, configured via `KubeSchedulerConfiguration`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
extenders:
- urlPrefix: "https://scheduler-extender.internal:8443"
  filterVerb: "filter"
  prioritizeVerb: "prioritize"
  bindVerb: "bind"
  weight: 1
  enableHTTPS: true
  nodeCacheCapable: true
  ignorable: false
```

The scheduler POSTs the candidate node list and pod spec to `/filter` and `/prioritize` and expects JSON back (filtered node list; per-node scores). Pros: language-agnostic, no need to recompile/rebuild the scheduler binary, easy to bolt onto an existing cluster. Cons: network round-trip on every scheduling decision (latency and an availability dependency), and a much smaller surface than the full framework (no access to `Reserve`/`Permit`/`PreBind` internal state, coarser-grained).

**Scheduler framework plugins** (current recommended approach for anything beyond simple filtering) are compiled into a custom scheduler binary and hook into named extension points in the scheduling cycle:

```
Sort -> PreFilter -> Filter -> PostFilter -> PreScore -> Score -> NormalizeScore
  -> Reserve -> Permit -> WaitOnPermit -> PreBind -> Bind -> PostBind
```

- **PreFilter** — cheap upfront checks/precompute before per-node filtering (e.g. reject the pod outright if a job-level gang-scheduling minimum can never be met).
- **Filter** — the per-node pass/fail predicate (equivalent to what extenders call `filter`).
- **Score** — per-node numeric scoring (equivalent to `prioritize`).
- **Reserve** — reserve resources on the chosen node before bind, with an `Unreserve` cleanup path if a later stage fails — this is where you'd hold a "slot" for gang scheduling.
- **Permit** — can block, deny, or *delay* binding (used for gang scheduling: hold all pods in a job at Permit until every pod in the group has a candidate node, then release them together).
- **PreBind / Bind / PostBind** — the actual binding call and any pre/post hooks (e.g. attach a volume before bind).

You write these as Go plugins compiled into the `kube-scheduler` binary using `k8s.io/kubernetes/pkg/scheduler/framework`, register them via `KubeSchedulerConfiguration`, and typically ship your own scheduler image (e.g. Kueue and Volcano do exactly this).

### Multiple profiles, one scheduler process

A single `kube-scheduler` process can run several named profiles simultaneously, each with its own plugin configuration, avoiding the need to run entirely separate scheduler deployments for moderate customization:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
- schedulerName: batch-scheduler
  plugins:
    score:
      disabled:
      - name: NodeResourcesBalancedAllocation
      enabled:
      - name: NodeResourcesFit
        weight: 5
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
```

Here `batch-scheduler` scores nodes with `MostAllocated` (pack batch jobs tightly to free up whole nodes for cluster-autoscaler scale-down) while `default-scheduler` keeps the standard `LeastAllocated`-leaning balance for regular workloads. Pods opt into the second profile purely via `schedulerName: batch-scheduler` — no separate binary or Deployment required.

---

## 6. Descheduler

The scheduler is a one-shot placement engine: it makes a bind decision when a pod is created (or recreated after deletion), and never revisits that decision. There is no background loop continuously asking "is this pod still optimally placed?" Consequences that surprise people:

- Scale a node pool down then back up (e.g. nightly cluster-autoscaler cycling, or a spot-instance reclaim/replace) — pods that come back up land wherever there's room *right now*, not necessarily spread the way your topology spread constraints intended, because those constraints were satisfied against the cluster state at the moment each pod was (re)scheduled, not against some ongoing global optimum.
- Change a `nodeAffinity` or add a new anti-affinity rule to a Deployment — existing pods created under the old rule keep running wherever they already are (recall `IgnoredDuringExecution` from section 1). Only pods created after the change get the new placement logic.
- A long-lived cluster accumulates skew over months: some nodes end up hosting 8 pods from one Deployment while others host 1, simply from the accretion of individual scheduling decisions each of which was locally valid at the time.

The **descheduler** (`kubernetes-sigs/descheduler`) fixes this by running as a periodic job that finds pods violating current policy and evicts them (via the eviction API, respecting PDBs) so the scheduler places them again — this time against current cluster state and current affinity/topology rules.

### Strategies (as a `DeschedulerPolicy`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: descheduler-policy
  namespace: kube-system
data:
  policy.yaml: |
    apiVersion: descheduler/v1alpha2
    kind: DeschedulerPolicy
    profiles:
    - name: default
      pluginConfig:
      - name: RemoveDuplicates
      - name: LowNodeUtilization
        args:
          thresholds:
            cpu: 20
            memory: 20
            pods: 20
          targetThresholds:
            cpu: 50
            memory: 50
            pods: 50
      - name: RemovePodsViolatingTopologySpreadConstraint
        args:
          constraints:
          - DoNotSchedule
      - name: RemovePodsViolatingNodeAffinity
        args:
          nodeAffinityType:
          - requiredDuringSchedulingIgnoredDuringExecution
      plugins:
        balance:
          enabled:
          - RemoveDuplicates
          - LowNodeUtilization
          - RemovePodsViolatingTopologySpreadConstraint
        deschedule:
          enabled:
          - RemovePodsViolatingNodeAffinity
```

- **RemoveDuplicates** — evicts extra pods of the same ReplicaSet/Job/StatefulSet stacked on the same node, so the replacement can spread onto an under-used node.
- **LowNodeUtilization** — moves pods off nodes below `thresholds` (considered "underutilized," candidates to drain toward) and off nodes above `targetThresholds` (considered "overutilized," candidates to relieve) — a bin-packing rebalance.
- **RemovePodsViolatingTopologySpreadConstraint** — re-checks live topology spread constraints against current pod placement and evicts pods causing skew beyond `maxSkew`, letting the scheduler re-place them correctly.
- **RemovePodsViolatingNodeAffinity** — catches pods whose node no longer satisfies a `required` node affinity rule that was changed *after* the pod was scheduled (the exact "affinity changed but pod wasn't evicted" gap from section 1) and evicts them so they land somewhere compliant.

### Running as a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: descheduler
          containers:
          - name: descheduler
            image: registry.k8s.io/descheduler/descheduler:v0.30.0
            args:
            - "--policy-config-file=/policy/policy.yaml"
            - "--v=3"
            volumeMounts:
            - name: policy
              mountPath: /policy
          restartPolicy: Never
          volumes:
          - name: policy
            configMap:
              name: descheduler-policy
```

### Risk of churn

Every eviction the descheduler performs costs you a pod restart, a fresh readiness-probe warmup, and (for stateful workloads) volume detach/reattach or cache-loss. Misconfigured thresholds (e.g. `LowNodeUtilization` targets set too close to each other, or a schedule that runs every 5 minutes instead of every few hours) create a fight between the descheduler evicting pods and the scheduler placing them right back in a similar imbalance, burning churn without improving anything — effectively a self-inflicted rolling restart of your cluster on a cron. Run it with `--dry-run` first, start with a long interval (hours, not minutes), keep thresholds with meaningful gaps, and monitor eviction counts before trusting it against production namespaces. Scope it with `namespaceSelector wildcard exclude` to skip anything stateful or genuinely sensitive to restarts unless you're confident in the strategy.

---

## 7. Debugging: Why Is My Pod Stuck Pending

Triage in this order — each step narrows the cause category before you touch YAML.

### Step 1: `kubectl describe pod` — read the Events section literally

```bash
kubectl describe pod checkout-api-7d9f8c6b5-x2k9p -n prod
```

The `Events` section at the bottom, with `Reason: FailedScheduling`, contains a message generated directly by the predicate that rejected each node. Read it verbatim — it names the exact failure category:

**Insufficient resources:**
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 8 Insufficient cpu, 4 Insufficient memory.
```
→ The cluster genuinely doesn't have a node with enough allocatable CPU/memory left. This needs more nodes or bigger nodes (cluster-autoscaler should be adding capacity — check if it's stuck, e.g. hit a cloud provider quota) or reducing the pod's `requests`.

**Node selector / affinity mismatch:**
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 12 node(s) didn't match Pod's node affinity/selector.
```
→ Not a capacity problem — a labeling/YAML problem. Check `kubectl get nodes --show-labels` against what your `nodeSelector`/`nodeAffinity` actually requires. Common cause: a typo'd label key/value, or a node pool that was supposed to carry a label but doesn't yet (new node pool, label applied by a bootstrap script that hasn't run).

**Untolerated taint:**
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 3 node(s) had untolerated taint {dedicated: ml-training}, 9 node(s) didn't match Pod's node affinity/selector.
```
→ Needs a toleration added to the pod spec (section 2), or you're intentionally being kept off those nodes and the pod belongs on a different pool entirely.

**Pod affinity/anti-affinity violation:**
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 6 node(s) didn't satisfy existing pods anti-affinity rules, 6 node(s) didn't satisfy anti-affinity/selector.
```
→ Not a resource or taint issue — the topology math doesn't work out (e.g. required zone anti-affinity with more replicas than zones, section 1's caution). Needs a YAML fix: loosen to `preferred`, or switch to topology spread constraints.

**Exceeded max pods per node:**
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 12 node(s) exceed max pods per node limit.
```
→ Hitting the kubelet's `--max-pods` (often 110 by default, or lower on constrained instance types where it's derived from ENI/IP limits on cloud providers). This is a pod-density ceiling independent of CPU/memory — you can have "spare" CPU and memory and still be unable to schedule because the pod-count limit per node is exhausted. Needs more nodes, not bigger ones, or raising `--max-pods` if the node's IP/ENI budget allows it.

**PVC/volume topology mismatch** (adjacent, often confused with affinity):
```
Warning  FailedScheduling  default-scheduler
0/12 nodes are available: 5 node(s) had volume node affinity conflict.
```
→ The PV is zone-pinned (common with cloud block storage — EBS/PD volumes are zone-local) and no node in that zone has room, or the pod's other constraints pushed it toward a different zone than the volume lives in. Needs alignment between storage class zone and pod scheduling constraints, not a compute fix.

### Step 2: check actual node capacity vs allocation

```bash
kubectl get nodes -o wide
kubectl describe node ip-10-0-4-212.ec2.internal
```

In the node description, compare:

```
Capacity:
  cpu:                8
  memory:             32925460Ki
  pods:               110
Allocatable:
  cpu:                7910m
  memory:             30219092Ki
  pods:               110
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                7500m (94%)   9800m (123%)
  memory             28Gi (95%)    31Gi (105%)
```

`Allocatable` is always slightly below `Capacity` (kubelet reserves resources for itself and the OS via `--system-reserved`/`--kube-reserved`). `Allocated resources` sums pod **requests**, which is what the scheduler actually uses for the Fit check — not current live usage. A node can show 95% CPU **requested** while live utilization (from `kubectl top node`) sits at 20%, because requests are reservations, not real-time measurements. Don't diagnose a `FailedScheduling`/insufficient-cpu event by looking at `kubectl top node` — it's the requested/allocatable numbers here that gate scheduling, not actual usage.

Quick fleet-wide scan for tight nodes:

```bash
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name) \(.status.allocatable.cpu) \(.status.allocatable.memory)"'
kubectl describe nodes | grep -A5 "Allocated resources"
```

### Step 3: scheduler logs

If the events don't fully explain it (e.g. the pod isn't even generating `FailedScheduling` events — meaning nothing is trying to schedule it at all, per the `schedulerName` typo trap in section 5), go to the scheduler itself.

On managed control planes / static-pod based scheduler:

```bash
kubectl logs -n kube-system -l component=kube-scheduler --tail=200
kubectl logs -n kube-system kube-scheduler-<control-plane-node> --tail=200
```

On a self-managed cluster where kube-scheduler runs as a systemd unit rather than a static pod:

```bash
journalctl -u kube-scheduler -n 200 --no-pager
```

Raise verbosity temporarily if the default log level doesn't show per-plugin filter reasoning (edit the static pod manifest or systemd unit to add `--v=4` or `--v=6`, restart, reproduce, then revert — this is chatty and shouldn't stay on in steady state).

Also confirm the scheduler itself is healthy and not crash-looping or leader-election-stuck:

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl get leases -n kube-system kube-scheduler -o yaml
```

A scheduler stuck failing leader election (common after a control-plane node replacement with stale lease objects) produces exactly the symptom of pods sitting `Pending` with zero events — nothing is running the scheduling loop at all.

### Decision flow (filter -> score -> bind), and where preemption fits

```
                          New Pod (Pending, unbound)
                                    |
                                    v
                     +---------------------------+
                     |   PreFilter (per-pod)      |
                     +---------------------------+
                                    |
                                    v
                 +-----------------------------------+
                 |  Filter (per node, all nodes)       |
                 |  - taints/tolerations                |
                 |  - node affinity / nodeSelector      |
                 |  - pod affinity/anti-affinity        |
                 |  - resource fit (requests vs         |
                 |    allocatable)                      |
                 |  - max-pods per node                 |
                 |  - volume node affinity (PV zone)    |
                 |  - topology spread (DoNotSchedule)   |
                 +-----------------------------------+
                                    |
                    node list after filtering
                                    |
                +-------------------+-------------------+
                |                                       |
        >= 1 node passed                        0 nodes passed
                |                                       |
                v                                       v
      +-------------------+                  +-----------------------+
      | Score (per node)   |                  |  Preemption cycle:     |
      | - NodeResourcesFit |                  |  find lower-priority   |
      | - InterPodAffinity |                  |  victims whose removal |
      | - PodTopologySpread|                  |  lets pod fit; evict   |
      | - node affinity    |                  |  fewest, prefer not    |
      |   preferred weight |                  |  violating PDBs (not   |
      | -> weighted sum    |                  |  guaranteed); pod gets |
      +-------------------+                  |  nominatedNodeName,    |
                |                              |  re-enters queue      |
                v                              +-----------------------+
      +-------------------+                              |
      | Reserve / Permit   |                              v
      | (framework hooks,  |                     retried on next cycle
      |  e.g. gang wait)   |                     (victims may already
      +-------------------+                      be terminating)
                |
                v
      +-------------------+
      | PreBind / Bind     |
      | (volume attach,    |
      |  API server bind)  |
      +-------------------+
                |
                v
         Pod bound to node
       (kubelet takes over)
```

If a pod fails Filter on every node with a resource-based reason (`Insufficient cpu`/`memory`), preemption is attempted before the pod is reported as unschedulable — but only lower-priority pods are ever candidates, and if every remaining pod on every node is equal-or-higher priority (or the pod's own priority is unset/zero, the default), there's nothing to preempt and it just sits `Pending` with the `FailedScheduling` event as the only signal, cycling through the scheduling queue with exponential backoff until something changes (new node, pod deletion elsewhere, label change).
