# Pods and Workload Controllers

## 1. Pods

### 1.1 What a Pod is and why not bare containers

A **Pod** is the smallest deployable unit in Kubernetes — not a single container. A Pod is a group of one or more containers that:

- **Share a network namespace**: one IP address per Pod, all containers in it see `localhost` as each other and share the same port space. Two containers in the same Pod cannot both bind port 8080.
- **Can share storage volumes**: volumes defined at the Pod level can be mounted into multiple containers within it.
- **Are scheduled together**: always land on the same node, start/stop as a unit as far as scheduling is concerned.
- **Share a lifecycle boundary**: the Pod exists as long as at least one of its containers is expected to run; it's the unit the scheduler and node-eviction logic operate on.

Kubernetes never schedules a bare container because real applications frequently need tightly coupled helper processes (log shippers, proxies, config reloaders) that must share network/storage with the main process, and modeling that as "container X always co-located with container Y" needs a first-class grouping — that's the Pod.

Internally, the container runtime implements this by creating a hidden **pause/infra container** first, which holds the network namespace; every other container in the Pod joins that namespace.

### 1.2 Pod spec anatomy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders-api-demo
  namespace: default
  labels:
    app: orders-api
    tier: backend
spec:
  restartPolicy: Always          # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30
  initContainers:
  - name: init-db-check
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z orders-db 5432; do sleep 2; done']
  containers:
  - name: api
    image: myorg/orders-api:1.4.2
    ports:
    - containerPort: 8080
    env:
    - name: LOG_LEVEL
      value: "info"
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
    volumeMounts:
    - name: cache
      mountPath: /var/cache/app
  - name: log-shipper
    image: myorg/fluent-bit-sidecar:2.1
    volumeMounts:
    - name: cache
      mountPath: /var/cache/app
      readOnly: true
  volumes:
  - name: cache
    emptyDir: {}
```

Key fields to internalize: `restartPolicy` applies to the whole Pod's containers uniformly, not per-container. `resources.requests` drives scheduling; `resources.limits` drives runtime enforcement (cgroups) — see the resource management doc for the full mechanics.

### 1.3 Multi-container Pods and the sidecar pattern

Reasons to put more than one container in a Pod (versus separate Pods/Deployments):

- **Sidecar**: a helper container augmenting the main container — a log/metrics shipper, a service-mesh proxy (Envoy in Istio/Linkerd), a config-reloader watching a mounted ConfigMap and signaling the main process. As of Kubernetes 1.28+, sidecars can be declared as a special class of `initContainers` with `restartPolicy: Always`, so they start before the main container and Kubernetes keeps them running and shuts them down last — solving the historical race where a sidecar container in a Job's Pod could keep the Pod alive after the main container finished.
- **Ambassador**: a proxy container that the main container talks to via `localhost`, which then forwards to external services (abstracts service discovery/retry logic away from the app).
- **Adapter**: normalizes output from the main container into a standard format for a monitoring pipeline (e.g., converting a custom metrics format into Prometheus exposition format).

You do **not** put unrelated services in the same Pod (e.g., a web app and a completely separate database) — those scale independently and should be separate Deployments/Pods.

### 1.4 Init containers

Init containers run **sequentially, to completion, before any app container starts**. Each must exit `0` before the next runs; if one fails, the kubelet retries it according to `restartPolicy` (Pod stays `Pending`/`Init` until they succeed, unless `restartPolicy: Never`, in which case the Pod is marked `Failed`).

Use cases:
- Wait for a dependency to become available (DB migrations complete, another service reachable).
- Populate a shared volume with data/config before the main container starts (e.g., cloning a git repo into an `emptyDir`).
- Run one-time setup with different privileges/image than the main container (e.g., a `chmod`/`chown` init container running as root while the app container runs unprivileged).

### 1.5 Pod lifecycle phases

`status.phase` (coarse, Pod-level):

| Phase | Meaning |
|---|---|
| `Pending` | Pod accepted by the API server but not all containers are running yet — could be waiting on scheduling, image pull, or init containers. |
| `Running` | Pod bound to a node and at least one container is running (or starting/restarting). |
| `Succeeded` | All containers terminated successfully (exit 0) and will not restart — normal for Jobs. |
| `Failed` | All containers terminated, and at least one terminated in failure (non-zero exit) and won't restart. |
| `Unknown` | The Pod's state can't be determined, typically because the node hosting it is unreachable (kubelet not reporting). |

Container-level states (`status.containerStatuses[].state`), finer grained:

| State | Meaning |
|---|---|
| `Waiting` | Container not yet running — pulling image, waiting on init containers, or in `CrashLoopBackOff` between restart attempts. `reason` field shows detail. |
| `Running` | Container process is executing; `startedAt` timestamp recorded. |
| `Terminated` | Container ran and stopped (or was killed); `exitCode`, `reason` (`Completed`, `Error`, `OOMKilled`), and `finishedAt` recorded. |

`CrashLoopBackOff` is a `Waiting` reason, not a phase — the Pod phase may still show `Running` (if other containers are up) while one container cycles crash → backoff → restart with exponential backoff (10s, 20s, 40s... capped at 5 min).

### 1.6 Pod restart semantics

`restartPolicy` governs whether kubelet restarts a container after it exits, for **all containers in the Pod**:
- `Always` (default): used for long-running services (Deployments always force this).
- `OnFailure`: restart only on non-zero exit — typical for Jobs that should retry on failure but not loop forever on success.
- `Never`: never restart — Jobs that want exactly one attempt per Pod (retries handled by creating new Pods instead, governed by `backoffLimit`).

## 2. ReplicaSets and Deployments

### 2.1 ReplicaSet

A ReplicaSet ensures a specified number of Pod replicas matching a label selector are running at all times. It watches Pods; if the count drops below `spec.replicas` (Pod deleted, node died), it creates more; if there are extras (e.g., stray Pod with matching labels created manually), it deletes them.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: orders-api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
      - name: api
        image: myorg/orders-api:1.4.2
```

You almost never create ReplicaSets directly — **Deployments manage ReplicaSets for you**, adding rollout/rollback semantics on top.

### 2.2 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # extra pods allowed above `replicas` during rollout
      maxUnavailable: 0    # pods allowed to be unavailable during rollout
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
      - name: api
        image: myorg/orders-api:1.4.2
        readinessProbe:
          httpGet: { path: /ready, port: 8080 }
```

**How a rolling update works internally**: changing `spec.template` (e.g., a new image tag) causes the Deployment controller to:
1. Create a **new ReplicaSet** with the updated Pod template (a hash of the template becomes part of its name, e.g. `orders-api-7d9f8b6c9d`).
2. Incrementally scale the new ReplicaSet up and the old one down, respecting `maxSurge` (how many Pods above desired count are allowed temporarily) and `maxUnavailable` (how many Pods below desired count are tolerated), waiting for new Pods to pass their readiness probe before proceeding further.
3. Once the new ReplicaSet is fully scaled and old Pods are drained, the old ReplicaSet is scaled to 0 (but kept around, not deleted, up to `revisionHistoryLimit`, to enable rollback).

```bash
kubectl set image deployment/orders-api api=myorg/orders-api:1.5.0
kubectl rollout status deployment/orders-api
kubectl rollout history deployment/orders-api
kubectl rollout undo deployment/orders-api                 # rollback to previous revision
kubectl rollout undo deployment/orders-api --to-revision=2  # rollback to specific revision
kubectl rollout pause deployment/orders-api                 # stop mid-rollout, batch further edits
kubectl rollout resume deployment/orders-api
```

Each ReplicaSet under a Deployment corresponds to one revision. `revisionHistoryLimit` controls how many old (scaled-to-0) ReplicaSets are retained for rollback — beyond that, they're garbage collected.

Other strategy: `type: Recreate` — kills all old Pods before creating new ones (downtime, but avoids two versions running simultaneously — used when the new version can't coexist with the old, e.g. incompatible schema).

## 3. StatefulSets

Deployments assume Pods are interchangeable, disposable, and identity-less — fine for stateless web tiers. Databases, queues, and clustered systems (etcd, Kafka, Elasticsearch, PostgreSQL replicas) need:

- **Stable, predictable network identity**: each replica keeps the same hostname across restarts/rescheduling.
- **Stable storage**: each replica's disk persists and reattaches to the *same* replica identity after a restart, not to a randomly chosen one.
- **Ordered deployment and scaling**: replica 0 comes up before replica 1, and scale-down happens in reverse order, so cluster join/leave sequencing is predictable.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: orders-db
spec:
  serviceName: orders-db-headless    # must point at a headless Service
  replicas: 3
  podManagementPolicy: OrderedReady  # default; Parallel skips ordering
  selector:
    matchLabels:
      app: orders-db
  template:
    metadata:
      labels:
        app: orders-db
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
          name: db
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: orders-db-headless
spec:
  clusterIP: None       # headless — no load-balancing VIP, DNS returns each Pod IP
  selector:
    app: orders-db
  ports:
  - port: 5432
```

**Stable identity mechanics**:
- Pods are named deterministically: `orders-db-0`, `orders-db-1`, `orders-db-2` (not random suffixes like a ReplicaSet's `orders-api-7d9f8b6c9d-x7k2p`).
- Each gets a stable DNS record via the headless Service: `orders-db-0.orders-db-headless.default.svc.cluster.local`, resolvable even as the Pod is rescheduled to a different node/IP — this is the hostname `orders-api`'s init container and app config point at.
- **`volumeClaimTemplates`** creates one PersistentVolumeClaim per replica (`data-orders-db-0`, `data-orders-db-1`, ...); when `orders-db-1`'s Pod is deleted and recreated (e.g., after a node failure), the StatefulSet controller recreates a Pod with the same name and re-attaches the *same* PVC — replica 1 always gets its own disk back, never replica 0's.
- **Ordering**: on creation, `orders-db-0` must reach Running+Ready before `orders-db-1` is created, and so on (unless `podManagementPolicy: Parallel`). On scale-down, the highest ordinal is terminated first.
- Deleting a StatefulSet does **not** delete its PVCs by default — this is a deliberate safety choice (protects data); you delete the PVCs explicitly if you truly want to reclaim the storage.

Use cases: PostgreSQL/MySQL replica sets, Kafka brokers, Zookeeper, Elasticsearch/OpenSearch data nodes, Cassandra, any system with per-instance persistent identity/state.

## 4. DaemonSets

A DaemonSet ensures **exactly one copy of a Pod runs on every node** (or every node matching a selector), automatically adding a Pod when a new node joins and removing it when a node is removed — you don't set a `replicas` count at all; it's implicitly "one per matching node."

Unlike the Deployment/StatefulSet above, a DaemonSet is cluster-wide infrastructure, not something you'd deploy per-service — the same `node-log-collector` below picks up stdout/stderr from every pod on its node, `orders-api` and `orders-db` included, without either of them needing their own logging sidecar.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - operator: Exists          # tolerate all taints, so it runs even on control-plane/tainted nodes
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests: { cpu: "50m", memory: "64Mi" }
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

Use cases:
- **Log collection agents** (Fluent Bit, Fluentd, Filebeat) reading `/var/log` from every node's host filesystem via `hostPath`.
- **Node monitoring/metrics agents** (node-exporter for Prometheus, Datadog agent) needing per-node system metrics (CPU, disk, network at the host level, not container level).
- **CNI/networking agents** (Calico's `calico-node`, Cilium agent) that must run on every node to program that node's networking rules.
- **Storage agents/CSI node plugins** that mount volumes for pods on their local node.

Update strategy is similar to Deployments: `RollingUpdate` (default, replaces DaemonSet Pods node by node with `maxUnavailable` control) or `OnDelete` (only replaced when you manually delete the old Pod).

## 5. Jobs and CronJobs

### 5.1 Job

A Job runs Pods to completion — for batch/one-off work, not long-running services.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: orders-api-db-migration
spec:
  completions: 1        # total successful completions needed
  parallelism: 1         # how many pods run concurrently
  backoffLimit: 4        # retries before marking Job failed
  activeDeadlineSeconds: 600   # overall time limit for the whole Job
  template:
    spec:
      restartPolicy: OnFailure   # Never or OnFailure only — Always is invalid for Jobs
      containers:
      - name: migrate
        image: myorg/orders-api-migrate:1.5.0   # run before rolling out orders-api:1.5.0
        command: ["./migrate.sh"]
```

- `restartPolicy` must be `OnFailure` or `Never` (not `Always` — a Job's whole point is to eventually complete, not run forever).
  - `OnFailure`: kubelet restarts the **same Pod's container** in place on failure.
  - `Never`: kubelet does not restart the container; instead the **Job controller creates a brand-new Pod** to retry (useful when you want a clean container state per attempt, and it's what you see counted against `backoffLimit`).
- `completions`/`parallelism` patterns:
  - `completions: 1, parallelism: 1` (default): run once.
  - `completions: 5, parallelism: 1`: run 5 Pods to completion, one at a time (sequential batch).
  - `completions: 5, parallelism: 3`: run up to 3 concurrently until 5 total succeed (parallel batch with a cap).
  - Omit `completions`, set `parallelism: 3` with a **work queue** pattern: Pods pull work from an external queue and exit when the queue is empty; the Job is done when any Pod exits successfully and you signal completion externally (less common; `Indexed` completion mode is the modern alternative for parallel-with-partitioning, giving each Pod a `JOB_COMPLETION_INDEX` env var).
- `backoffLimit`: number of retries before the Job is marked `Failed` outright (with exponential backoff between retries, capped at 6 minutes).

```bash
kubectl get jobs
kubectl logs job/orders-api-db-migration
kubectl delete job orders-api-db-migration     # jobs are not auto-deleted; clean up manually or via ttlSecondsAfterFinished
```

Add `spec.ttlSecondsAfterFinished: 3600` to auto-delete the Job (and its Pods) some time after completion, avoiding manual cleanup accumulation.

### 5.2 CronJob

A CronJob creates Jobs on a schedule, same syntax as Unix cron.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: orders-db-backup
spec:
  schedule: "0 2 * * *"          # 2 AM daily, in the kube-controller-manager's configured timezone (UTC by default unless spec.timeZone is set)
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  startingDeadlineSeconds: 200    # how late a missed run can start before being skipped
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: myorg/orders-db-backup:1.0
            command: ["./backup.sh"]
```

- `concurrencyPolicy`:
  - `Allow` (default): multiple Job runs can overlap if a previous one is still running when the next is due.
  - `Forbid`: skip the new run entirely if the previous is still active.
  - `Replace`: kill the currently running Job's Pods and start the new one.
- `startingDeadlineSeconds`: if the CronJob controller (or apiserver) was down and missed the scheduled time by more than this, the run is skipped rather than fired late — important for jobs where a stale run is worse than a skipped one.
- History limits keep a small number of completed/failed Job objects around for debugging (`kubectl get jobs`, `kubectl logs job/<name>-<timestamp>`), rest are garbage collected.

```bash
kubectl create job manual-run --from=cronjob/orders-db-backup   # trigger an ad-hoc run outside the schedule
kubectl get cronjobs
```

## 6. Choosing the Right Workload Controller

| Need | Controller |
|---|---|
| Stateless, interchangeable replicas, rolling updates | Deployment |
| Stable per-replica identity + storage (databases, clustered stateful systems) | StatefulSet |
| Exactly one Pod per node (agents, collectors, CNI) | DaemonSet |
| Run-to-completion, batch work, possibly parallel | Job |
| Scheduled recurring batch work | CronJob |
| Directly creating bare Pods | Rare — only for one-off debugging (`kubectl run`); production workloads always go through a controller so failures are healed |
