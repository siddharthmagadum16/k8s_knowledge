# Custom Resources and Operators

## 1. CRDs: extending the API server, not bolting something on the side

A CustomResourceDefinition does not create a sidecar service or a new database. It registers a new
**resource type inside kube-apiserver itself**. Once the CRD is accepted, `kube-apiserver` dynamically
adds a new REST path, wires it into the same discovery, authentication, authorization, admission,
validation, and etcd-storage pipeline that `Pod`, `Deployment`, and every other built-in type go
through. There is no separate "CRD apiserver" — it is the same binary, same process, same request
handling code path, just with a schema and storage path generated at runtime instead of compiled in.

```
kubectl get widgets.example.com my-widget
        |
        v
kube-apiserver (aggregation layer routes /apis/example.com/v1/widgets/my-widget)
        |
        +--> authn/authz (same as any resource)
        +--> admission chain (mutating + validating webhooks, same chain)
        +--> schema validation (structural schema, same OpenAPI validation machinery)
        v
etcd key: /registry/example.com/widgets/<namespace>/my-widget
```

This is why CRDs "feel native": `kubectl get`, `kubectl describe`, `kubectl edit`, RBAC verbs
(get/list/watch/create/update/patch/delete), label selectors, field selectors (limited), server-side
apply, and finalizers all work identically to built-in types, because it is genuinely the same
mechanism, not an approximation of it.

### A real CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.example.com
spec:
  group: db.example.com
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames:
      - pgc
  versions:
    - name: v1beta1
      served: true
      storage: false
      deprecated: true
      deprecationWarning: "db.example.com/v1beta1 PostgresCluster is deprecated; use v1"
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                version:
                  type: string
      subresources:
        status: {}
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: ["spec"]
          properties:
            spec:
              type: object
              required: ["replicas", "version"]
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 9
                version:
                  type: string
                  enum: ["14", "15", "16"]
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                      pattern: '^[0-9]+Gi$'
                    storageClassName:
                      type: string
            status:
              type: object
              properties:
                readyReplicas:
                  type: integer
                primary:
                  type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type: { type: string }
                      status: { type: string }
                      reason: { type: string }
                      message: { type: string }
                      lastTransitionTime: { type: string, format: date-time }
                      observedGeneration: { type: integer }
      additionalPrinterColumns:
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Ready
          type: integer
          jsonPath: .status.readyReplicas
        - name: Primary
          type: string
          jsonPath: .status.primary
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
          labelSelectorPath: .status.labelSelector
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: db-operator-system
          name: db-operator-conversion-webhook
          path: /convert
          port: 443
        caBundle: <base64 CA bundle>
```

Discovery confirms the new type is a first-class citizen:

```bash
kubectl api-resources | grep postgres
# postgresclusters   pgc          db.example.com/v1   true         PostgresCluster

kubectl get --raw /apis/db.example.com/v1 | jq .
```

### Structural schema requirement (since v1)

Since `apiextensions.k8s.io/v1`, every version's `schema.openAPIV3Schema` **must be a structural
schema**. That means, non-negotiably:

- Every field that can appear must be declared under `properties` (or `additionalProperties` if you
  genuinely need a map), with a `type`.
- `type` must be set at every level of the tree that isn't a pure logical combinator.
- You cannot mix `oneOf`/`anyOf`/`allOf` at the root without also giving the root a `type`.
- Unknown fields are pruned by default (`x-kubernetes-preserve-unknown-fields: true` opts a subtree
  out of pruning, which you rarely want).

The API server enforces this at CRD-creation time — an unstructured schema is **rejected outright**,
not silently accepted:

```
The CustomResourceDefinition "widgets.example.com" is invalid:
spec.versions[0].schema.openAPIV3Schema: Invalid value: ...: must be structural
```

Why this matters operationally: structural schema is what enables pruning, defaulting
(`default:` in the schema), the `status`/`scale` subresources, and server-side apply's field-manager
merge logic. A non-structural schema silently accepted fields the schema-writer never intended,
which is exactly the class of bug (typo'd field names accepted and silently ignored) that structural
schema was introduced to kill.

### Versioning and the conversion webhook trap

A CRD can serve multiple versions concurrently (`v1beta1` and `v1` above), but **only one version
is the storage version** (`storage: true`). Objects in etcd are always persisted in the storage
version's wire format. When a client requests a non-storage version, the apiserver must convert
on the fly.

For a single-version CRD, or versions that only differ by adding optional fields, Kubernetes can do
this conversion automatically (`strategy: None`). The moment your versions have actually incompatible
shapes — e.g. `spec.version: "14"` (a string) became `spec.postgresVersion: {major: 14, minor: 2}`
(a structured object) — automatic conversion cannot do the right thing, and you must supply a
**conversion webhook**.

```
kubectl get postgrescluster my-db -o yaml   (client wants v1beta1)
        |
        v
apiserver reads stored object (which is in v1, the storage version)
        |
        v
apiserver calls conversion webhook: "convert this v1 object to v1beta1"
        |
        v
webhook pod (deployment in db-operator-system namespace) applies conversion logic in Go
        |
        v
apiserver returns the converted object to kubectl
```

**The production gotcha**: the conversion webhook is not optional infrastructure you can let degrade.
If the webhook Service has no ready endpoints (crashlooping pod, wrong `caBundle`, network policy
blocking apiserver -> webhook traffic, or the webhook Deployment scaled to zero by an unrelated
change), then **every single read of that resource type in every non-storage version fails**,
cluster-wide, for every namespace, for every client, including `kubectl get`, `kubectl describe`,
controllers doing informer list/watch, and Prometheus scraping via `kube-state-metrics` if it uses
a different apiVersion than storage.

```
$ kubectl get postgresclusters -A
Error from server: conversion webhook for db.example.com/v1beta1, Kind=PostgresCluster failed:
Post "https://db-operator-conversion-webhook.db-operator-system.svc:443/convert?timeout=30s":
context deadline exceeded
```

Diagnosis steps in an incident:

```bash
# 1. Is the webhook Service backed by ready pods?
kubectl get endpoints db-operator-conversion-webhook -n db-operator-system

# 2. Are the webhook pods actually healthy?
kubectl get pods -n db-operator-system -l app=db-operator
kubectl logs -n db-operator-system deploy/db-operator --previous

# 3. Cross-check the caBundle on the CRD matches the webhook's serving cert
kubectl get crd postgresclusters.db.example.com -o jsonpath='{.spec.conversion.webhook.clientConfig.caBundle}' | base64 -d | openssl x509 -noout -dates

# 4. If clients only ever request the storage version, the bug is masked —
#    reproduce with the exact non-storage version:
kubectl get postgresclusters.v1beta1.db.example.com -A
```

**Mitigation that actually works under pressure**: if the webhook cannot be fixed quickly, patch the
CRD to make the previously-non-storage version the storage version temporarily (if all stored objects
are already convertible), or scale up/redeploy the webhook. There is no "disable conversion, fall back
to raw" switch — conversion is mandatory once declared. This is a strong argument for **not**
introducing incompatible schema changes across CRD versions unless you have solid webhook rollout
practices (readiness probes, PodDisruptionBudget, at least 2 replicas, and the webhook Deployment
living outside the blast radius of whatever the operator itself manages).

## 2. The operator pattern: control loops as codified operational knowledge

An operator is a controller (or set of controllers) that manages a CRD the way built-in controllers
(ReplicaSet controller, Deployment controller, Job controller) manage built-in resources. The core
unit of work is the **control loop**:

```
        +-------------------------------------------------------+
        |                                                        |
        v                                                        |
  +-----------+     +--------------+     +--------------+        |
  |  OBSERVE  | --> |     DIFF     | --> |     ACT      | -------+
  | (watch/   |     | (desired vs  |     | (API calls:  |
  |  list      |     |  current,    |     |  create/     |
  |  current   |     |  compute     |     |  update/     |
  |  state)    |     |  delta)      |     |  delete)     |
  +-----------+     +--------------+     +--------------+
```

Observe: the controller watches its CRD plus every child/related resource it manages (StatefulSets,
Services, Secrets, PVCs). Diff: it compares "what does the spec say should exist" against "what does
the cluster actually have" — not "what event just fired." Act: it issues the minimal set of API
calls needed to move current state toward desired state. Then it loops again, forever, regardless of
whether anything "happened."

### Why this encodes operational knowledge, concretely

Consider a PostgreSQL operator managing a 3-node cluster (1 primary, 2 replicas) that needs a rolling
restart to pick up a new `postgresql.conf` (say, `max_connections` changed). A human on-call engineer
doing this by hand knows a checklist that lives in their head or in a runbook:

1. Never restart the primary first — that forces a failover under load, mid-change.
2. Before touching any replica, check its replication lag (`pg_stat_replication.replay_lag` or
   the WAL receive/replay diff). If lag > 30s, wait or investigate — a lagging replica falling further
   behind during a restart risks becoming unusable as a failover target.
3. Restart replicas one at a time, waiting for each to rejoin and catch up before moving to the next.
4. Only after all replicas are healthy and caught up, consider restarting the primary — and only by
   first promoting a fully-caught-up replica, demoting the old primary to a replica, then restarting
   the demoted node.
5. Update the read/write Service selector to point at the new primary the instant promotion completes,
   not before (avoid a window where writes go to a node mid-promotion).

An operator's `Reconcile` function encodes exactly this sequence as Go code, run automatically every
time the desired config changes, every time — with no variance for engineer fatigue, no missed step
at 3 AM, no "the person on call this week doesn't know this database as well." That consistency, not
raw automation, is the actual value proposition. A naive automation (a script that just does a
rolling `kubectl rollout restart` on a StatefulSet) does not know any of this domain logic — it will
happily restart the primary first and cause an avoidable failover.

## 3. Building blocks

### client-go: Clientset, dynamic client, RESTMapper

- **Clientset** (`k8s.io/client-go/kubernetes`): strongly typed, generated per API group/version
  (`clientset.CoreV1().Pods("ns").Get(...)`). Compile-time safety, but only covers types it was
  generated for — built-ins plus anything you ran `client-gen` against.
- **Dynamic client** (`k8s.io/client-go/dynamic`): works against `unstructured.Unstructured` +
  `GroupVersionResource`, no generated Go types required. Essential for tools that must operate on
  arbitrary CRDs they don't compile against (`kubectl`, backup tools, GitOps controllers like Argo CD).

```go
gvr := schema.GroupVersionResource{Group: "db.example.com", Version: "v1", Resource: "postgresclusters"}
obj, err := dynamicClient.Resource(gvr).Namespace("prod").Get(ctx, "my-db", metav1.GetOptions{})
```

- **RESTMapper**: translates a `GroupVersionKind` (what you write in YAML: `kind: PostgresCluster`)
  into a `GroupVersionResource` (what the REST path actually needs: plural `postgresclusters`) and
  tells you whether the resource is namespaced or cluster-scoped. This mapping is not always
  mechanical (irregular plurals, `Endpoints` staying singular-looking) — it's discovered from the
  apiserver's discovery API and cached. `kubectl apply` and any generic tooling that accepts arbitrary
  manifests depends on a working RESTMapper.

### Informers and listers

A naive controller that calls `List` + `Get` on every reconcile against the live apiserver does not
scale: N controllers each doing their own polling multiplies apiserver load linearly with controller
count and reconcile frequency. **Informers** fix this:

```
apiserver  <---- one long-lived watch connection ---->  SharedInformer
                                                              |
                                                       local in-memory
                                                       cache (thread-safe
                                                       indexed store)
                                                              |
                                          +-------------------+-------------------+
                                          |                   |                   |
                                     Lister (read)      event handlers      event handlers
                                     used by            (Reconciler A)      (Reconciler B)
                                     Reconciler A
```

A `SharedInformerFactory` means multiple controllers/reconcilers watching the same GVK share **one**
watch connection and **one** cache, instead of each opening its own. This is the single biggest
reason large clusters remain usable with dozens of controllers running: apiserver sees one watch per
resource type per informer-sharing process, not one per controller.

- The informer maintains a local cache (a `cache.Indexer`) kept in sync via the watch stream, doing
  an initial `List` to establish a resource-version baseline, then applying `Added`/`Modified`/
  `Deleted` watch events incrementally.
- A **resync period** periodically re-enqueues every object from the local cache (not from the
  apiserver) as a synthetic "update" event, as insurance against a missed watch event silently leaving
  the controller in a stale state forever. This is a safety net, not the primary sync mechanism.
- A **Lister** is a read-only, cache-backed interface (`podLister.Pods("ns").Get("name")`) — it never
  hits the apiserver. Reconcile loops should read through listers, not direct client Gets, except when
  you specifically need a strongly consistent read (e.g. immediately after your own write).

### Workqueue: why you enqueue a key, not the object

The informer's event handlers do not call business logic directly. They push a **key**
(`namespace/name` string) onto a rate-limited workqueue (`k8s.io/client-go/util/workqueue`). Worker
goroutines pop keys, look the current object up fresh from the lister, and reconcile.

Why a key and not the object:

- By the time the worker pops the key and processes it, the object may have changed again (or been
  deleted) — re-fetching from the lister at process time guarantees you're reconciling against
  something close to current state, not a stale snapshot captured at enqueue time.
- Multiple events for the same object coalesce into a single queue entry (the workqueue de-duplicates
  pending keys), so a burst of 10 rapid updates to one object results in one reconcile of current
  state, not 10 reconciles of 10 stale intermediate states — this is a feature, not data loss, because
  reconcile is level-based (see section 4).
- Keys are cheap and comparable, which is what a de-duplicating set-like queue needs.

Requeue-with-backoff: `workqueue.RateLimitingInterface` exposes `AddRateLimited(key)` — on repeated
failures for the same key, backoff increases exponentially (capped), so a permanently-broken object
doesn't hot-loop the controller or hammer the apiserver; `Forget(key)` resets backoff once reconcile
succeeds.

### controller-runtime

`sigs.k8s.io/controller-runtime` is the library Kubebuilder and Operator SDK generate code against.
It wraps everything above so you rarely touch informers or workqueues directly:

- **Manager**: owns a shared cache (informers), a client (reads go through cache, writes go straight
  to the apiserver), leader election, metrics endpoint, health/readiness endpoints, and the set of
  registered controllers.
- **Reconciler interface**:

```go
type Reconciler interface {
    Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
}
```

`ctrl.Request` is exactly the namespace/name key described above — controller-runtime hides the
workqueue and informer wiring behind `Watches`/`Owns`/`For` calls at controller setup time:

```go
func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1.PostgresCluster{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

`Owns` sets up a watch on StatefulSets/Services and maps any change on an owned object back to the
owning PostgresCluster's key via owner references — so the reconciler also runs when someone
manually edits a child StatefulSet, not just when the CRD itself changes.

Leader election matters operationally: run the operator Deployment with 2+ replicas for availability,
but only the elected leader actively reconciles — followers sit idle holding a lease
(`coordination.k8s.io/Lease` object), ready to take over on leader crash/restart without a network
split-brain where two replicas act simultaneously.

### Kubebuilder / Operator SDK scaffolding

`kubebuilder init --domain example.com --repo github.com/you/db-operator` scaffolds the project
skeleton (`main.go` wiring up the Manager, `PROJECT` file, `Makefile`, `config/` Kustomize base).

`kubebuilder create api --group db --version v1 --kind PostgresCluster` generates:

- `api/v1/postgrescluster_types.go` — the Go struct that IS the source of truth for the CRD schema,
  annotated with **kubebuilder markers** that generate both the CRD YAML and deepcopy code:

```go
// +kubebuilder:validation:Enum=14;15;16
// +kubebuilder:default=16
Version string `json:"version"`

// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Maximum=9
Replicas int32 `json:"replicas"`

// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=".status.readyReplicas"
type PostgresCluster struct { ... }
```

  Running `make manifests` (which calls `controller-gen`) turns these markers into the
  `CustomResourceDefinition` YAML shown in section 1 — the Go type is the single source of truth, the
  YAML is generated output, not hand-maintained.

- `internal/controller/postgrescluster_controller.go` — the reconciler skeleton with an empty
  `Reconcile` and a `SetupWithManager`, plus **RBAC markers** that generate the ClusterRole the
  operator needs:

```go
// +kubebuilder:rbac:groups=db.example.com,resources=postgresclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=db.example.com,resources=postgresclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
```

  This matters for security review: the generated RBAC is exactly as broad as the markers declare —
  it's easy to over-grant (e.g. `verbs=*` on `secrets` cluster-wide) by copy-pasting markers without
  thinking about least privilege. Reviewing `config/rbac/role.yaml` after every `make manifests` run
  should be a standing habit, not a one-time setup step.

## 4. Reconcile loop design principles

This is where operator bugs actually come from in production. Almost every operator incident traces
back to a violation of one of the following.

### Idempotency

`Reconcile` will be called repeatedly for the same underlying state: on informer resync (every
`resyncPeriod`, default often 10h but frequently configured much shorter), on operator pod restart
(full cache rebuild replays every object as an "add"), on any unrelated field change that still maps
to the same key, and on requeue after error. Every action inside Reconcile must be safe to execute
more than once with no additional side effect the second time.

**Anti-pattern:**

```go
func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var pg dbv1.PostgresCluster
    r.Get(ctx, req.NamespacedName, &pg)

    sts := buildStatefulSet(&pg)
    if err := r.Create(ctx, sts); err != nil {
        return ctrl.Result{}, err   // second reconcile: AlreadyExists, returns error forever
    }
    return ctrl.Result{}, nil
}
```

The first reconcile creates the StatefulSet successfully. The very next reconcile (resync, or any
unrelated status update on the CRD triggering re-enqueue) calls `Create` again, gets a 409
`AlreadyExists`, returns an error, and the controller is now permanently unhealthy for that object —
backing off, retrying, failing, forever, even though the cluster is actually in the desired state.

**Correct pattern — get-or-create:**

```go
existing := &appsv1.StatefulSet{}
err := r.Get(ctx, client.ObjectKeyFromObject(desired), existing)
switch {
case apierrors.IsNotFound(err):
    if err := r.Create(ctx, desired); err != nil {
        return ctrl.Result{}, err
    }
case err != nil:
    return ctrl.Result{}, err
default:
    if !equality.Semantic.DeepEqual(existing.Spec, desired.Spec) {
        existing.Spec = desired.Spec
        if err := r.Update(ctx, existing); err != nil {
            return ctrl.Result{}, err
        }
    }
}
```

**Better pattern — Server-Side Apply**, which makes the operator declare the *entire* desired object
and lets the apiserver do the diff/merge against its own field-manager-tracked state, with `Create`
semantics on first apply and `Update`-if-changed semantics on every subsequent apply — no explicit
Get-then-branch needed:

```go
err := r.Patch(ctx, desired, client.Apply,
    client.ForceOwnership, client.FieldOwner("postgres-operator"))
```

### Level-based, not edge-based

The reconciler must compute "what should exist" vs "what currently exists" fresh, every single time,
from authoritative current state (the lister/cache, and live Gets for the CRD itself and its direct
children). It must not carry forward assumptions from "what the watch event told me changed" — because
watch events can be coalesced, missed during a disconnect/reconnect, or simply arrive out of order
relative to a controller restart.

**Concrete failure scenario for an edge-triggered design:** suppose (incorrectly) the reconciler is
written to only apply the *delta* implied by an `Update` event — e.g. "the event says
`spec.replicas` went from 3 to 5, so scale the StatefulSet up by 2":

```
t0: PostgresCluster spec.replicas: 3 -> 5     (edit #1)
t1: controller pod crashes before processing the watch event for edit #1
t2: controller pod restarts, informer does a fresh List — sees replicas=5 as the
    "initial state", no "diff" to compute because there's no prior event to
    compare against in an edge-triggered design that keeps per-object deltas
    in memory only
t3: the in-memory record of "last processed replicas" was lost with the crash;
    if the code's fallback logic assumes "no delta recorded means nothing to do",
    the StatefulSet is never scaled up. The CRD says 5, the StatefulSet still
    has 3, and nothing will ever fix it until the next unrelated spec edit
    coincidentally triggers a reconcile that happens to check total state.
```

A level-based reconciler has no such gap: every reconcile — no matter what triggered it, no matter
what the operator's memory of "previous state" was — reads current `spec.replicas` from the live
object and current StatefulSet replica count from the cache, diffs them, and corrects any mismatch.
Restart, resync, missed event, coalesced burst of edits: all converge to the same correct end state
because the loop never trusts anything except "what does the world look like right now."

### Partial failure handling

A single `Reconcile` invocation commonly needs to reconcile several sub-resources: a Secret, a
StatefulSet, a Service, maybe a PodDisruptionBudget. If updating the Secret succeeds and updating the
StatefulSet then fails (transient apiserver 5xx, admission webhook timeout, resource version
conflict), the function returns an error, controller-runtime requeues with backoff, and Reconcile runs
again from the top.

The design requirement this imposes: **each sub-resource's reconciliation must independently
check-then-act**, never assume "I already touched this in a previous pass so I can skip it now" and
never assume "nothing has been done yet so start from scratch." Structure it as a sequence of
independent idempotent steps, not a saga with implicit ordering state kept only in memory:

```go
if err := r.reconcileSecret(ctx, &pg); err != nil {
    return ctrl.Result{}, fmt.Errorf("reconciling secret: %w", err)
}
if err := r.reconcileStatefulSet(ctx, &pg); err != nil {
    return ctrl.Result{}, fmt.Errorf("reconciling statefulset: %w", err)
}
if err := r.reconcileService(ctx, &pg); err != nil {
    return ctrl.Result{}, fmt.Errorf("reconciling service: %w", err)
}
return ctrl.Result{}, r.updateStatus(ctx, &pg)
```

Each `reconcileX` function does its own Get-or-Create/diff-then-Update internally, so re-running the
whole sequence from the top after a mid-way failure is always correct — the earlier steps just find
"already matches desired state, nothing to do" and fall through instantly.

### Requeue strategies

- `return ctrl.Result{}, err` — an error causes controller-runtime's underlying workqueue to requeue
  automatically with **exponential backoff** (default base ~5ms up to a cap, per
  `workqueue.DefaultControllerRateLimiter`). Use this for anything unexpected: failed API calls,
  transient conflicts. Do not log-and-swallow errors just to avoid a backoff — that hides real
  problems from `kubectl describe` (no Warning events) and from metrics (`controller_runtime_
  reconcile_errors_total` stays flat while things are actually broken).
- `return ctrl.Result{Requeue: true}, nil` — immediate requeue, no backoff, no error recorded. Rare;
  usually used mid-migration between two controller-runtime versions or for genuinely-expected
  "there's more work, go around again right away" signals within the same reconcile chain.
- `return ctrl.Result{RequeueAfter: 12 * time.Hour}, nil` — nil error, scheduled future reconcile.
  This is the correct shape for expected polling that isn't an error condition: "certificate is valid
  for another 60 days, check again in 12h," or "database backup schedule says next check in 6h."
  Crucially this does not count as a failure and does not trigger backoff — it's a deliberate,
  successful "done for now, come back later" signal, which is exactly what cert-manager does for
  renewal checks (see section 5).

Mixing these up is a common bug: using `RequeueAfter` for something that is actually a transient
failure hides the failure from error-rate alerting; using bare error-return for expected periodic
polling means a permanently-non-erroring situation (like "still waiting for cert to approach
renewal window") looks the same, metrics-wise, as an actual persistent failure, defeating alerting
based on reconcile error rate.

### Status subresource, conditions, and observedGeneration

`.status` is served through a separate subresource endpoint (`/status`) when the CRD declares
`subresources: {status: {}}`. This exists so that:

- A user (or GitOps tool) doing `kubectl apply` against `.spec` and the controller writing `.status`
  do not race on the same optimistic-concurrency `resourceVersion` for unrelated fields — updating
  `.status` via the subresource does not bump `.metadata.generation` and vice versa, so a controller
  writing status every reconcile does not create update conflicts against a human editing spec, and
  RBAC can grant `update` on `postgresclusters/status` separately from `update` on
  `postgresclusters` (spec), so the operator's ServiceAccount can report status without being able to
  alter user intent, and users can't forge status.
- `.metadata.generation` increments only on a spec change (not on status-only updates, not on
  annotation/label-only updates). The convention every well-behaved controller follows is to record
  `status.observedGeneration = metadata.generation` as part of the same reconcile that acted on that
  spec version.

Conditions follow the standard shape (mirrored from `metav1.Condition`):

```yaml
status:
  observedGeneration: 7
  conditions:
    - type: Available
      status: "True"
      reason: AllReplicasReady
      message: "3/3 replicas ready, primary=pg-0"
      lastTransitionTime: "2026-07-18T09:12:03Z"
      observedGeneration: 7
    - type: Progressing
      status: "False"
      reason: ReconcileComplete
      message: "No changes pending"
      lastTransitionTime: "2026-07-18T09:12:03Z"
      observedGeneration: 7
```

Why `observedGeneration` matters to consumers: a human running `kubectl edit postgrescluster my-db`
and bumping `replicas: 3 -> 5` sees `metadata.generation` jump to 8 immediately. If
`status.observedGeneration` still reads 7, that is the signal "the controller has not yet even looked
at your latest change" — as opposed to "the controller looked at it and Available is still False
because it's genuinely still catching up." Automation (a CI pipeline waiting for a rollout, an
`kubectl wait --for=condition=Available`-style check) that ignores `observedGeneration` and only checks
`status.conditions[].status` can be fooled into declaring success based on a stale status snapshot
from before the spec change was even processed — a real class of race condition in GitOps pipelines
that poll status immediately after applying a change.

## 5. Real-world operators, concretely

### Prometheus Operator

CRDs: `ServiceMonitor`, `PodMonitor`, `Probe`, `PrometheusRule`, `Alertmanager`, `Prometheus`,
`AlertmanagerConfig`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-api
  namespace: payments
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: payments-api
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

The Prometheus Operator's reconcile loop watches every `ServiceMonitor`/`PodMonitor` matching the
`Prometheus` CR's `serviceMonitorSelector`, resolves each to concrete scrape targets via Endpoints/Pod
label matching, and regenerates the **entire Prometheus scrape configuration** as a single rendered
`prometheus.yml`, written into a Secret mounted into the Prometheus pod. It does not edit Prometheus's
config in place incrementally — every ServiceMonitor change causes a full config regeneration from
current state (level-based, exactly as section 4 describes), then triggers a config reload via
Prometheus's `/-/reload` HTTP endpoint (or a sidecar, `configmap-reload`, watching the mounted file).
`PrometheusRule` CRs are similarly aggregated into the alerting/recording rules file and reloaded the
same way. This is precisely why a malformed `PrometheusRule` (bad PromQL) can take down alerting
cluster-wide if the operator doesn't validate before merging — production Prometheus Operator versions
run an admission webhook that lints `PrometheusRule` PromQL at admission time specifically to prevent
this.

### cert-manager

CRDs: `Certificate`, `Issuer`/`ClusterIssuer`, `CertificateRequest`, `Order`, `Challenge`.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-example-com
  namespace: prod
spec:
  secretName: api-example-com-tls
  dnsNames: ["api.example.com"]
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  duration: 2160h      # 90d
  renewBefore: 360h    # 15d
```

The `Certificate` controller's reconcile loop is a clean example of `RequeueAfter`-driven polling
(section 4): each reconcile inspects the current `tls.crt` in `spec.secretName`, computes
`notAfter - renewBefore`, and if that time hasn't arrived, returns
`ctrl.Result{RequeueAfter: timeUntilRenewal}, nil` — an expected, non-error wait. Once inside the
renewal window, it creates a `CertificateRequest`, which a separate controller turns into an ACME
`Order`, which spawns a `Challenge` (HTTP01: cert-manager temporarily creates an Ingress/Pod serving
the ACME validation token at `/.well-known/acme-challenge/<token>`; DNS01: it creates a `TXT` record
via a DNS provider API and polls for propagation). On successful validation, the CA issues the cert,
cert-manager writes `tls.crt`/`tls.key`/`ca.crt` into the target Secret, and any workload mounting that
Secret (commonly via a reloader sidecar or a rolling restart triggered by a checksum annotation) picks
up the new cert. The entire multi-day, multi-step ACME dance is fully described by CRD status fields
at every stage — `kubectl describe certificate api-example-com` shows exactly which step is stuck
(e.g. `Challenge` stuck in `Pending` because the HTTP01 solver Ingress never got a public DNS record
pointed at it) instead of requiring you to read application logs.

### Database failover operators (Zalando/CrunchyData Postgres pattern)

Generic pattern common to production Postgres operators:

1. The operator (or a co-deployed component like Patroni, which many Postgres operators embed)
   maintains **leader election for the database itself**, distinct from the operator's own
   controller-runtime leader election — typically via a DCS (distributed configuration store: etcd,
   Kubernetes-native `Lease`/`ConfigMap` objects, or Consul) that all Postgres nodes and the operator
   watch.
2. On primary failure (missed heartbeats past a threshold), the DCS lease expires, remaining replicas
   race to acquire it; the one with the least replication lag / most advanced WAL position wins and is
   promoted (`pg_promote()` or `patronictl`).
3. The operator watches for the promotion event and immediately updates the **Service selector** (or
   an Endpoints/EndpointSlice object it manages directly, bypassing the normal Service selector
   mechanism for faster convergence) so that `postgres-primary.prod.svc` now routes to the newly
   promoted pod. This is the step that actually restores write availability to applications — DNS/
   Service-level redirection, not a client-side reconnect-and-retry that happens to land elsewhere.
4. Schema migrations across replicas are handled by only ever running DDL against the primary
   (enforced by the operator refusing to run a migration Job against anything but the pod currently
   labeled primary) and letting streaming replication propagate the DDL to replicas — the operator
   explicitly does not attempt to run migrations independently on each pod, which would diverge
   schema state across the cluster.

The operational knowledge embedded here — "don't promote a lagging replica," "flip the Service
selector only after promotion is confirmed, not speculatively," "migrations only ever touch the
primary" — is exactly the kind of judgment call that used to live in a DBA's runbook and now executes
deterministically without a human paged at 3 AM for a routine failover.

## 6. When to build a custom operator vs when it's overkill

Decision framework, in order of what to check first:

**Build an operator when all of these are true:**

- There is a genuine multi-step **state machine** with conditional branching based on live cluster
  state (failover, cert rotation with external ACME challenges, backup/restore with point-in-time
  recovery, blue/green schema migrations) — not just "apply these five manifests in order."
- The logic needs to **react continuously** to drift and external state changes (replication lag
  changing, a cert nearing expiry, a pod becoming unready) rather than run once on a human-triggered
  event.
- The same operational sequence **recurs** across many instances/clusters/tenants, so codifying it
  once amortizes real engineering cost across many future incidents avoided.
- You need the CRD-as-API benefit specifically: users declare desired state and never touch the
  imperative steps (`kubectl apply -f postgrescluster.yaml`, not "SSH in and run this 12-step
  runbook").

**Don't build one — a Helm chart, a Job/CronJob, or a plain script is sufficient — when:**

- The task is genuinely "apply this fixed set of manifests," full stop, with no ongoing reactive
  behavior. A Helm chart's templating covers "same manifests, different values per environment" without
  needing a control loop at all.
- Periodic-but-simple maintenance (nightly backup trigger, log rotation, cert renewal handled by an
  existing tool rather than custom logic) fits a `CronJob` with a plain script. Do not build a
  control loop to replace `0 2 * * *`.
- Config rarely changes and doesn't need continuous drift-correction — a one-off `kubectl apply`, or
  a GitOps tool (Argo CD/Flux) reconciling plain manifests, already gives you the "make cluster match
  Git" loop without you writing any Go.

**The maintenance cost is real and often under-priced during the "let's build an operator" decision:**

- You now own a **Go codebase** with its own dependency updates (client-go/controller-runtime version
  bumps are not always trivial — API changes across major versions, CRD conversion needs when you
  change your own types), its own test suite (envtest/kind-based integration tests, which are slower
  and flakier than unit tests), and its own release/versioning process independent of the workloads it
  manages.
- You now own a new, potentially broad **RBAC surface** — a ClusterRole with `create/update/delete` on
  Secrets, StatefulSets, Services cluster-wide is a substantial blast radius if the operator's
  ServiceAccount token or the operator pod itself is ever compromised.
- You now own a genuinely new **failure mode that didn't exist before**: a buggy reconcile loop that
  misdiagnoses "desired vs current" (e.g. a diff function with a bug that always sees a mismatch) will
  hot-loop, issuing writes continuously, at the workqueue's max backoff-free rate, hammering the
  apiserver and potentially the child resources themselves (repeatedly rolling a StatefulSet that the
  operator incorrectly believes is out of sync). This is measurably worse than having no automation at
  all — no automation fails passively; a buggy automated control loop fails *actively and repeatedly*,
  often faster than a human could notice and intervene. `client-go`'s workqueue rate limiting and
  controller-runtime's leader election reduce but do not eliminate this risk — a bad diff function
  still issues bad writes, just at a rate-limited pace instead of unboundedly.

The honest summary of the trade-off: an operator converts "a human has to remember and correctly
execute a runbook" into "a piece of software has to correctly implement that runbook forever, across
every future Kubernetes and dependency version, with someone maintaining it." That's worth it exactly
when the runbook is complex, recurring, and reactive enough that the software's correctness cost is
lower than the aggregate human-error cost of running it manually — and not before.
