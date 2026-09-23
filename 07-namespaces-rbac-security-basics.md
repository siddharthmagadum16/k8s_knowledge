# Namespaces, RBAC, and Security Basics

## Namespaces

A namespace is a virtual partition inside a single cluster. It provides **name scoping** and a boundary for **access control** and **resource quotas** — it is not a security boundary by itself (no network isolation, no kernel-level isolation) unless you additionally apply NetworkPolicies and RBAC.

```bash
kubectl create namespace payments
kubectl get namespaces
kubectl config set-context --current --namespace=payments
```

### What namespaces are for

- **Name scoping**: two Deployments both named `api` can coexist if they're in different namespaces. Fully-qualified DNS for a Service reflects this: `api.payments.svc.cluster.local`.
- **Access control boundary**: RBAC Roles are namespaced — you can grant a team access to `namespace: payments` without touching `namespace: billing`.
- **Resource quota boundary**: `ResourceQuota` and `LimitRange` objects apply per-namespace.
- **Blast radius**: deleting a namespace deletes everything inside it. `kubectl delete namespace payments` is one of the most destructive commands in Kubernetes — it cascades to every namespaced object in it with no undo.

### Cluster-scoped vs namespaced resources

Every API resource is either namespaced or cluster-scoped. Check with:

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

Common **cluster-scoped** resources (exist once, globally, not inside any namespace):
- `Node`
- `Namespace` itself
- `PersistentVolume` (PV — note: `PersistentVolumeClaim` IS namespaced)
- `ClusterRole`, `ClusterRoleBinding`
- `StorageClass`
- `CustomResourceDefinition`
- `PodSecurityPolicy` (deprecated/removed in 1.25+)

Common **namespaced** resources:
- `Pod`, `Deployment`, `ReplicaSet`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`
- `Service`, `Ingress`, `Endpoints`
- `ConfigMap`, `Secret`
- `Role`, `RoleBinding`
- `PersistentVolumeClaim`
- `ServiceAccount`

Default namespaces present in every cluster: `default`, `kube-system` (control plane and system components — kube-dns/CoreDNS, kube-proxy pods, CNI daemonsets), `kube-public` (readable by all, including unauthenticated users — used for cluster info), `kube-node-lease` (Node heartbeat lease objects, used for faster node failure detection than the old NodeStatus-update method).

### Practical namespace hygiene

- Namespace-per-environment (`dev`, `staging`, `prod`) inside one cluster is common but weaker than physically separate clusters for prod isolation — a misconfigured RBAC rule or a compromised token in `dev` can, at best, be blocked by RBAC from touching `prod`; at worst (overly broad ClusterRole), it cannot.
- Namespace-per-team/service is the more common pattern for multi-tenant clusters (`payments`, `billing`, `search`).
- Always pair namespaces with a `ResourceQuota` (cap total CPU/memory/object counts) — without it, one namespace can starve the whole cluster's resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

## ServiceAccounts

A ServiceAccount (SA) is an identity for **processes running inside pods** to authenticate to the Kubernetes API server — distinct from a User identity (which represents a human, and is not a first-class API object in Kubernetes at all; Users come from an external auth mechanism like a cloud IAM integration or client certs).

```bash
kubectl get serviceaccounts
kubectl create serviceaccount payments-worker
```

### Default SA behavior

Every namespace has a `default` ServiceAccount, automatically created. Every pod that doesn't explicitly set `spec.serviceAccountName` uses it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: payments-worker   # explicit — best practice
  containers:
  - name: app
    image: myapp:1.4
```

**Security implication**: if you never set `serviceAccountName`, every pod in that namespace runs as the `default` SA, and if that SA (via a RoleBinding) has been granted any permissions, every pod in the namespace inherits them — including pods you didn't intend to grant API access to. Best practice: create a dedicated, minimally-scoped SA per workload, and explicitly disable auto-mounting for pods that never call the API server.

### Token mounting

Historically (pre-1.24), every pod automatically got a long-lived SA token mounted at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace
```

This token was a static Secret object with no expiry — a major risk if exfiltrated (e.g., via SSRF or a container breakout), since it worked forever until manually deleted.

Since Kubernetes 1.24: SA tokens are **projected volumes** with the `BoundServiceAccountTokenVolume` feature (default on) — tokens are:
- Time-bound (default 1 hour expiry, auto-rotated by kubelet before expiry).
- Audience-bound (scoped to the API server audience by default, can be scoped further).
- Bound to the specific pod (invalidated if the pod is deleted, via the TokenRequest API).

```yaml
spec:
  serviceAccountName: payments-worker
  automountServiceAccountToken: false   # disable if pod never talks to the API server
```

Set `automountServiceAccountToken: false` on the ServiceAccount or the Pod spec for any workload that doesn't call `kubectl`/the K8s API from inside the container — this removes an entire class of credential-theft risk for the majority of workloads (e.g., a plain web app that only talks to its own database has no business holding an API server token).

### How pods authenticate to the API server

1. kubelet mounts the projected SA token into the pod at pod creation.
2. The app (or a tool like `kubectl` running inside the pod, or a client library like `client-go`'s in-cluster config) reads the token and sends it as a Bearer token: `Authorization: Bearer <token>`.
3. The API server validates the token (signature + expiry + audience), extracting the identity `system:serviceaccount:<namespace>:<sa-name>`.
4. RBAC is then evaluated for that identity against the requested verb/resource.

```bash
# from inside a pod
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -sS --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/payments/pods
```

## RBAC fundamentals

RBAC (Role-Based Access Control) answers: "can identity X perform verb V on resource R in apiGroup G (optionally: named resource N), in namespace S?" It is purely additive — there is no explicit "deny" rule type in core RBAC; access is the union of everything granted, default-deny for anything not explicitly granted.

### Role vs ClusterRole

- **Role**: namespaced. Grants permissions only within the namespace it's created in.
- **ClusterRole**: cluster-scoped. Can grant permissions either (a) across all namespaces when bound with a ClusterRoleBinding, (b) within a single namespace when bound with a RoleBinding that references it, or (c) to cluster-scoped resources (Nodes, PVs) which by definition can only be granted via a ClusterRole (a plain Role cannot grant access to cluster-scoped resources at all).

### RoleBinding vs ClusterRoleBinding

- **RoleBinding**: namespaced. Binds a Role (or a ClusterRole) to subjects (Users, Groups, or ServiceAccounts), granting the permissions only within that RoleBinding's namespace.
- **ClusterRoleBinding**: cluster-scoped. Binds a ClusterRole to subjects, granting the permissions cluster-wide, across every namespace.

Key nuance: a ClusterRole is a reusable permission template. Whether its scope ends up cluster-wide or namespace-local depends entirely on whether it's bound via ClusterRoleBinding or RoleBinding. This lets you define one `pod-reader` ClusterRole and bind it narrowly in one namespace via RoleBinding while binding it cluster-wide elsewhere via ClusterRoleBinding — avoiding duplicated Role definitions.

**Role vs RoleBinding, in one line: a Role is just a list of permissions, inert on its own — it grants nothing until a RoleBinding maps it to an actual identity.** A `RoleBinding` is that map — `subjects` (who) on one side, `roleRef` (which Role/ClusterRole) on the other. `subjects` is a list, and each entry can be one of three kinds: `ServiceAccount` (for pods/workloads), `User` (a human, not a real K8s API object — identity comes from external auth like a client cert or OIDC token), or `Group` (also external-auth-derived, a set of users). One RoleBinding can even mix kinds in its `subjects` list, granting the same Role's permissions to a ServiceAccount, a User, and a Group all at once.

### Anatomy of a rule

```yaml
rules:
- apiGroups: [""]                      # "" = core API group (Pods, Services, ConfigMaps, Secrets...)
  resources: ["pods", "pods/log"]      # resource types (subresources use "resource/subresource")
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["delete"]
  resourceNames: ["specific-pod-name"]  # restrict to a named object, rare in practice
```

- `apiGroups`: `""` for core (v1) resources; `"apps"` for Deployments/StatefulSets/DaemonSets/ReplicaSets; `"batch"` for Jobs/CronJobs; `"rbac.authorization.k8s.io"` for Roles/Bindings themselves, etc. Find the right group with `kubectl api-resources`.
- `resources`: plural lowercase resource name, as shown by `kubectl api-resources`.
- `verbs`: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`. `list`/`watch` are distinct from `get` — a Role granting only `get` cannot `list` (you'd need to know exact object names).

### Full worked example

Goal: a CI/CD ServiceAccount in namespace `payments` that can deploy (create/update Deployments) and read logs, but cannot touch Secrets or delete namespaces.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: payments
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: payments
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: payments
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: payments
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

Notice: no rule mentions `secrets` — this SA cannot read, list, or create Secrets in `payments`, even though it can read ConfigMaps. There is no rule for `namespaces` at all — `Role`s can't grant access to a cluster-scoped resource like Namespace regardless; that would require a ClusterRole.

**Cluster-wide read-only auditor example** (ClusterRole + ClusterRoleBinding):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-auditor
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-auditor-binding
subjects:
- kind: Group
  name: "platform-team"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-auditor
  apiGroup: rbac.authorization.k8s.io
```

Note that `get`/`list`/`watch` on `secrets` with `resources: ["*"]` means this auditor group **can read every Secret in the cluster** — read access is still access; "read-only" RBAC is not automatically safe.

### Checking effective permissions

```bash
kubectl auth can-i create deployments --namespace payments
kubectl auth can-i delete pods --as system:serviceaccount:payments:ci-deployer -n payments
kubectl auth can-i --list --as system:serviceaccount:payments:ci-deployer -n payments
```

### Built-in ClusterRoles

Kubernetes ships aggregated default ClusterRoles: `cluster-admin` (full access, do not bind broadly), `admin` (full access within a namespace, including Roles/RoleBindings, but not quota/namespace itself), `edit` (read/write to most objects in a namespace, not Roles/RoleBindings/quota), `view` (read-only, excludes Secrets in recent versions). Prefer binding these over inventing new equivalents when they fit.

## Pod Security Standards

The Pod Security Standards (PSS) replaced the deprecated PodSecurityPolicy (removed in 1.25). They define three profiles, enforced via the built-in **Pod Security Admission** controller, applied per-namespace via labels:

```bash
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Modes: `enforce` (reject non-compliant pods), `audit` (allow but log a violation to the audit log), `warn` (allow but return a warning to the client, e.g., visible in `kubectl apply` output).

Profiles, least to most restrictive:

- **Privileged**: no restrictions at all. Default if no label is set. Appropriate only for system/infra namespaces (`kube-system` CNI, CSI driver pods needing host access).
- **Baseline**: blocks known privilege-escalation paths while staying broadly compatible: disallows `privileged: true` containers, disallows host namespaces (`hostNetwork`, `hostPID`, `hostIPC`), disallows most `hostPath` volumes, restricts dangerous `capabilities` additions (e.g., `NET_ADMIN`, `SYS_ADMIN`), restricts `hostPort`.
- **Restricted**: current pod-hardening best practice, on top of Baseline: requires `runAsNonRoot: true`, disallows privilege escalation (`allowPrivilegeEscalation: false`), requires dropping `ALL` capabilities (only `NET_BIND_SERVICE` may be re-added), requires a defined `seccompProfile` (`RuntimeDefault` or `Localhost`), disallows setting `runAsUser: 0`.

### securityContext fields

Two levels: `spec.securityContext` (pod-level defaults) and `spec.containers[].securityContext` (per-container, overrides pod-level).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001            # group ownership applied to mounted volumes
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.4
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]   # only if binding to port <1024
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

- **runAsNonRoot: true**: kubelet refuses to start the container if the image's default user (or `runAsUser`) resolves to UID 0. Does not by itself set the UID — pair with `runAsUser`.
- **runAsUser / runAsGroup**: explicit UID/GID, overrides whatever the image's Dockerfile `USER` sets.
- **readOnlyRootFilesystem: true**: mounts the container's root filesystem read-only. Forces you to explicitly mount writable `emptyDir` volumes for any directory the app needs to write to (`/tmp`, cache dirs, upload staging) — this is a strong containment control: even a full RCE in the app can't persist a backdoor to the image filesystem.
- **allowPrivilegeEscalation: false**: blocks `setuid`/`setgid` binaries and any syscall path that could grant more privileges than the parent process had (must be `false` for the `no_new_privs` kernel flag to be set).
- **capabilities.drop: ["ALL"]** then selectively `add`: default container capabilities include things like `NET_RAW` (raw socket access, usable for packet sniffing/spoofing) that almost no application needs. Dropping ALL and adding back only what's required (e.g., `NET_BIND_SERVICE` to bind ports <1024 without running as root) is the standard hardening pattern.
- **fsGroup**: sets the GID for mounted volumes so a non-root container can still read/write them (relevant for CSI-backed PVCs and `emptyDir`).
- **seccompProfile.type: RuntimeDefault**: applies the container runtime's default seccomp filter, blocking a large set of rarely-needed and historically-exploited syscalls.
- `privileged: true` (avoid): grants the container essentially all host capabilities and device access — equivalent to root on the host. Reserved for specific infra pods (CNI plugins, some CSI/monitoring agents) that genuinely require host-level access; never for application workloads.

## Secrets and RBAC — who can read them

Secrets are base64-encoded (not encrypted by default at the API/etcd level unless you've configured [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)) — base64 is an encoding, not a security control. Anyone with `get`/`list` RBAC access to `secrets` in a namespace can trivially decode them: `kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d`.

Practical guidance:

- Never grant `resources: ["*"]` or a rule listing `secrets` alongside broad verbs unless the subject genuinely needs it — read-only ClusterRoles for auditors/monitoring should explicitly exclude `secrets`, since the built-in `view` ClusterRole intentionally does this (recent Kubernetes versions strip Secret read access from `view`).
- Any ServiceAccount whose token could be exfiltrated (e.g., mounted in a pod with a large network-facing attack surface) should have no RBAC access to `secrets` at all if it doesn't need it — check with `kubectl auth can-i get secrets --as system:serviceaccount:ns:sa-name -n ns`.
- Scope Secret access with `resourceNames` where possible instead of granting access to all Secrets in a namespace:

```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-creds"]
  verbs: ["get"]
```

- Consider an external secrets manager (Vault, cloud KMS-backed secret stores via the External Secrets Operator or CSI Secret Store driver) for anything sensitive — Kubernetes-native Secrets are convenient but weakest on encryption-at-rest and audit trail unless you've explicitly hardened both.
- Enable etcd encryption at rest (`EncryptionConfiguration` on the API server) in any cluster holding real secrets — without it, anyone with etcd access (backups included) can read every Secret in plaintext-equivalent (base64) form directly from the datastore, bypassing RBAC entirely.
