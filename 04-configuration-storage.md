# Configuration and Storage

## 1. ConfigMaps

A ConfigMap holds non-sensitive configuration data as key-value pairs, decoupling configuration from container images so the same image can run in dev/staging/prod with different config.

### 1.1 Creation methods

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# From a file (key = filename, value = file contents)
kubectl create configmap nginx-conf --from-file=nginx.conf

# From an entire directory (one key per file)
kubectl create configmap app-configs --from-file=./config-dir/

# From a .env-style file (each line becomes a key=value entry)
kubectl create configmap app-env --from-env-file=app.env
```

Or declaratively:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  app.properties: |
    server.port=8080
    server.timeout=30s
binaryData:
  logo.png: <base64-encoded-bytes>   # for non-UTF8 data
```

### 1.2 Consuming as environment variables

```yaml
containers:
- name: app
  envFrom:
  - configMapRef:
      name: app-config          # Option A: load-entire: every key becomes an env var
  env:
  - name: LOG_LEVEL              # Option B: load-speific: select individual keys explicitly
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  - name: SENTRY_DSN              # can do, but not recommend to hardcode values
    value: '82384892302.sentry.com/project/vault'
```

### 1.3 Consuming as a mounted volume

```yaml
containers:
- name: app
  volumeMounts:
  - name: config-vol
    mountPath: /etc/app-config     # each key appears as a file here
volumes:
- name: config-vol
  configMap:
    name: app-config
    items:                          # optional: project only specific keys
    - key: app.properties
      path: app.properties
```

### 1.4 Update behavior and limitations

- **Env var injection is a one-time snapshot at Pod creation.** Updating the ConfigMap does **not** update already-injected environment variables in running Pods — the Pod must be restarted (new Pod created) to pick up changes. This is a common gotcha: `kubectl edit configmap app-config` followed by "why didn't my app pick up the new value" — because it's mounted as env vars.
- **Volume-mounted ConfigMaps are updated live** (with a propagation delay — controlled by kubelet's sync period, typically up to ~1 minute, longer if using the local cache-based kubelet config) — the files inside the mounted directory change in place via a kubelet-managed atomic symlink swap. The application itself must notice the file changed (inotify watch, polling, or a reload signal) — Kubernetes doesn't restart or signal the process automatically.
- Exception: if `subPath` is used to mount a single file out of a ConfigMap (rather than the whole ConfigMap as a directory), that file **does not** get live-updated — subPath mounts are a static bind-mount snapshotted at pod start. This is a frequent trap when people only want one file, not the whole directory.
- A ConfigMap can be marked `immutable: true` — once set, its `data`/`binaryData` cannot be changed (only delete+recreate). This is a performance/safety optimization: kubelet doesn't need to watch immutable ConfigMaps for changes, reducing apiserver watch load, and it prevents accidental in-place edits to config that should be versioned by name instead (e.g., `app-config-v2`).
- Max size for a ConfigMap (and Secret) is ~1MiB, since they're stored as single etcd objects.

## 2. Secrets

Secrets are structurally identical to ConfigMaps (key-value store, same consumption patterns) but intended for sensitive data, with some different defaults and handling.

### 2.1 How Secrets differ from ConfigMaps

- **Base64 encoding is not encryption.** `data` fields in a Secret are base64-encoded, purely so the API can carry arbitrary binary content in JSON — anyone with API read access to the Secret (or a copy of the etcd data, or the YAML) can trivially decode it (`echo <value> | base64 -d`). Base64 provides zero confidentiality.
- **Encryption at rest** is a separate, optional cluster-level feature: without configuring an `EncryptionConfiguration` for the apiserver, Secrets are stored in etcd in plaintext (just base64 in the JSON blob, unencrypted on disk). To actually protect Secret data at rest, the cluster admin must configure encryption providers (`aescbc`, `secretbox`, or an external KMS plugin like AWS KMS/GCP Cloud KMS/Vault) so the apiserver encrypts Secret values before writing to etcd. Managed platforms increasingly enable this by default; self-hosted clusters often do not unless explicitly configured.
- Secrets get some operational protections ConfigMaps don't: kubelet avoids writing Secret data into container logs/events by default, and by default Secret volumes are mounted as `tmpfs` (in-memory, not persisted to node disk) to reduce residual-data risk.
- RBAC is typically stricter on Secrets in practice (least-privilege — most Pods shouldn't have `get`/`list` on arbitrary Secrets).

### 2.2 Secret types

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque              # generic key-value, the default/most common type
data:
  username: YWRtaW4=       # base64("admin")
  password: cGFzc3dvcmQxMjM=
```

Other built-in types with special handling/validation:
- `kubernetes.io/tls` — requires `tls.crt` and `tls.key` keys; used by Ingress TLS termination.
- `kubernetes.io/dockerconfigjson` — holds container registry credentials (`.dockerconfigjson`), referenced via `imagePullSecrets` on a Pod/ServiceAccount to pull images from private registries.
- `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth` — structured credential types for specific auth schemes.
- `kubernetes.io/service-account-token` — auto-generated (or requested via the `TokenRequest` API) tokens used by Pods to authenticate to the apiserver as their ServiceAccount.

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=password123

kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key

kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=deployer \
  --docker-password=$REGISTRY_TOKEN
```

### 2.3 Consuming Secrets

```yaml
containers:
- name: app
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
  volumeMounts:
  - name: creds
    mountPath: /etc/secrets
    readOnly: true
volumes:
- name: creds
  secret:
    secretName: db-credentials
    defaultMode: 0400          # restrict file permissions inside the pod
imagePullSecrets:
- name: regcred
```

Same env-var-vs-volume update caveats apply as ConfigMaps (env vars are a snapshot; volume mounts update live with a delay, subPath mounts don't update).

For genuinely sensitive production secrets, many teams avoid native Kubernetes Secrets as the source of truth entirely and instead use **External Secrets Operator**, **Vault Agent Injector**, or cloud-native secret managers (AWS Secrets Manager, GCP Secret Manager) that sync into Kubernetes Secrets at runtime or inject directly — reducing the blast radius of anyone with etcd/API read access, and enabling rotation without manual re-encoding.

## 3. Ephemeral Volumes: emptyDir and hostPath

Containers are ephemeral by nature — their writable layer disappears when the container is removed, and even within a Pod's lifetime, a container restart wipes its own filesystem changes. Volumes attached at the Pod level solve two distinct problems: sharing files between containers in a Pod, and (for the persistent kinds, section 4) surviving Pod deletion/rescheduling entirely.

### 3.1 emptyDir

```yaml
volumes:
- name: scratch
  emptyDir: {}
    # sizeLimit: 1Gi
    # medium: Memory    # backs it with tmpfs (RAM) instead of node disk — fast, but consumes node memory and is lost on node reboot
```

- Created empty when the Pod is assigned to a node; **exists only as long as that Pod exists on that node** — deleted permanently when the Pod is removed (including on a container crash-restart within the same Pod, the volume survives; on Pod deletion/rescheduling, it does not).
- Shared between all containers in the Pod that mount it — the primary way sidecars exchange data with the main container (e.g., a main container writing logs, a sidecar tailing and shipping them from the same `emptyDir`).
- Use cases: scratch space for sorting/computation, a cache that's fine to lose, a handoff buffer between an init container and the main container (e.g., init container clones a repo into it, main container serves from it).

### 3.2 hostPath

```yaml
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket
```

- Mounts a path from the **node's own filesystem** directly into the Pod. Data persists on that node's disk independent of the Pod's lifecycle, but is tied to that specific node — if the Pod is rescheduled elsewhere, it sees a different (likely empty) path on the new node.
- Legitimate uses: DaemonSets that need node-level access (log collectors reading `/var/log`, monitoring agents reading `/proc` or `/sys`, CNI/CSI node plugins needing access to host device/socket paths).
- Risky/discouraged for regular application storage: breaks the node-agnostic scheduling model, and depending on `type` and path, can be a serious security hole (mounting `/` or the Docker socket effectively grants host-level access/root-equivalent capability from inside a container) — Pod Security Standards' "restricted"/"baseline" profiles disallow or restrict `hostPath` for this reason.

### 3.3 Why ephemeral volumes exist at all

Not every workload needs durability. Forcing every volume through the full PV/PVC/StorageClass provisioning machinery (below) would be wasteful for a sidecar's scratch buffer or a build cache. Ephemeral volumes are cheap, fast, tied to Pod lifecycle, and require no external storage backend or provisioning latency.

## 4. PersistentVolumes and PersistentVolumeClaims

For data that must outlive a specific Pod (and often a specific node) — databases, uploaded files, message queue logs — Kubernetes decouples **storage provisioning** from **storage consumption** via two objects.

- **PersistentVolume (PV)**: a cluster-level resource representing an actual piece of storage (an AWS EBS volume, a GCE PD, an NFS export, a Ceph RBD image, etc.) — provisioned either manually by an admin or dynamically (see StorageClasses below). Exists independently of any Pod or namespace.
- **PersistentVolumeClaim (PVC)**: a namespaced request for storage made by a user/application ("I need 10Gi, ReadWriteOnce, from the `fast-ssd` class") — analogous to a Pod requesting CPU/memory, but for storage.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myorg/app:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

### 4.1 Binding lifecycle

1. A PVC is created, describing size/accessMode/class requirements.
2. The **PV controller** watches for unbound PVCs and unbound PVs; it binds the PVC to a matching existing PV (manual/static provisioning), **or** if the PVC references a `StorageClass` with a provisioner, the corresponding **CSI driver's external-provisioner** dynamically creates a brand-new PV (and the underlying cloud disk) sized to match, then binds it.
3. Once bound, the PVC↔PV relationship is exclusive and 1:1 — no other PVC can claim that PV while bound.
4. A Pod referencing the PVC by name gets the underlying volume mounted wherever it's scheduled — the CSI node plugin on that node attaches/mounts the actual disk.
5. On PVC deletion, what happens to the PV is governed by its **reclaim policy** (below).

`kubectl get pv,pvc` shows `STATUS` transitioning `Pending` → `Bound`, and for PVs also `Available` (unbound, ready) and `Released` (was bound, PVC deleted, not yet reclaimed).

### 4.2 Access modes

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Mountable read-write by a single node at a time (most block storage: EBS, GCE PD, Azure Disk). Note: as of newer Kubernetes versions, RWO can allow multiple **Pods on the same node** to mount it simultaneously — it's a per-node restriction, not strictly per-Pod. |
| `ReadOnlyMany` (ROX) | Mountable read-only by many nodes simultaneously — useful for shared reference data/config distributed via a shared filesystem. |
| `ReadWriteMany` (RWX) | Mountable read-write by many nodes simultaneously — requires a genuinely shared filesystem backend (NFS, CephFS, EFS, Azure Files, Filestore) — block storage (EBS/GCE PD/Azure Disk) fundamentally cannot do this. |
| `ReadWriteOncePod` (RWOP) | Newer, stricter mode: guarantees the volume is mounted by at most a single **Pod** in the whole cluster (closes the RWO same-node-multiple-pods loophole) — for workloads that must have exclusive single-writer guarantees (e.g., some databases). |

**Note: how can `ROX`/`RWX` let many nodes mount the same volume, if each node has its own physically separate disk?** They can't, if the backing storage is local disk — that's exactly the constraint. `ROX`/`RWX` only work with **network-attached/shared filesystem storage** (NFS, EFS, Azure Files, CephFS, Filestore), never with block storage like EBS/GCE PD/local SSD which is local for pod (node-level). The data physically lives on a separate storage server/service reachable over the network; every node mounts that *same remote export* over the network protocol.

A database replica typically wants RWO (each replica has its own exclusive disk, as modeled by StatefulSet's `volumeClaimTemplates` — see below); a shared upload directory read by many web-tier Pods wants RWX.

### 4.3 Reclaim policies

Set on the PV (`spec.persistentVolumeReclaimPolicy`), determines what happens to the underlying storage once its PVC is deleted:

- **Retain**: the PV and underlying storage are kept, marked `Released` — not automatically reusable by a new PVC (needs manual admin intervention: manually delete/recreate the PV object or clean and repurpose it). Safest default for anything with valuable data — accidental PVC deletion doesn't destroy the actual disk/data.
- **Delete** (default for most dynamically provisioned PVs): the PV object **and the underlying cloud storage resource** (the actual EBS volume, etc.) are deleted automatically when the PVC is deleted. Convenient for ephemeral/dev workloads, dangerous for production data unless you're certain you want automatic teardown.
- **Recycle** (deprecated/removed in current Kubernetes): used to do a basic `rm -rf` scrub and make the PV available again — replaced by dynamic provisioning patterns entirely.

### 4.4 PV/PVC vs "just use a database"

PersistentVolumes aren't a competitor to databases — they're the storage layer *underneath* one. A self-hosted database (e.g. the `orders-db` Postgres StatefulSet) needs somewhere physical to write its data files/WAL/indexes; that's exactly what the PV provides. The database software is what turns that raw storage into something queryable (SQL, transactions, indexing) — without the PV, it has nowhere to persist anything.

Where each actually applies:
- **Managed database** (RDS, DynamoDB): runs entirely outside the cluster on the provider's infra — no PV/PVC/StatefulSet involved at all. Usually the right call for structured app data (orders, users) when one fits your budget/compliance needs.
- **Self-hosted stateful software** (a database you run yourself, Kafka, Elasticsearch, Redis with persistence): needs a PV, same pattern as `orders-db` — there's no way around it if you're not using a managed equivalent.
- **Large files/blobs** (uploads, ML artifacts, backups): usually better served by object storage (S3/GCS) than either a PV or a database — cheaper, infinitely scalable, no attach/detach semantics. Reach for a PV here only when you specifically need real POSIX filesystem paths, not an HTTP API.

## 5. StorageClasses and Dynamic Provisioning

A `StorageClass` is a named "profile" of storage that lets PVCs request storage dynamically without an admin having to hand-provision a PV up front for every claim.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # which CSI driver handles this class
parameters:
  type: gp3
  iops: "4000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

- `provisioner`: identifies which CSI driver services requests for this class (e.g., `ebs.csi.aws.com`, `pd.csi.storage.gke.io`, `disk.csi.azure.com`, `rook-ceph.rbd.csi.ceph.com`).
- `parameters`: driver-specific knobs (disk type, IOPS, filesystem type, encryption key ID, replication factor, etc.).
- `volumeBindingMode`:
  - `Immediate`: PV is provisioned as soon as the PVC is created, before any Pod is scheduled — risk of provisioning storage in a zone/region the eventually-scheduled Pod's node can't reach (block storage is usually zone-locked).
  - `WaitForFirstConsumer` (recommended for zonal block storage): delays provisioning until a Pod referencing the PVC is actually scheduled, so the PV is created in the same zone as the chosen node — avoids the classic "PVC bound to a volume in us-east-1a but the Pod got scheduled in us-east-1b" failure.
- `allowVolumeExpansion: true`: permits growing a PVC's `requests.storage` later (`kubectl edit pvc` / patch) without recreating it — the underlying CSI driver must support resize; filesystem expansion inside the Pod may need a Pod restart depending on the driver/filesystem.
- A cluster typically has one **default** StorageClass (annotated `storageclass.kubernetes.io/is-default-class: "true"`) used automatically when a PVC omits `storageClassName`.

### 5.1 CSI (Container Storage Interface)

CSI is to storage what CRI is to container runtimes: a standard gRPC interface decoupling Kubernetes core from any specific storage backend's implementation details. Before CSI, storage plugins ("in-tree" volume plugins) had to be compiled directly into Kubernetes core binaries — every new storage backend required a Kubernetes release. CSI moved this out-of-tree: each vendor ships their own CSI driver (a set of Pods, typically a DaemonSet for the node plugin + a Deployment for the controller plugin) that Kubernetes talks to over a well-defined API, independent of core Kubernetes release cycles.

A CSI driver deployment typically has two parts:
- **Controller plugin**: handles cluster-level operations — create/delete volume (talking to the cloud/storage API), attach/detach to a node, snapshot/clone, resize.
- **Node plugin** (usually a DaemonSet, since it must run wherever a Pod using that storage might land): handles node-level operations — mount/unmount the already-attached volume into the Pod's filesystem namespace, format if needed.

Nearly every storage backend you'll encounter in production Kubernetes today (AWS EBS/EFS, GCE PD/Filestore, Azure Disk/Files, Ceph/Rook, Portworx, OpenEBS, Longhorn, vSphere) is consumed through a CSI driver.

## 6. volumeClaimTemplates in StatefulSets — Storage Perspective

Revisiting from the workloads doc, purely from the storage angle: `volumeClaimTemplates` in a StatefulSet is a template for generating **one distinct PVC per replica ordinal**, automatically, rather than you hand-creating N PVCs.

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 20Gi
```

For a StatefulSet named `postgres` with 3 replicas, this generates PVCs `data-postgres-0`, `data-postgres-1`, `data-postgres-2` — each dynamically provisioned (via the referenced StorageClass) as its Pod is first created. The binding is durable across the Pod's lifecycle: if `postgres-1`'s Pod is deleted and the StatefulSet recreates it (same node or a different one, since these are RWO block volumes that reattach wherever the Pod lands), it mounts `data-postgres-1` again — not a randomly picked PVC. This is precisely how each stateful replica keeps its own independent, durable dataset across restarts and rescheduling, which is the entire reason StatefulSets exist for anything backed by per-replica disks (each Kafka broker's log segments, each Postgres replica's data directory, etc.).

Scaling a StatefulSet down does **not** delete its PVCs (by default) — scaling back up reattaches the existing PVC for that ordinal rather than provisioning fresh empty storage, which is exactly the safety behavior you want (you don't want to accidentally lose replica 2's data because you scaled down to 2 replicas and back up).
