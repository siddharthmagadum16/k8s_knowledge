# Security Hardening

## 1. Pod Security Admission (PSA)

PodSecurityPolicy was removed in 1.25. PSA is a built-in admission controller driven entirely by namespace labels — no separate policy objects, no RBAC binding to a policy resource. This is a downgrade in flexibility (no per-workload exceptions without a separate mutating step) but a huge simplification in failure mode: you can't accidentally leave a cluster with zero policy because no PSP matched, which was the classic PSP footgun.

### The three levels

**Privileged** — no restrictions. Exists so you can explicitly opt namespaces (kube-system, CNI/CSI daemonset namespaces, ingress controllers that need host networking) out of policy rather than silently exempting them.

**Baseline** — blocks known privilege escalations, stays compatible with most off-the-shelf workloads. Specifically it rejects:

- `spec.hostNetwork`, `spec.hostPID`, `spec.hostIPC` set to true
- `spec.containers[*].securityContext.privileged: true`
- `spec.containers[*].securityContext.capabilities.add` containing anything outside an allowed set (it blocks `SYS_ADMIN`, `NET_ADMIN`, `SYS_PTRACE`, etc. — full disallow list is in the k8s PSA spec)
- `hostPath` volumes are still allowed under Baseline — this trips people up, they assume Baseline is safe against node filesystem access
- `spec.containers[*].securityContext.procMount` other than `Default`
- `spec.containers[*].securityContext.allowPrivilegeEscalation` is NOT restricted at Baseline — only Restricted enforces this
- AppArmor/seccomp annotations, if present, must not be `Unconfined`
- Sysctls are limited to a safe allowlist (`kernel.shm*`, `net.ipv4.ip_local_port_range`, etc.)

**Restricted** — the hardened tier, everything Baseline blocks plus:

- `runAsNonRoot: true` required (at pod or container level)
- `allowPrivilegeEscalation: false` required on every container
- `capabilities.drop: ["ALL"]` required; only `NET_BIND_SERVICE` may be re-added
- `seccompProfile.type` must be `RuntimeDefault` or `Localhost` (`Unconfined` rejected, and unset is rejected too — must be explicit)
- volume types restricted to a safe list — `hostPath`, `gcePersistentDisk`-as-raw-block, and other node-touching volume types are rejected outright (Baseline allows hostPath, Restricted does not)
- `runAsUser` cannot be set to `0` anywhere it's specified

Restricted is what you want for application namespaces. Nothing that legitimately needs host access should live there.

### Enforcement is a namespace label, with three independent modes

Each namespace can set a level for each of three modes independently:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.30
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.30
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.30
```

`enforce` actually rejects the pod at admission. `audit` writes a policy-violation annotation into the audit log without blocking. `warn` returns a client-visible warning (shows up in `kubectl apply` output) without blocking.

### Why set audit/warn stricter than enforce during migration

You inherit a namespace running Baseline workloads and want to move it to Restricted without an outage. Set:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Now nothing is blocked (enforce stays at baseline, existing deploys keep working), but every `kubectl apply`/`create` that would fail Restricted prints a warning to the person applying it, and every existing pod that violates Restricted shows up in the audit log on a schedule (via periodic pod audit, not just at admission — the audit annotation is attached whenever the pod object is evaluated). You grep the audit log for `pod-security.kubernetes.io/audit-violations` over a week, fix the offending workloads one by one, then flip `enforce` to `restricted` once the audit log is clean. This is the only safe way to raise the bar on a namespace nobody has a full inventory of.

### Staged rollout across an environment

```yaml
# Stage 1: dev namespace — go straight to restricted, low risk
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
# Stage 2: staging — observe before enforcing
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
---
# Stage 3: prod — same observation window, longer soak
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

A cluster-wide default can also be set via the `PodSecurity` admission plugin's config (`--admission-control-config-file`), giving every unlabeled namespace a default of `baseline` so nothing lands at `privileged` by omission.

## 2. securityContext deep dive

### runAsNonRoot — the gotcha that bites almost everyone

`runAsNonRoot: true` doesn't set a UID. It tells the kubelet "after resolving the effective UID, reject the pod if that UID is 0." The effective UID resolution order is: container's `securityContext.runAsUser` → pod's `securityContext.runAsUser` → the image's `USER` directive (from the Dockerfile) → root if nothing set.

If you set `runAsNonRoot: true` but don't set `runAsUser`, and the image has no `USER` line (the vast majority of base images, including most official language images before they added non-root variants), the resolved UID is 0 and the pod is rejected at admission with something like:

```
container has runAsNonRoot and image will run as root
```

This looks like a policy failure but is actually a scheduling-time UID resolution failure — it happens even without PSA, purely from the container runtime/kubelet check. The fix is always to pin the UID explicitly, not rely on the image:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
```

Never trust "the image probably doesn't run as root" — verify with `docker run --rm <image> id` before shipping the manifest.

### readOnlyRootFilesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

This mounts the container's root filesystem read-only. Anything the process needs to write — `/tmp`, `/var/cache/nginx`, `/run`, a PID file directory, an app-specific writable path — must get an explicit volume, or the process crashes on first write with `EROFS`. This is the single most effective mitigation against a large class of container escape and persistence techniques: an attacker with RCE in the app can't drop a webshell, can't modify a binary, can't write a cron job into the container's filesystem for persistence.

```yaml
containers:
- name: app
  securityContext:
    readOnlyRootFilesystem: true
  volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: cache
    mountPath: /var/cache/nginx
  - name: run
    mountPath: /var/run
volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir: {}
- name: run
  emptyDir: {}
```

Test this by running the container locally with `--read-only --tmpfs /tmp` before committing to the manifest — half the CVE-fixing app rebuilds fail this in CI because nobody checked where the app actually writes (session files, npm/pip caches invoked at runtime, etc).

### allowPrivilegeEscalation: false

Sets `no_new_privs` on the process via prctl. This blocks setuid/setgid binaries from elevating privilege on exec — even if a setuid-root binary exists in the image (e.g. `ping`, `sudo`, some package leftover), it cannot use its setuid bit once `no_new_privs` is set. This is independent of `privileged` and independent of capabilities — it's a kernel-level exec-time restriction. Restricted PSA requires this to be `false` on every container; there is essentially no legitimate reason to run with this `true` unless you specifically need a setuid binary to function (rare, and a smell if you find yourself needing it).

### Capabilities: drop ALL, add back only what's needed

Default Linux capability set granted to containers (via containerd/CRI-O defaults) is already reduced from full root capabilities, but it still includes things like `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `NET_RAW` — enough for a compromised process to do meaningful damage (NET_RAW alone enables raw socket crafting for spoofing/sniffing).

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
```

`NET_BIND_SERVICE` is the canonical "add back" case: it lets a non-root process bind to ports < 1024 (e.g. port 80/443) without running as UID 0. This replaces the old anti-pattern of running the whole container as root just so nginx could bind port 80. Any other capability should be justified per-workload — `SYS_PTRACE` for a debugger sidecar, `NET_ADMIN` for something manipulating iptables/routes inside its own netns — and reviewed, because each added capability is effectively an exception carved into an otherwise-audited baseline.

### seccompProfile

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

`RuntimeDefault` uses the container runtime's shipped seccomp profile (containerd/CRI-O ship one modeled on Docker's default), which blocks roughly 44 syscalls that have no business being called from inside a container — `mount`, `umount2`, `reboot`, `kexec_load`, `ptrace` (partially), `keyctl`, `add_key`, `clock_settime`, `swapon`/`swapoff`, `iopl`/`ioperm`, `syslog` (kernel ring buffer read). Applications never call these directly in normal operation — they're avenues for container escape, kernel exploitation, or host tampering, not application functionality.

`Localhost` with a `localhostProfile` path points at a custom, workload-specific JSON profile that goes further than the runtime default — see section 3.

### Why `privileged: true` is dangerous

`privileged: true` disables essentially all container isolation: the container gets every Linux capability, seccomp filtering is disabled, AppArmor/SELinux confinement is disabled, and it gets access to all host devices under `/dev` (not just a curated cgroup device allowlist). A privileged container can mount the host's root filesystem (`mount /dev/sda1 /mnt`, or more directly `nsenter` into the host's mount namespace using access it already has), load kernel modules, reconfigure host networking, and read/write host memory devices. In practice, privileged means "root on the node," full stop — a workload requesting `privileged: true` should be treated as equivalent to requesting node-level SSH access, and reviewed with that level of scrutiny. Legitimate uses are narrow: some CNI plugins, some storage drivers, and low-level monitoring agents (and even most of those have moved to precise `capabilities.add` lists instead of blanket `privileged`).

### Full production-grade securityContext

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      securityContext:               # pod-level defaults
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        image: registry.internal/api:1.4.2
        securityContext:             # container-level overrides win
          runAsNonRoot: true
          runAsUser: 10001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/app
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

## 3. seccomp and AppArmor

### seccomp

Seccomp (secure computing mode) is a kernel feature enforced via a BPF filter attached to a process, evaluated on every syscall entry, before the syscall executes. It's not a userspace policy — the kernel itself refuses the syscall if the filter says so, so even a fully compromised process with arbitrary code execution cannot make a blocked syscall, short of a kernel bug in the seccomp/BPF path itself.

A profile is a JSON document: a `defaultAction` applied to any syscall not explicitly listed, plus a list of syscalls with per-syscall actions.

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "access", "arch_prctl", "bind", "brk",
        "clone", "close", "connect", "epoll_create1", "epoll_ctl",
        "epoll_wait", "execve", "exit", "exit_group", "fcntl",
        "fstat", "futex", "getcwd", "getdents64", "getpid",
        "getsockname", "getsockopt", "listen", "mmap", "mprotect",
        "munmap", "nanosleep", "openat", "poll", "prctl",
        "pread64", "pwrite64", "read", "readlink", "recvfrom",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn", "sched_yield",
        "set_robust_list", "setsockopt", "sigaltstack", "socket",
        "stat", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

`SCMP_ACT_ERRNO` as default means anything not on the allowlist returns an error to the calling process (usually `EPERM`) rather than killing it outright (`SCMP_ACT_KILL` is stricter but breaks apps that probe for a syscall and gracefully fall back — errno is generally the safer default action for compatibility).

Authoring workflow in practice: run the workload under `strace -f -c` (or use `runc`'s seccomp trace mode / `docker run --security-opt seccomp=unconfined` + tracing) against a realistic load and test suite to capture every syscall it actually makes, generate the allowlist from that trace, then run the workload again with the draft profile under load — including failure paths, restarts, and any admin/debug endpoints — because syscalls only exercised on cold start or error paths are easy to miss and will surface as a production outage under load, not in a smoke test.

Custom profiles are distributed as files on every node (not as Kubernetes objects) at:

```
/var/lib/kubelet/seccomp/profiles/my-app.json
```

They need to be pushed to every node that can schedule the pod — via a DaemonSet init step, a node bootstrap script, or baked into the node image. Reference it from the pod spec:

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/my-app.json
```

The path is relative to `/var/lib/kubelet/seccomp/`. If the file is missing on the node the pod schedules to, the pod fails to start with a clear error — always deploy the profile file before deploying the workload, and keep them in version control alongside the manifest referencing them.

### AppArmor

AppArmor is Mandatory Access Control (MAC) — unlike seccomp (which restricts syscalls), AppArmor restricts what a process can do with those syscalls: which file paths it can read/write/execute, which capabilities it can use, network access at a coarse level, and mount/ptrace restrictions by target. A profile is a named policy loaded into the kernel on the node, and pods opt in by name.

Example profile fragment (loaded via `apparmor_parser` on the node, typically from `/etc/apparmor.d/`):

```
#include <tunables/global>

profile k8s-api-restricted flags=(attach_disconnected) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,

  /app/** r,
  /app/bin/api ix,
  /tmp/** rw,

  deny /proc/sys/kernel/** wklx,
  deny /sys/** wklx,
  deny mount,
  deny ptrace,
}
```

Load it on the node and reference it from the pod (1.30+ uses the securityContext field; older clusters use the `container.apparmor.security.beta.kubernetes.io/<container-name>` annotation):

```yaml
securityContext:
  appArmorProfile:
    type: Localhost
    localhostProfile: k8s-api-restricted
```

Caveat that matters operationally: AppArmor is a Linux Security Module and its availability depends on the node's distro. Ubuntu and Debian ship it enabled by default — it's the natural choice on GKE (Ubuntu/COS nodes) and most Ubuntu-based EKS/AKS node pools. RHEL, CentOS, Amazon Linux 2/2023, and Fedora derivatives ship SELinux instead and typically don't have AppArmor available at all (`/sys/module/apparmor` won't exist). Trying to schedule an AppArmor-annotated pod onto a node without AppArmor support fails at admission with the node reporting the profile unavailable. If your node pool is mixed-OS, treat this as a per-nodepool decision, not a cluster-wide policy, or use SELinux-native controls (`seLinuxOptions` in securityContext) on the RHEL-family nodes instead.

## 4. NetworkPolicy deep patterns

Without any NetworkPolicy, all pods can reach all other pods across all namespaces — this is the default CNI behavior on flat networks (Calico, Cilium, etc. all default-allow until a policy selects a pod). NetworkPolicies are additive/allow-only once any policy selects a pod — the moment a pod is selected by at least one NetworkPolicy for a given direction (ingress/egress), that direction becomes default-deny except for what's explicitly allowed, across all policies that select it.

### Default-deny-all baseline

An empty `podSelector: {}` matches every pod in the namespace. No `ingress`/`egress` list means no traffic is allowed in that direction.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Apply this to every application namespace as the floor. Everything else in this section is an additive allow carved out of this baseline.

### Explicit allow: DNS egress

Without this, default-deny-all breaks every pod's ability to resolve DNS, which breaks essentially everything, including calls to explicitly allowed external endpoints (an IP-based egress rule survives, but any app doing hostname lookups does not).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Explicit allow: ingress from same namespace only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

An unqualified `podSelector: {}` inside a `from` block means "any pod in the same namespace as this policy" — it does not reach across namespaces unless paired with a `namespaceSelector`. This is the correct default for a shared/multi-tenant cluster: a pod in `payments` cannot be hit by a pod in `marketing` unless a separate rule explicitly allows it.

### Namespace isolation via label, for controlled cross-namespace traffic

If `payments` needs to accept traffic only from a specific `checkout` namespace (not all namespaces, not even all of `payments`'s neighbors):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-checkout-only
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payments-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: checkout
```

Label the source namespace explicitly rather than relying on its name — `namespaceSelector` matches on namespace *labels*, not the namespace name field, so `kubectl label namespace checkout team=checkout` has to be done deliberately, and it's a common miss when people expect `namespaceSelector: {name: checkout}` syntax to work (it doesn't; only `matchLabels`/`matchExpressions` against actual labels).

### Egress restriction against exfiltration

The realistic threat model: an app has an RCE (dependency CVE, deserialization bug), the attacker's shell is inside the pod, and now they try to `curl` to their own server to pull a second-stage payload or `POST` stolen data out. A default-deny egress policy with a narrow allowlist for only the endpoints the app legitimately needs (its database, a specific third-party API by IP/CIDR) turns this from "quiet exfiltration" into "connection refused, and an audit-log egress-denied event you can alert on."

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-db-and-external-api
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payments-api
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.20.30.0/24     # internal DB subnet
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - ipBlock:
        cidr: 203.0.113.10/32  # third-party payment gateway, single pinned IP
    ports:
    - protocol: TCP
      port: 443
```

Combine with the DNS rule above and the default-deny-all baseline; nothing else egresses. Note the ceiling on this control: it operates on IP/port, so it can't distinguish legitimate HTTPS to an allowed CIDR from an attacker tunneling exfil traffic over that same allowed HTTPS endpoint (e.g. abusing a webhook callback to the allowed third party) — pair egress policy with actual DLP/proxy-layer inspection if that's a real threat for the workload, don't treat NetworkPolicy as a complete answer to exfiltration.

## 5. Secrets management maturity ladder

### Stage 0: raw Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=
```

Base64 is an encoding, not encryption — anyone who can read the object (`kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d`) has the plaintext instantly. By default, Secrets are stored in etcd unencrypted — anyone with direct etcd access (a compromised etcd backup, an etcd snapshot lying around, a misconfigured etcd exposed without TLS/auth) reads every secret in the cluster in plaintext. This stage's only real protection is RBAC — `get`/`list` on `secrets`, and that's routinely over-granted (see section 7).

### Stage 1: encryption at rest via EncryptionConfiguration

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

Passed to the API server via `--encryption-provider-config`. Now etcd stores ciphertext for Secret objects; a raw etcd dump or backup no longer yields plaintext directly. `aescbc`/`aesgcm` are local providers — the key material sits in a file on the control plane node, which just moves the "who can read this" problem to "who can read the control plane's encryption config file and rotate/manage it," a smaller and more auditable set than "who has an etcd snapshot," but still a locally-managed key with manual rotation.

A `kms` provider delegates key management to an external KMS (cloud KMS, HashiCorp Vault's transit engine) via a gRPC plugin — the API server calls out to encrypt/decrypt using a key that never leaves the KMS, giving you centralized key rotation, access logging on the KMS side, and revocability independent of the cluster:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - kms:
      apiVersion: v2
      name: myKmsPlugin
      endpoint: unix:///var/run/kmsplugin/socket.sock
  - identity: {}
```

Critical limitation to state plainly: this only protects data at rest inside etcd. It does nothing against someone with legitimate (or stolen) RBAC permission to `kubectl get secret` through the API — the API server transparently decrypts before returning the object to any authorized caller. Encryption-at-rest and RBAC are answering two different threats; neither substitutes for the other.

### Stage 2: external secret stores

Vault issues dynamic, short-TTL credentials instead of static ones — a database secrets engine generates a unique DB user/password per lease, valid for e.g. 1 hour, auto-revoked on expiry. A compromised credential is worthless soon after capture, and every credential is individually attributable to the workload/lease that requested it (no more "which of the 40 pods using this shared DB password did the breach come from").

External Secrets Operator bridges Vault/AWS Secrets Manager/GCP Secret Manager into native `Secret` objects your app can keep consuming unmodified, on a poll/refresh interval:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
  namespace: payments
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-creds
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/payments/db
      property: password
```

This is strictly better than stage 1 because the source of truth and the audit trail move to a system built for it — Vault logs every secret access with identity attribution, supports revocation of a single lease without touching others, and secrets rotate automatically without a deploy. The synced `Secret` object in Kubernetes is still subject to the same etcd/RBAC exposure as stage 0/1 — ESO does not remove that surface, it removes the "static long-lived credential" problem underneath it.

### Stage 3: short-lived tokens / workload identity — the actual end state

IRSA (EKS), Workload Identity (GKE), Azure AD Workload Identity all do the same thing: the pod's ServiceAccount token (a projected, audience-bound, auto-rotating JWT — not a static long-lived SA token) is federated with the cloud IAM provider. The pod exchanges its Kubernetes SA token for short-lived cloud credentials (an AWS STS `AssumeRoleWithWebIdentity` call, e.g.), with no long-lived secret stored anywhere — not in etcd, not in Vault, not in an env var.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-api
  namespace: payments
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/payments-api-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
spec:
  template:
    spec:
      serviceAccountName: payments-api
      containers:
      - name: api
        image: registry.internal/payments-api:1.4.2
        # no AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY anywhere
```

The kubelet projects a token bound to the AWS OIDC audience (`eks.amazonaws.com/serviceaccount`), the AWS SDK inside the pod automatically discovers and exchanges it via the mounted token path (`AWS_WEB_IDENTITY_TOKEN_FILE`), and credentials expire on the order of an hour and rotate transparently. This is the best end state because there is no secret to steal in the first place — an attacker with a shell in the pod gets credentials valid only as long as the pod lives and scoped only to the IAM role attached to that one ServiceAccount, not a static key usable indefinitely from anywhere. Each stage up this ladder removes a category of standing risk: base64 → encrypted at rest → externally managed and rotated → no long-lived secret material at all.

## 6. Image supply chain security

### Scanning with Trivy

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed registry.internal/api:1.4.2
```

Scans OS package layers (apt/apk/yum manifests baked into the image) and language-level dependency manifests (`package-lock.json`, `requirements.txt`/`poetry.lock`, `go.sum`, `pom.xml`) against CVE databases, matching installed version ranges against known-vulnerable ranges. `--exit-code 1` makes it fail the CI job on a match; `--ignore-unfixed` avoids blocking builds on CVEs with no available fix yet (otherwise you get permanently red pipelines for issues you can't act on). Wire into CI as a gate before pushing to the registry that feeds the cluster:

```yaml
# CI step (GitLab/GitHub Actions style, illustrative)
- name: scan image
  run: |
    trivy image --severity HIGH,CRITICAL --exit-code 1 \
      --ignore-unfixed $IMAGE_TAG
```

### Signing and verification with cosign

Keyless signing uses the workload's CI OIDC identity (e.g. a GitHub Actions or GitLab CI OIDC token) instead of a long-lived private key — Sigstore's Fulcio issues a short-lived certificate binding the signature to that identity, and the signature event is recorded in Rekor, a public append-only transparency log, so a signature can be verified as having existed at a specific time even without trusting the signer's key custody.

```bash
cosign sign registry.internal/api:1.4.2 --yes   # keyless, uses CI OIDC token

cosign verify registry.internal/api:1.4.2 \
  --certificate-identity "https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

### Admission-time enforcement

Scanning and signing in CI are advisory unless something at the cluster boundary refuses to run images that skipped them — otherwise a bypassed pipeline, a manually pushed image, or a compromised registry can still land a workload. Kyverno and OPA Gatekeeper both intercept at admission.

Gatekeeper uses the `ConstraintTemplate` (the reusable policy logic, written in Rego) + `Constraint` (an instance of that template, parameterized per namespace/label) pattern. Kyverno uses `ClusterPolicy` directly with declarative rules, no separate Rego required, and has a built-in `verifyImages` rule type purpose-built for cosign.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: verify-cosign-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "registry.internal/*"
      attestors:
      - entries:
        - keyless:
            subject: "https://github.com/org/repo/.github/workflows/build.yml@refs/heads/main"
            issuer: "https://token.actions.githubusercontent.com"
            rekor:
              url: https://rekor.sigstore.dev
```

`validationFailureAction: Enforce` blocks admission on failure (`Audit` would log-only, useful for the same staged-rollout pattern as PSA). This is the modern replacement for the old `ImagePolicyWebhook` admission plugin, which required a bespoke external webhook service and is largely superseded by Kyverno/Gatekeeper's built-in image verification support. Pair this with a second policy blocking images from any registry other than your internal one, so the signature check can't be routed around by pulling an unsigned image from Docker Hub directly.

## 7. RBAC hardening

### Wildcards are a red flag, not a convenience

```yaml
# DO NOT DO THIS
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: overprivileged
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

This isn't just "broad" — it's a moving target. Every new resource type the API server gains (new CRDs installed later, a new built-in resource in a future Kubernetes version) is automatically in scope for anyone bound to this role, with zero further review. A role written this way at cluster bring-up quietly grants access to things that didn't exist when it was written. Scope explicitly:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: payments
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### Auditing what a ServiceAccount can actually do

```bash
kubectl auth can-i --list --as=system:serviceaccount:payments:payments-api -n payments
```

Run this for every ServiceAccount before it ships, not after an incident. It resolves aggregated bindings and shows the effective permission set — the thing that actually matters, versus reading YAML and mentally computing what a chain of RoleBindings adds up to.

### Avoiding cluster-admin sprawl

The classic escalation path in a real breach: a namespace has one ServiceAccount bound (for "convenience," often during initial setup and never revisited) to `cluster-admin` via a ClusterRoleBinding. Every pod in that namespace that doesn't explicitly set a different `serviceAccountName` gets the `default` SA mounted automatically, and if that's the bound one, every pod in the namespace — including a low-risk sidecar or a batch job nobody thought about — carries a token that can do anything in the cluster. An attacker who gets code execution in any single pod in that namespace now has full cluster control by just reading the mounted token at `/var/run/secrets/kubernetes.io/serviceaccount/token` and using it against the API server directly.

```bash
# find this before someone else does
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .subjects[]? | "\(.kind)/\(.namespace // "cluster")/\(.name)"'
```

Never bind `cluster-admin` to a namespaced ServiceAccount. If a workload genuinely needs broad permissions, write a scoped ClusterRole enumerating exactly the resources/verbs it needs, and also disable auto-mounting for pods that don't need API access at all:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: payments
automountServiceAccountToken: false
```

### Aggregated ClusterRoles can silently expand scope

Aggregation composes ClusterRoles by label match rather than explicit reference:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-aggregate
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # populated automatically from matching ClusterRoles
```

Any ClusterRole later created (by anyone, including a helm chart, an operator's CRD installation, or a compromised deploy pipeline) carrying that same label gets folded into `monitoring-aggregate`'s effective rule set automatically, with no further review of the aggregate role itself. If `monitoring-aggregate` is bound somewhere broadly (a common pattern for giving an observability stack cluster-wide read access), a rogue ClusterRole with the matching label — installed by a compromised chart dependency, say — silently grants itself whatever the aggregate is bound to. Treat aggregation labels as a security-relevant string, review anything that adds them, and never use a generic/guessable label like `rbac-aggregate: "true"` that other charts might reuse by coincidence.

## 8. Audit logging

### Levels and the volume tradeoff

- `None` — don't log the event
- `Metadata` — log the request metadata (who, what verb, what resource, timestamp, response code) but not bodies
- `Request` — Metadata plus the request body
- `RequestResponse` — Metadata plus both request and response bodies

`RequestResponse` on everything is the naive first instinct for "log everything for forensics" and it's usually the wrong default cluster-wide: response bodies for `list`/`get` on Secrets contain the actual secret values in plaintext in your audit log, response bodies for large `list` calls (all pods, all configmaps) balloon log volume by orders of magnitude, and most of that volume is never read. Scope `RequestResponse` narrowly to the handful of resource types where the forensic value justifies the volume and the plaintext-secret risk, and handle that risk by routing the audit log to a store with equal or stricter access control than the Secrets themselves.

### Real audit policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]
- level: RequestResponse
  resources:
  - group: "networking.k8s.io"
    resources: ["networkpolicies"]
- level: Metadata
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["pods", "configmaps", "services"]
- level: Metadata
  omitStages:
  - RequestReceived
```

The last catch-all rule ensures anything not explicitly matched above still gets `Metadata`-level coverage rather than silently falling through unlogged — audit policy rules are evaluated top-down, first match wins, so ordering here (specific/sensitive resources first, catch-all last) is load-bearing, not stylistic.

If logging Secret `RequestResponse` bodies in plaintext is unacceptable even with access control on the audit sink, an audit webhook backend can apply redaction before persisting (strip `.data`/`.stringData` fields, keep everything else) — vanilla file/webhook backends don't redact automatically, so this requires either a custom webhook receiver or accepting the plaintext-in-audit-log tradeoff consciously.

### What to prioritize in a real investigation

When triaging a suspected compromise, the audit log queries that matter most, roughly in the order you'll actually need them:

- Every `get`/`list` on `secrets`, filtered to unexpected identities — who read a Secret they don't normally touch, and when
- Every `create` on `rolebindings`/`clusterrolebindings`/`roles`/`clusterroles` — privilege escalation almost always shows up here first, before any secret is touched
- Every `create` on `pods/exec` and `pods/attach` — this is "someone got an interactive shell into a running container," the single highest-signal event for lateral movement
- Every `create`/`update`/`delete` on `networkpolicies` — an attacker disabling or loosening egress restriction to enable exfiltration leaves a clear audit trail here
- Correlate all of the above by `user.username` / `impersonatedUser` and by `sourceIPs` — a ServiceAccount token used from an IP outside the expected node CIDR range is a strong compromise indicator on its own, independent of what it was used for
