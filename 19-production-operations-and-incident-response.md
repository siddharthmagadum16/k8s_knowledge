# Production Operations and Incident Response

## GitOps: Git as the source of truth

The core idea: the live cluster state is a **derived artifact**, not the source of truth. The Git repository is the source of truth. A controller running inside (or alongside) the cluster continuously reconciles — diffs desired state (manifests in Git) against actual state (live objects in the API server) — and either auto-syncs the drift away or flags it for a human.

```
   Git repo                    ArgoCD/Flux controller               Cluster
 (deployment.yaml)  ---diff-->   reconcile loop            ---apply-->  live objects
       ^                              |                                    |
       |                        detects drift <----------------------------
       |                              |
       +----- git revert / PR merge --+
```

This is fundamentally different from imperative `kubectl apply` pipelines where CI pushes to the cluster and nothing pulls. In GitOps, the controller pulls, on a loop (typically every 3 minutes for ArgoCD by default, or on webhook).

### The `kubectl edit` gotcha

This is the single most common thing that trips up engineers coming from a pre-GitOps world:

```bash
# "just a quick hotfix" during an incident
kubectl edit deployment checkout-api -n prod
# bump replicas 3 -> 10, or patch an env var
```

If the Application is set to auto-sync, ArgoCD's next reconcile loop (default every 180s) sees that live state no longer matches Git, and **reverts your edit back to what's in Git** — silently, with no warning banner, no Slack message. Your hotfix vanishes and the on-call engineer has no idea why until they check ArgoCD's sync history and realize it "self-healed" your change away.

If auto-sync is off (manual sync mode), the edit doesn't get reverted immediately, but the Application shows `OutOfSync`, and the next person who clicks "Sync" (possibly weeks later, possibly a different engineer doing an unrelated release) will silently revert your hotfix as collateral damage of their sync, because they didn't even know it existed.

The correct move during an incident in a GitOps-managed cluster:
1. Make the fix in Git (a real commit, even a rushed one on a hotfix branch), push, let it sync. This is almost always fast enough — seconds to a couple minutes.
2. Only if genuinely `kubectl edit` is faster than a git push in a live-fire situation: pause auto-sync (`argocd app set <app> --sync-policy none`) or annotate the resource to exclude it from that sync (`argocd.argoproj.io/compare-options: IgnoreExtraneous` on that field), do the edit, and **immediately** open the PR to bring Git in line with what you just did live, then re-enable auto-sync. Skipping the "bring Git in line" step is how the same incident recurs a week later when someone else syncs and silently undoes your fix.

### Why this matters: auditability and rollback speed

- **Auditability**: every production change is a Git commit, tied to an author, ideally behind a PR review. `git log -p -- deployment.yaml` gives you a complete, tamper-evident (if the repo is protected) history of every change ever made to prod. Compare this to a fleet of engineers running ad hoc `kubectl apply -f` / `kubectl edit` from their laptops — the only audit trail is the Kubernetes audit log (if enabled, if retained long enough, if anyone's looking at it), and there's no code review gate at all.
- **Rollback speed**: `git revert <bad-commit> && git push` — the controller resyncs automatically and the cluster goes back to the exact prior state. Compare to the imperative world: you're now reconstructing "what did prod actually look like before someone's kubectl edit an hour ago" from memory, Slack scrollback, or `kubectl rollout history` (which only covers Deployment pod-template revisions, not ConfigMaps, RBAC, NetworkPolicies, or anything else that changed at the same time).

### Example ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/acme/k8s-manifests.git
    targetRevision: main
    path: apps/checkout-api/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true     # revert manual drift automatically (the gotcha above)
    syncOptions:
      - CreateNamespace=false
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

`prune: true` is the other sharp edge: if someone deletes a file from the Git path without meaning to, ArgoCD will delete the corresponding live object on the next sync. There is no "are you sure" prompt in automated mode.

### Sync waves and hooks — ordering matters

A flat `kubectl apply -f manifests/` applies everything roughly at once and lets Kubernetes' own eventual consistency sort out ordering (which mostly works, until it doesn't — e.g. a CRD-backed resource applied before its CRD exists just errors out). ArgoCD sync waves give you explicit ordering within one Application sync:

```yaml
# CRD must exist before any CR using it
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqls.acid.zalan.do
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
---
# DB migration Job runs after CRD, before app rollout
apiVersion: batch/v1
kind: Job
metadata:
  name: checkout-api-migrate
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: acme/checkout-api-migrate:1.4.2
          command: ["./migrate", "up"]
      restartPolicy: Never
---
# app deployment, default wave 0, runs last
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

Waves run low-to-high; ArgoCD waits for each wave's resources to reach Healthy before starting the next wave. `PreSync`/`Sync`/`PostSync` hooks let you run one-shot Jobs (migrations, cache warmers, smoke tests) at specific points without them being permanent Application members that get pruned/reconciled like a Deployment. `hook-delete-policy: HookSucceeded` cleans the migration Job up once it completes so it doesn't linger and get re-run oddly on subsequent syncs.

Flux equivalent: `Kustomization` resources with `dependsOn` fields for cross-Kustomization ordering, and Flux's own `HelmRelease` with a `postRenderers`/health check gate. The conceptual pattern (declare a dependency graph rather than a flat unordered apply) is the same across both tools.

## Deployment strategies beyond RollingUpdate

### Blue-green

Two fully separate environments running simultaneously (`checkout-api-blue`, `checkout-api-green`), each with its own full replica set. Traffic cutover is instant — flip a Service selector or an Ingress/LB target group from blue to green:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout-api
spec:
  selector:
    app: checkout-api
    version: blue   # change to green to cut over — takes effect in seconds
  ports:
    - port: 80
      targetPort: 8080
```

Trade-off: you're running 2x the pod count (and often 2x the cost) during the transition window, and databases/shared state are the hard part — blue and green usually share the same DB, so schema changes must be backward-and-forward compatible with both versions running at once (expand/contract migration pattern). Rollback is trivially fast (flip the selector back), which is the entire point — you pay the 2x cost for near-zero rollback risk.

### Canary with automated analysis

Manual canaries ("route 10% of traffic to the new version, watch a dashboard, decide by eye") don't scale past a handful of services and are slow under incident pressure. Argo Rollouts and Flagger automate the "watch a dashboard, decide" part by querying a metrics backend (usually Prometheus) at each traffic step and automatically rolling back on threshold breach.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-api
spec:
  replicas: 10
  strategy:
    canary:
      analysis:
        templates:
          - templateName: success-rate-and-latency
        startingStep: 1        # skip analysis on the first step (5%)
        args:
          - name: service-name
            value: checkout-api
      steps:
        - setWeight: 5
        - pause: {duration: 2m}
        - setWeight: 25
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
  selector:
    matchLabels:
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      containers:
        - name: checkout-api
          image: acme/checkout-api:8f3a91c
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-and-latency
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 1m
      count: 5
      successCondition: result < 0.02
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
    - name: p99-latency
      interval: 1m
      count: 5
      successCondition: result < 0.5
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[2m])) by (le)
            )
```

If `error-rate` or `p99-latency` breach their `successCondition` for `failureLimit` consecutive checks, Argo Rollouts automatically sets weight back to 0 and marks the Rollout `Degraded` — no human needed to notice and no human needed to hit the rollback button. This is the difference between a canary that catches a bad deploy at 2am with nobody watching, and one that doesn't.

### Feature flags — decoupling deploy from release

A **deploy** puts new code onto production nodes. A **release** is the moment users are actually exposed to the new behavior. Feature flags let these be two separate, independently-controlled events:

```
deploy (code ships, flag OFF, dormant)  -->  release (flag flipped ON for some/all users)
```

Why this matters operationally:
- You can deploy during business hours (when your team is fully staffed and alert, the ideal time to catch a bad deploy) without exposing risky new behavior to users yet — the code path is dark.
- When something breaks after a release, you now have two independent questions and two independent fast levers: "did the deploy break something" (rollback the deployment / revert the image) vs. "did the *feature* break something" (flip the flag off — instant, no rollout, no pod restarts, no rebuild). Flag-off is usually a config change that takes effect in seconds; a Kubernetes rollback of a Deployment still takes a full rollout cycle.
- Progressive exposure (1% of users, then 10%, then 100%) can be done independently of infrastructure canary percentages, and can be targeted (internal staff first, then a specific customer segment) in ways a traffic-percentage canary at the Kubernetes Service layer cannot.

The trade-off: flag debt. Flags that are never cleaned up after full rollout accumulate into a maze of dead conditionals. Treat "remove the flag" as a mandatory follow-up ticket at rollout time, not an optional cleanup.

## CI/CD pipeline shape for Kubernetes

```
source commit
   -> build image
   -> vulnerability scan (Trivy/Grype) -- fail build on CRITICAL/HIGH (policy-dependent)
   -> push to registry, tagged by git-sha or semver
   -> bot (Renovate / Argo CD Image Updater) opens a PR bumping the tag in the
      manifests repo
   -> PR reviewed/merged
   -> GitOps controller (ArgoCD/Flux) detects the Git change, syncs the cluster
```

```bash
# vulnerability gate, typical CI step
trivy image --severity CRITICAL,HIGH --exit-code 1 acme/checkout-api:8f3a91c
```

The separation between "CI pushes an image" and "CI touches the cluster" is intentional: CI systems get zero direct write access to the cluster's API server. The only path into the cluster is a Git commit that the GitOps controller pulls — which keeps the previous section's auditability guarantee intact even for fully automated pipelines.

### Why `:latest` is actually dangerous, not just sloppy

The "don't use `:latest`" rule is usually taught as a style preference. The real mechanism is worse than that:

`imagePullPolicy: IfNotPresent` (the default when the tag isn't `latest`... except when the tag *is* `latest`, `imagePullPolicy` defaults to `Always` — but plenty of manifests explicitly set `IfNotPresent` anyway, or nodes already have a cached layer that matches) means: a node only re-pulls an image if it doesn't already have something cached under that exact tag string. The tag `checkout-api:latest` is not a pointer to one immutable artifact — it's a mutable label that gets reassigned to a new image digest every time you push. Concretely:

```
Day 1, 10:00 - build pushes checkout-api:latest -> digest sha256:aaa...
Day 1, 10:05 - Node A schedules a pod, pulls latest -> gets sha256:aaa
Day 1, 14:00 - build pushes checkout-api:latest -> digest sha256:bbb... (new code)
Day 1, 14:10 - Node B schedules a pod (scaled up, or rescheduled after eviction),
               pulls latest -> gets sha256:bbb
```

Node A and Node B are now running **different actual code** while both `kubectl get pods -o wide` and `kubectl describe pod` report the same image string `checkout-api:latest`. There is no way to tell them apart from the Kubernetes API without inspecting the actual running digest (`kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].imageID}'`).

It gets worse for rollback. `kubectl rollout undo deployment/checkout-api` works by reverting the pod template to a prior ReplicaSet's spec. If that spec says `image: checkout-api:latest`, "rolling back" doesn't roll back anything — it re-pulls whatever `latest` currently resolves to, which by now is the *same new broken image* you were trying to escape, not the old good one. The ReplicaSet history is intact but semantically useless because the tag it stores is not an immutable reference to a specific artifact.

**Fix — immutable tagging:**

```yaml
# git-sha tagging: every build is unambiguous, tied back to source
image: acme/checkout-api:8f3a91c

# or full digest pinning: the only fully deterministic option
image: acme/checkout-api@sha256:1a2b3c4d5e6f...
```

Digest pinning is strictly stronger than sha-tag pinning: a tag (even a git-sha tag) can technically be force-overwritten in the registry (`docker push` to the same tag again, or a rebuild that reuses a tag), whereas a digest is a content hash — it is the artifact, not a label pointing at one. Renovate and Argo CD Image Updater both support digest-pinning update strategies, so "immutable" doesn't have to mean "manual."

## On-call reality: what actually pages

Concretely, in a Kubernetes environment, these are the pages that recur:

- **Node NotReady** — kubelet stopped heartbeating to the API server. Root cause is one of: kubelet process crashed/hung, container runtime (containerd/CRI-O) wedged, network partition between node and control plane, disk pressure severe enough that kubelet itself can't operate, or the node is genuinely dead (hardware, host-level VM eviction on cloud spot/preemptible instances).
- **High error rate / high latency** — a symptom-based SLO alert (burn-rate alerting, covered in doc 18), not a cause. The page tells you *that* users are affected, not *why*.
- **PVC full** — either disk-pressure-triggered pod eviction cascades (see the war story below) or application-level write failures (`no space left on device` in app logs, writes failing, WAL/log files unable to rotate).
- **Control plane API latency/errors** — etcd running slow (usually disk latency on etcd's data directory, or too many large objects/events bloating the keyspace), apiserver overloaded (too many concurrent requests, admission webhooks with slow backends adding serial latency to every write, or informer/watch storms — see the war story below).

### Runbook structure that survives being read at 3am

A runbook that reads well in a design doc but requires the on-call engineer to *think* under pressure has failed at its one job. The structure that actually works:

```
Title / Severity
Symptom            <- verbatim what the alert/dashboard says
Likely causes       <- ranked by probability, not by interestingness
Diagnostic commands <- copy-pasteable, in the order you'd actually run them
Mitigation          <- fastest SAFE action first (stop the bleeding),
                       root-causing comes after, not before
Escalation path     <- who/what next if mitigation doesn't work in N minutes
Dashboards/links
```

The ordering of "mitigation before root cause" is deliberate and the most commonly violated rule in badly-written runbooks. At 3am, with users actively affected, the job is to stop the bleeding (scale up, roll back, fail over) first and understand exactly why later, in daylight, with the postmortem. A runbook that opens with "first, investigate whether it's X or Y" instead of "first, run this mitigation" costs minutes that matter.

#### Worked example: API server latency high

```
Title: API server p99 latency > 2s
Severity: SEV2 (cluster-wide degradation, not yet a full outage)

Symptom:
  Alert: apiserver_request_duration_seconds p99 > 2s for 5m, on GET/LIST verbs
  kubectl commands from any client (including on-call's own laptop) feel sluggish

Likely causes (ranked):
  1. etcd slow — disk latency on etcd's WAL/data dir, or etcd DB size approaching
     the 8GB default quota causing frequent compaction/defrag pauses
  2. A controller/operator doing full LIST calls in a hot reconcile loop instead
     of watch+resourceVersion (see war story below)
  3. Admission webhook(s) with high latency or timeouts, serializing onto every
     write request
  4. Genuine request volume spike (legit traffic, or a client stuck in a retry
     loop with no backoff)

Diagnostic commands (run in this order):
  # 1. Confirm and scope: which verb/resource is slow
  kubectl get --raw /metrics | grep apiserver_request_duration_seconds | grep -v quantile=\"0\"

  # 2. etcd health directly
  kubectl -n kube-system exec etcd-<node> -- etcdctl endpoint health --cluster
  kubectl -n kube-system exec etcd-<node> -- etcdctl endpoint status --cluster -w table
  # look at DB_SIZE column vs quota, and RAFT_TERM stability (flapping = trouble)

  # 3. Identify noisy API clients by user-agent from audit logs
  kubectl get --raw /metrics | grep apiserver_current_inflight_requests
  # then, if audit logging is enabled:
  grep '"verb":"list"' /var/log/kubernetes/audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn | head

  # 4. Check admission webhook latency
  kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds

Mitigation (fastest safe action first):
  1. If a specific client/controller is identified as the noisy one (step 3):
     scale it to 0 replicas immediately. This is reversible and low-risk —
     do it before you're fully sure, if it's the top suspect.
     kubectl scale deployment <noisy-controller> -n <ns> --replicas=0
  2. If etcd DB size is near quota: trigger compaction + defrag (coordinate with
     whoever owns etcd — defrag briefly blocks that member).
  3. If a webhook is the cause and it's non-critical (e.g. a policy-check
     webhook, not an auth webhook): temporarily set failurePolicy: Ignore or
     delete the webhook configuration to unblock the API server, file a
     follow-up to fix the webhook's own latency.
  4. If it's a genuine traffic spike with no single bad actor: this is a
     capacity problem, not a config problem — escalate for apiserver vertical
     scaling / additional apiserver replicas behind the LB, this isn't a
     15-minute fix.

Escalation path:
  - No improvement in 15 min -> page platform-infra secondary
  - etcd cluster health degraded (quorum at risk) -> page immediately, do not wait

Dashboards:
  - Grafana: "Kubernetes API Server" (apiserver latency, inflight requests, etcd)
  - Grafana: "etcd" (DB size, WAL fsync latency, leader changes)
```

## War stories

These are the failure modes that don't show up in tutorials because they only emerge from real production load, real hardware, and real timing. Each one is a chain of individually-reasonable decisions that combine into an outage.

### Cascading OOM eviction chain reaction

**What happened:** A `Deployment` for a batch-processing sidecar had its memory limit set to `256Mi` — copy-pasted from a template, never sized against the workload's actual memory profile (which, on real data, spiked to ~400Mi under load). The container got OOMKilled repeatedly, restarted, OOMKilled again. So far this looks contained — one workload, unhealthy, restarting.

It wasn't contained, because the node itself was running close to its allocatable memory across *all* pods scheduled on it (bin-packed tightly, as cost optimization intends). Every OOMKill of the misconfigured container briefly spiked memory pressure on the node as the kernel reclaimed and the container restarted and re-warmed. The node crossed its `memory.available` eviction threshold. The kubelet's eviction manager kicked in — and it doesn't just evict the pod causing the pressure. It ranks *all* pods on the node by QoS class (BestEffort evicted before Burstable before Guaranteed) and evicts to reclaim, regardless of who's actually responsible. Several unrelated BestEffort pods (no requests/limits set at all — another "someone forgot to set these" problem) got evicted from that node.

Those evicted pods rescheduled onto other nodes in the cluster — which, because the cluster was running close to overall capacity, pushed *those* nodes closer to their own memory thresholds. The eviction manager started tripping on the next nodes. This is the cascade: one misconfigured limit turned into a multi-node memory-pressure wave, taking down services that had nothing to do with the original bad container.

**Diagnostic trail:**

```bash
# Node-level condition — this is the first thing to check when things "just started failing"
kubectl get nodes -o json | jq '.items[] | {name:.metadata.name, conditions:.status.conditions}'
# look for MemoryPressure: "True"

# Events show both the OOMKills and the evictions
kubectl get events -A --sort-by='.lastTimestamp' | grep -Ei 'oom|evict'

# Confirm actual OOM kills at the kernel level (kubectl events can lag or be pruned)
# on the node itself:
journalctl -u kubelet --since "1 hour ago" | grep -i evict
dmesg -T | grep -i "killed process"
# dmesg is the ground truth here — the OOM killer is a kernel mechanism,
# kubectl/kubelet reporting is secondary to it

# Which pods actually got evicted, and why
kubectl get pods -A -o json | jq -r '.items[] | select(.status.reason=="Evicted") | "\(.metadata.namespace)/\(.metadata.name): \(.status.message)"'
```

**Fix:**
- Correct the offending container's requests/limits against actual observed memory usage (with headroom, not copy-pasted defaults) — `256Mi` limit for a `400Mi` real workload was the root trigger.
- Give every pod requests/limits — the BestEffort pods that got caught in the blast radius had none, which is *why* they were first in line for eviction. BestEffort should be a deliberate choice for genuinely disposable workloads, not the accidental default from a missing YAML field.
- Move critical services to Guaranteed QoS (requests == limits for both CPU and memory) so they're evicted last, not first, under node pressure.
- Add PodDisruptionBudgets on critical Deployments so a wave of evictions/reschedules can't take down every replica of a service at once — this doesn't prevent the eviction but slows and bounds the blast radius.
- Consider `system-reserved`/`kube-reserved` kubelet flags and eviction threshold tuning (`eviction-hard`, `eviction-soft`) so nodes trip into MemoryPressure earlier, with a smaller number of pods affected, rather than late and violently.

### Thundering herd on pod restart hitting a database

**What happened:** A routine Deployment rollout (`kubectl rollout restart deployment/api`, done to pick up a ConfigMap change) restarted 40 pods across the fleet. Because `maxUnavailable`/`maxSurge` weren't tuned, and there was no jitter on startup, a large fraction of those 40 pods hit "ready" (per their readiness probe) within the same few-second window, and every one of them opened a full connection pool (say, 20 connections each) to Postgres on startup. 40 pods x 20 connections landing within a 5-second window is 800 new connections hitting a database configured for a few hundred max — connections got refused or queued, requests already in flight from *already-running* pods (not just the new ones) started timing out waiting on the connection pool, and the timeout retries added even more connection attempts. The database's own connection count alerting paged separately from the application error-rate alerting, making the incident initially look like two unrelated problems.

**Root cause chain:** rollout restart -> synchronized readiness -> synchronized connection pool warm-up -> connection count spike exceeds DB max_connections -> queuing/refusal -> app-level timeouts -> retry amplification -> sustained overload that outlasted the original rollout.

**Fix:**
- Cap per-pod connection pool size conservatively and enforce it at the DB side too (`max_connections` with headroom, PgBouncer/RDS Proxy as a connection multiplexer in front of Postgres so pod-count x pool-size doesn't map 1:1 onto real DB connections).
- Add jitter to startup — a small random delay before a pod opens its DB pool (or before its readiness probe can pass) spreads the reconnect storm over tens of seconds instead of one 5-second burst.
- Tune `maxUnavailable`/`maxSurge` on the Deployment (or `maxUnavailable` on a PDB used during node drains) so restarts roll through the fleet in smaller batches rather than most-at-once.
- Client-side exponential backoff with jitter on connection retry, not fixed-interval retry — fixed-interval retry from many clients simultaneously just re-synchronizes the next spike.
- A readiness gate that only flips ready after the pod has done a lightweight warm-up (open pool gradually, confirm one successful query) rather than the instant the process forks, spreads the "hit the DB" moment out naturally.

### DNS resolution storm from `ndots:5`

**What happened:** A service calling an external API (`api.stripe.com`) started seeing DNS resolution timeouts and elevated latency cluster-wide under load, and CoreDNS pods showed CPU throttling and dropped queries. The service itself hadn't changed.

**Mechanism, precisely:** every pod's `/etc/resolv.conf` (unless overridden) is generated with Kubernetes' default `ndots:5` and a search list of the cluster's DNS suffixes:

```
search prod.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

`ndots:5` means: if a queried name has *fewer than 5 dots* in it, the resolver tries every entry in `search` appended to the name **before** trying the name as given (absolute). `api.stripe.com` has 2 dots — well under 5 — so the actual sequence of DNS queries the pod issues, for a single application-level lookup, is:

```
api.stripe.com.prod.svc.cluster.local.   -> NXDOMAIN
api.stripe.com.svc.cluster.local.        -> NXDOMAIN
api.stripe.com.cluster.local.            -> NXDOMAIN
api.stripe.com.<node search domain>.     -> NXDOMAIN (if the node adds its own)
api.stripe.com.                          -> success, finally
```

One external DNS lookup that should be one query becomes 4-5 queries, all but the last guaranteed to fail. Under light load this is just wasted latency (a few extra round trips to CoreDNS, each one usually fast). Under real load — many pods, all doing this for every external hostname on every connection setup, especially if the app or its HTTP client library isn't caching resolved addresses and re-resolves per request — this multiplies real query volume against CoreDNS by 4-5x. CoreDNS, sized for the "1 query per lookup" assumption, saturates: query latency rises, some queries get dropped, and now *internal* service-to-service DNS lookups queue behind the external-lookup storm too, so the failure spreads to traffic that has nothing to do with Stripe.

**Fix:**
- Trailing dot on fully-qualified external hostnames in app config/code (`api.stripe.com.`) — a trailing dot marks the name as already-absolute, skipping the search-list expansion entirely. Easy to forget, easy to enforce via lint/config convention.
- Explicit `dnsConfig` on the pod spec to lower `ndots` for workloads that mostly talk to external hosts:

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```
  (Lowering cluster-wide via kubelet/cluster DNS config is riskier — short internal names with fewer than the new ndots value might stop resolving correctly; tune per-workload via `dnsConfig` first.)
- **NodeLocal DNSCache** — runs a caching DNS agent as a DaemonSet on every node, so repeated identical queries (including the repeated failed search-suffix attempts) get served from a local cache instead of hitting CoreDNS over the network every time. This is the highest-leverage fix at cluster scale because it helps every workload without requiring each one to be individually reconfigured.
- Application-level connection reuse/keep-alive so DNS is resolved once per connection lifetime, not per request.

### Bad HPA config causing scale flapping

**What happened:** An HPA targeting CPU utilization on a bursty, spiky workload (traffic pattern: real load in irregular short bursts, not smooth) had no stabilization window configured. Each burst pushed average CPU briefly over the target, HPA scaled up; the burst subsided a minute later, average CPU dropped, HPA scaled back down; the next burst arrived while the fleet was still small, pushing CPU over target again. The result was a scale-up/scale-down cycle repeating every few minutes, each cycle churning pods (new pods costing cold-start time before serving traffic effectively, old pods being terminated mid-connection-drain), producing periodic p99 latency spikes and connection resets that were harder to diagnose than a flat capacity shortage because they came and went.

**Fix** (detailed HPA behavior tuning is covered in doc 18 — summary here):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # require 5 min of sustained low load before scaling down
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up fast, that direction is safe to be eager on
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
```

Asymmetric stabilization — fast to scale up, slow and dampened to scale down — is usually the right default: over-provisioning briefly costs money, flapping costs user-facing latency and reliability.

### Control plane overload from too many watch connections

**What happened:** A custom operator, written in-house, had a reconcile loop bug: instead of using the informer/watch pattern (establish one long-lived watch per resource type, get incremental updates via `resourceVersion`), a code path fell back to issuing a fresh `LIST` across all namespaces on every reconcile tick, and the reconcile loop itself was firing far more often than intended (a requeue-on-every-event bug with no rate limiting). Multiplied across every namespace the operator watched and every replica of the operator (it wasn't leader-elected correctly, so more than one replica was active), this produced a sustained high rate of expensive full-LIST calls against the apiserver, each one paginating through potentially thousands of objects and serializing the results — CPU- and memory-expensive on the apiserver side, and each one held connections and consumed inflight-request budget. Apiserver CPU and memory climbed, request latency for *all* clients (including kubectl for on-call engineers trying to diagnose the problem) rose sharply, and eventually admission started timing out for unrelated write requests across the cluster.

**Diagnostic:**

```bash
# inflight request counts — a sudden climb here with no legitimate traffic
# increase is the tell
kubectl get --raw /metrics | grep apiserver_current_inflight_requests

# identify the noisy client by user-agent (every apiserver request is tagged
# with the client's user-agent string, which for a controller-runtime-based
# operator usually includes its binary name/version)
kubectl get --raw /metrics | grep apiserver_request_total | grep -v ' 0$' | sort -t'=' -k2 | head -30

# audit logs, if enabled, give a definitive per-request breakdown
grep '"verb":"list"' /var/log/kubernetes/audit.log \
  | jq -r '.userAgent' | sort | uniq -c | sort -rn | head
```

**Fix:**
- Fix the operator itself: use `resourceVersion`-based resumable watches (what `client-go` informers give you for free when used correctly) instead of polling with LIST, and add a requeue rate limiter (`workqueue.RateLimiter`) so a single hot object can't cause an unbounded reconcile-retry loop.
- Fix leader election so only one replica of the operator is active against a given set of resources — most controller-runtime scaffolding (kubebuilder, operator-sdk) enables this by default; it had been explicitly (and incorrectly) disabled in this case for a "want all replicas warm" reason that didn't actually require all replicas to be reconciling simultaneously.
- Client-side QPS/Burst settings: every client built on client-go has `--kube-api-qps` / `--kube-api-burst` (or the equivalent in-code rest.Config fields) that cap how fast that one client can hammer the apiserver, independent of fixing the underlying bug — a useful blunt-force mitigation while the real fix is being rolled out.
- **API Priority and Fairness (APF)**: define a `FlowSchema` + `PriorityLevelConfiguration` so a misbehaving or excessive client gets its own bounded concurrency slice instead of competing unbounded with critical traffic (kubelet heartbeats, scheduler, other controllers):

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: custom-operator-limited
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 5
    limitResponse:
      type: Queue
      queuing:
        queues: 16
        queueLengthLimit: 25
        handSize: 4
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: custom-operator-flow
spec:
  priorityLevelConfiguration:
    name: custom-operator-limited
  matchingPrecedence: 500
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: custom-operator
            namespace: platform
      resourceRules:
        - apiGroups: ["*"]
          resources: ["*"]
          verbs: ["*"]
```

APF is the structural fix — it bounds the blast radius of *any* future noisy client (not just this specific operator) so one badly-behaved controller can degrade its own slice of apiserver capacity, not the whole control plane.

## Chaos engineering

The premise: assumptions about resilience ("we run 3 replicas so we can survive losing 1 pod", "we have anti-affinity so we're spread across AZs") are untested claims until something actually kills a pod, a node, or a network path and you observe what really happens. Production will eventually run that test for you, at the worst possible time, with no rollback button. Chaos engineering means running it deliberately, on your own schedule, with a rollback button.

Common experiment types (Chaos Mesh, Litmus):

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-one-checkout-api-pod
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: ["prod"]
    labelSelectors:
      app: checkout-api
  scheduler:
    cron: "@every 1h"
---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-network-delay
spec:
  action: delay
  mode: all
  selector:
    namespaces: ["prod"]
    labelSelectors:
      app: checkout-api
  delay:
    latency: "200ms"
    jitter: "50ms"
  duration: "5m"
  direction: to
  target:
    selector:
      namespaces: ["prod"]
      labelSelectors:
        app: postgres
    mode: all
```

Other common experiment types: disk-fill (validate PVC-full / disk-pressure handling actually behaves the way the runbook above assumes), simulated node-drain (validate PDBs actually prevent full-service outage during a drain, not just during a normal rolling deploy), CPU/memory stress (validate HPA and eviction thresholds behave as configured under real pressure rather than theoretical assumption).

The specific assumption worth testing hardest: "3 replicas survives 1 failure" is only true if those 3 replicas are actually spread across failure domains. Without explicit `topologySpreadConstraints` (or at minimum pod anti-affinity), the scheduler is free to place all 3 on the same node or the same AZ if that happens to be where capacity is available at scheduling time — nothing about `replicas: 3` alone guarantees distribution:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: checkout-api
```

A `pod-kill` or `node-drain` chaos experiment against a Deployment that lacks this constraint is exactly how a team discovers, in staging rather than during a real AZ outage, that all 3 replicas were sitting on the same node the whole time.

Sequencing that works in practice: run experiments in staging first with realistic traffic replay, fix what breaks, then graduate to **game days** — scheduled, announced, controlled chaos experiments in production, with the on-call team actively watching dashboards and ready to abort, treated as a drill rather than a surprise. The goal of a game day is confirming the runbooks and dashboards actually work under a real failure, not just that the system survives — an unreadable dashboard or a wrong runbook step discovered during a scheduled game day is a win; discovered during a real 3am outage it's an added cost on top of the outage itself.

## Blameless postmortem culture

The war stories above only get written down accurately, with the real root cause chain intact, if the engineers involved are not afraid that admitting "I set the memory limit to 256Mi from a template without checking" or "I ran `kubectl rollout restart` without thinking about connection pool warm-up" will be held against them personally. Blame-oriented postmortem culture doesn't reduce mistakes — it reduces the *reporting* of mistakes. The cascading OOM story above has four or five distinct contributing layers (wrong limit, missing limits elsewhere, no PDB, wrong QoS class); a postmortem culture where the first responder is worried about being blamed for "causing the outage" will tend to stop at layer one ("fixed the memory limit, done") and the other layers stay live, waiting to cause the next incident.

Structure of a postmortem that actually improves the system, rather than just documenting that an incident happened:

```
Title, date, severity, duration, services affected

Timeline
  - timestamped, factual, no interpretation mixed in
  - 14:02 alert fired: checkout-api error rate > 5%
  - 14:05 on-call acknowledged, began investigation
  - 14:11 identified OOMKill events on node-17
  - 14:19 mitigated via cordon + drain of node-17
  - 14:40 error rate back to baseline

Impact
  - concrete numbers: X requests failed, Y minutes of degraded checkout,
    estimated revenue/customer impact if material

Root cause
  - the *real* one, pursued with "5 whys" past the first plausible answer:
    why did checkout fail? -> pods on node-17 were unreachable
    why? -> node-17 had MemoryPressure and was evicting pods
    why? -> the sidecar's OOMKill loop triggered eviction manager
    why? -> sidecar memory limit (256Mi) was below its real usage (400Mi)
    why? -> limit was copy-pasted from a template during initial setup and
            never validated against actual load testing
  - stopping at "sidecar OOMKilled" is a superficial fix (bump the limit) that
    leaves the templating/no-load-testing process defect in place for the
    next service

Contributing factors
  - no PDB on the affected BestEffort pods, which widened the blast radius
  - no alert on node MemoryPressure condition specifically (only alerted on
    the downstream symptom, error rate, costing ~10 minutes of detection time)

Action items (owner, deadline — every item has both)
  - [ ] set correct requests/limits on sidecar, validated against load test
        data -- @owner, due in 3 days
  - [ ] add default requests/limits enforcement via LimitRange or admission
        policy so no pod can ship with none set -- @owner, due in 2 weeks
  - [ ] add PodDisruptionBudget to all prod Deployments via policy check in CI
        -- @owner, due in 2 weeks
  - [ ] alert directly on node MemoryPressure=True, not just downstream
        symptoms -- @owner, due in 1 week

No blame section (explicit, not implied)
  - this postmortem intentionally does not attribute individual fault; the
    action items address process and system gaps, not individual performance
```

The "5 whys" habit is what separates a postmortem that closes with "we bumped the memory limit" from one that closes with "we now enforce that no pod ships without requests/limits set, cluster-wide, via policy" — the second one prevents an entire category of future incidents, the first prevents exactly one recurrence of this exact one.
