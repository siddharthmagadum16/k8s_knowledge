# Helm and Package Management

## The problem raw YAML doesn't solve

A real application is rarely one YAML file. A typical service needs a Deployment, a Service, a ConfigMap, a Secret reference, an Ingress, an HPA, a ServiceAccount, and RBAC objects — often 8-15 files. Managing these as raw YAML across multiple environments (dev/staging/prod) creates concrete problems:

- **Duplication**: the same Deployment YAML copy-pasted per environment, differing only in replica count, image tag, and resource limits — drift creeps in as one copy gets edited and others don't.
- **No parameterization**: raw `kubectl apply -f` has no native concept of "same template, different values." You either hand-edit files per environment or build your own templating with `sed`/`envsubst`, which is unversioned and error-prone.
- **No release concept**: `kubectl apply` doesn't track "what did I actually deploy as a unit" or "what changed between this deploy and the last one" — no atomic rollback of a whole application's set of resources.
- **No dependency management**: if your app needs Redis and Postgres as supporting services, there's no standard way to declare "install this dependency chart too" with raw manifests.
- **No packaging/distribution**: no standard way to version, share, and pull a "known-good" set of manifests for common software (nginx-ingress, cert-manager, Prometheus) the way `apt`/`npm`/`pip` work for packages.

Helm addresses all of these: templated manifests, versioned values, atomic multi-resource releases with rollback, and a package registry model (chart repositories, OCI registries).

## Helm architecture

- **Chart**: a package of Kubernetes manifest templates plus metadata — the unit of distribution (analogous to a `.deb` or an npm package).
- **Values**: the parameters injected into a chart's templates (`values.yaml`, plus overrides via `-f` files or `--set` flags).
- **Release**: an instance of a chart deployed into a cluster with a specific set of values, tracked by name. You can install the same chart multiple times under different release names (e.g., `helm install redis-cache bitnami/redis` and `helm install redis-sessions bitnami/redis`).
- **Repository**: an HTTP(S) endpoint (or OCI registry) serving an index of chart packages, analogous to a package repo.

### Helm 2 vs Helm 3

Helm 2 required **Tiller**, a server-side component running inside the cluster with broad (often cluster-admin) RBAC permissions, acting as a proxy between the `helm` CLI and the API server. This was Helm 2's biggest liability: Tiller's permissions were a major attack surface — anyone who could talk to Tiller effectively had Tiller's RBAC permissions, and by default that was frequently cluster-admin.

Helm 3 (current, since late 2019) removed Tiller entirely:
- The `helm` CLI talks directly to the Kubernetes API server, authenticating with your kubeconfig — same permission model as `kubectl`. No separate in-cluster privileged component.
- Release state (what's installed, at what revision, with what values) is stored as **Secrets in the target namespace** (`sh.helm.release.v1.<release>.v<revision>`), not in a separate Tiller-managed ConfigMap in `kube-system`. This means releases are namespace-scoped and RBAC-governed like everything else.
- Native support for OCI registries as chart repositories (`helm push`/`helm pull` against an OCI registry, e.g., ECR, GHCR, ACR).
- Improved upgrade semantics via three-way merge (see diffing below) instead of Helm 2's two-way diff, reducing "upgrade silently reverted a manual `kubectl edit`" surprises (though not eliminating them).

If you see any reference to `tiller` or `helm init` in documentation, it's Helm 2 — obsolete, unsupported, do not use for new work.

## Chart structure

```
mychart/
├── Chart.yaml           # chart metadata: name, version, appVersion, dependencies
├── values.yaml          # default configuration values
├── values.schema.json   # optional: JSON Schema to validate values
├── charts/              # subcharts (dependencies), vendored or fetched
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # named template definitions, no manifest output itself
│   ├── NOTES.txt        # post-install usage message shown to the user
│   └── tests/
│       └── test-connection.yaml
└── .helmignore           # files to exclude when packaging
```

### Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for the payments API
type: application
version: 1.4.2        # chart version — bump on every change to the chart itself
appVersion: "2.3.0"    # version of the application the chart deploys (informational)
dependencies:
- name: redis
  version: "18.x.x"
  repository: "https://charts.bitnami.com/bitnami"
  condition: redis.enabled
```

Distinguish **chart version** (`version`, semver of the packaging) from **appVersion** (the version of the software inside, purely informational — Helm does not parse or enforce it).

### values.yaml

```yaml
replicaCount: 3
image:
  repository: myregistry/payments-api
  tag: "2.3.0"
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
ingress:
  enabled: true
  host: payments.example.com
redis:
  enabled: true
```

### _helpers.tpl

Defines reusable named templates (not standalone manifests — files prefixed `_` are not rendered as manifests themselves):

```yaml
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

Used in a template via `{{ include "mychart.labels" . }}`.

### Subcharts / dependencies

Declared in `Chart.yaml` under `dependencies`, fetched with:

```bash
helm dependency update mychart/    # downloads into charts/ based on Chart.yaml + Chart.lock
helm dependency build mychart/     # rebuilds from the existing Chart.lock without re-resolving versions
```

Parent chart values namespace subchart values under the subchart's name:

```yaml
# parent's values.yaml
redis:
  auth:
    enabled: true
  master:
    persistence:
      size: 8Gi
```

This overrides the `redis` subchart's own `auth.enabled` and `master.persistence.size` defaults. `condition: redis.enabled` in Chart.yaml lets the parent chart toggle the entire subchart on/off.

## Templating basics

Helm templates are Go templates (`text/template`) plus the Sprig function library, evaluated against a context that includes `.Values`, `.Release`, `.Chart`, `.Files`, `.Capabilities`.

### Values injection

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

- `{{ .Values.x }}` reads from `values.yaml` (or overrides).
- `nindent N` / `indent N`: reindents a block to fit YAML nesting — critical, since YAML is whitespace-sensitive and Go templates aren't indentation-aware by default.
- `toYaml`: dumps an arbitrary values subtree (map/list) as YAML — used heavily for passing through free-form blocks like `resources`, `nodeSelector`, `tolerations`, `affinity` without re-declaring their structure in the template.

### Conditionals

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "mychart.fullname" . }}
            port:
              number: 80
{{- end }}
```

If `ingress.enabled` is `false`, this entire template file renders to empty output and Helm skips creating an empty manifest for it.

### Ranges (loops)

```yaml
env:
{{- range .Values.extraEnvVars }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

with values:

```yaml
extraEnvVars:
- name: LOG_LEVEL
  value: info
- name: FEATURE_FLAG_X
  value: "true"
```

### Named templates

```yaml
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

Called with `{{ include "mychart.selectorLabels" . | nindent 4 }}` — `include` (vs the built-in `template`) is preferred because its output can be piped to functions like `nindent`.

### Debugging templates

```bash
helm template mychart/ -f values-prod.yaml      # render locally, no cluster contact
helm template mychart/ --debug                  # show values used + rendering errors
helm install myrelease mychart/ --dry-run --debug
helm lint mychart/                               # static checks: required fields, YAML validity
```

Always `helm template`/`--dry-run` before applying an unfamiliar chart — a chart with an error deep in a conditional branch you didn't trigger locally can otherwise reach the cluster.

## Helm lifecycle commands

```bash
# Install
helm install myrelease mychart/ -f values-prod.yaml --namespace payments --create-namespace

# Upgrade (creates a new revision; if release doesn't exist, --install creates it)
helm upgrade myrelease mychart/ -f values-prod.yaml --namespace payments

# Upgrade with atomic rollback on failure
helm upgrade myrelease mychart/ --atomic --timeout 5m

# Rollback to a previous revision
helm history myrelease -n payments
helm rollback myrelease 3 -n payments   # roll back to revision 3

# Uninstall
helm uninstall myrelease -n payments

# Diff before applying (requires the helm-diff plugin)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myrelease mychart/ -f values-prod.yaml -n payments
```

### How release history works

Every `helm install`/`upgrade`/`rollback` creates a new **revision** for that release, stored as a Secret (`sh.helm.release.v1.<name>.v<N>`) in the release's namespace, containing the fully-rendered manifest and the values used. `helm rollback` doesn't recompute anything — it re-applies the manifest captured in the target revision's stored Secret.

```bash
helm history myrelease -n payments
# REVISION  UPDATED                   STATUS      CHART            APP VERSION  DESCRIPTION
# 1         Mon Jul 14 10:00:00 2026  superseded  mychart-1.4.0    2.2.0        Install complete
# 2         Tue Jul 15 09:00:00 2026  superseded  mychart-1.4.1    2.2.1        Upgrade complete
# 3         Wed Jul 16 14:00:00 2026  deployed    mychart-1.4.2    2.3.0        Upgrade complete
```

- Default retained history: 10 revisions (`--history-max` on install/upgrade to change).
- `helm get values myrelease -n payments` shows the values used for the current (or a specified `--revision`) deployed revision — critical for reproducing what's actually running versus what's in your local `values-prod.yaml`, since ad-hoc `--set` flags used in a past deploy aren't otherwise visible in git.
- `helm get manifest myrelease -n payments` dumps the fully-rendered YAML actually applied for the current release — use this, not the raw templates, to see what's really in the cluster.

### Upgrade mechanics (3-way merge)

Helm 3 upgrades compute a diff across three states: the old manifest (last stored release), the new manifest (freshly rendered), and the **live object state in the cluster**. This means a field manually changed via `kubectl edit` since the last Helm operation (drift) is detected and, depending on the field, may be reconciled or preserved — but Helm generally does not preserve manual `kubectl edit` changes to fields it manages; anything Helm's template controls will be reset to the templated value on the next `helm upgrade`. Don't hand-edit Helm-managed objects and expect it to stick.

## Kustomize as an alternative

Kustomize takes a different philosophy: no templating language at all. You write **plain, valid YAML** manifests (a "base"), then declare **patches** and **overlays** that Kustomize merges structurally (strategic merge patch or JSON patch) to produce environment-specific variants — no `{{ }}` syntax, no values files, no rendering engine to learn.

### Structure

```
base/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
overlays/
├── dev/
│   ├── kustomization.yaml
│   └── replica-patch.yaml
└── prod/
    ├── kustomization.yaml
    └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
- ../../base
patches:
- path: replica-patch.yaml
  target:
    kind: Deployment
    name: payments-api
images:
- name: myregistry/payments-api
  newTag: "2.3.0"
namespace: payments-prod
```

```yaml
# overlays/prod/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
spec:
  replicas: 5
```

```bash
kubectl apply -k overlays/prod/
kustomize build overlays/prod/   # render without applying, for review
```

`kubectl` has built-in Kustomize support (`-k` flag) — no separate binary required for basic use, though the standalone `kustomize` CLI has more features (newer patch strategies, plugins).

### When to prefer Kustomize over Helm

| | Helm | Kustomize |
|---|---|---|
| Learning curve | Go templates + Sprig, a real templating language to learn | Plain YAML + patch syntax, easier for teams unfamiliar with templating |
| Packaging/distribution | Strong — versioned charts, public/private repos, OCI registries, dependency management | Weak — no native packaging or versioning of "charts"; typically consumed via git directly |
| Environment overlays | Via values files per environment (`values-dev.yaml`, `values-prod.yaml`) | Native, first-class concept (`overlays/`) |
| Third-party software installs | Dominant model — most projects (ingress-nginx, cert-manager, Prometheus stack, most CNCF projects) ship Helm charts | Rare for third-party distribution |
| Release/rollback tracking | Built-in (`helm history`, `helm rollback`) | None — relies on your GitOps tool (Argo CD/Flux) or `kubectl rollout undo` at the Deployment level |
| Conditional logic (loops, if/else) | Full templating power, can get complex/hard to read | Deliberately limited — patches only, keeps output auditable as plain YAML |
| Best fit | Installing/managing third-party charts; complex apps needing subcharts and conditional logic | In-house apps where you want git-diffable, template-free manifests; simpler mental model for overlay-per-environment |

**Practical pattern many teams use**: Kustomize for their own application manifests (dev/staging/prod overlays, no templating magic, easy to `git diff`), Helm for consuming third-party infrastructure charts (ingress controllers, cert-manager, monitoring stacks) where the upstream project already publishes a chart. GitOps tools (Argo CD, Flux) support both natively, so this hybrid is a mainstream production pattern rather than a compromise.

### Kustomize limitations to know

- No conditional installs of whole components based on a boolean flag the way `{{- if .Values.x }}` works in Helm — you either include a resource in an overlay's `resources:` list or you don't; toggling requires maintaining separate overlay variants or using `components` (a newer Kustomize feature for optional reusable patch sets).
- No dependency management equivalent to Helm's `charts/` subchart mechanism — if you need "install Redis alongside my app," you either vendor Redis's manifests yourself or reach for Helm for that piece.
- Patch precedence and merge behavior (strategic merge vs JSON6902 patches) has a learning curve of its own once overlays get deep (overlay of an overlay), even though the base language (YAML) stays simple.
