# kubectl and Cluster Interaction

## kubectl architecture

`kubectl` is a stateless HTTP client. It does not talk to nodes, kubelets, or containers directly — every operation is a REST call to the **kube-apiserver**, authenticated and authorized like any other client.

Flow for `kubectl get pods`:
1. `kubectl` reads `~/.kubeconfig` (or `$KUBECONFIG`) to determine the current context: which cluster (API server URL + CA cert), which user (credentials), which default namespace.
2. It builds an HTTPS request to `<api-server>/api/v1/namespaces/<ns>/pods`, attaching credentials (client cert, bearer token, exec-plugin-generated token, or basic auth — deprecated).
3. The API server authenticates the request, then runs it through admission control and RBAC authorization.
4. The API server reads from **etcd** (via its own internal cache) and returns the object list as JSON; `kubectl` formats it for display.

`kubectl exec`/`logs`/`port-forward`/`cp`/`attach` are the exception to "never touches nodes directly" — for these, the API server proxies (or upgrades to a streaming connection via SPDY/WebSocket) to the target node's **kubelet**, which then talks to the container runtime. So `kubectl logs` still goes through the API server as a proxy, but the API server in turn calls the kubelet, which calls the CRI (containerd/CRI-O) to stream logs — a longer path than a plain `get`.

### kubeconfig structure

```yaml
apiVersion: v1
kind: Config
clusters:
- name: prod-cluster
  cluster:
    server: https://prod-api.example.com:6443
    certificate-authority-data: <base64 CA cert>
- name: staging-cluster
  cluster:
    server: https://staging-api.example.com:6443
    certificate-authority-data: <base64 CA cert>
users:
- name: siddharth-prod
  user:
    client-certificate-data: <base64 cert>
    client-key-data: <base64 key>
- name: siddharth-eks
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args: ["eks", "get-token", "--cluster-name", "prod-cluster"]
contexts:
- name: prod
  context:
    cluster: prod-cluster
    user: siddharth-prod
    namespace: payments
- name: staging
  context:
    cluster: staging-cluster
    user: siddharth-eks
    namespace: default
current-context: prod
```

Three independent object types, combined by a **context**:
- **clusters**: where the API server is and how to trust its cert.
- **users**: how to authenticate — client cert/key, bearer token, or an `exec` plugin (common for cloud providers: `aws eks get-token`, `gcloud container clusters get-credentials`-generated configs, `az aks` — these generate short-lived tokens on demand instead of storing long-lived credentials in the file).
- **contexts**: a named (cluster, user, namespace) triple — what you actually switch between.

### Context switching

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context staging
kubectl config set-context --current --namespace=billing
kubectl config view --minify              # show only the active context's config, secrets redacted
kubectl config view --minify --raw         # include actual credential data (be careful with this)

# merge multiple kubeconfig files (colon-separated on Linux/Mac)
KUBECONFIG=~/.kube/config:~/.kube/staging-config kubectl config view --flatten > ~/.kube/merged-config
```

`--context`, `--namespace`, `--cluster`, `--user` flags on any `kubectl` command override the current context for that single invocation without switching it persistently — safer for one-off cross-cluster commands than `use-context`.

Tools like `kubectx`/`kubens` (not built into kubectl) wrap the same config-switching for speed; worth knowing the raw `kubectl config` commands since not every environment has them installed.

## Essential command patterns

### get

```bash
kubectl get pods                                  # current namespace
kubectl get pods -A                                # all namespaces (--all-namespaces)
kubectl get pods -o wide                           # + node, IP, readiness gates
kubectl get pods -o yaml                            # full object as YAML
kubectl get pods -o json | jq '.items[].metadata.name'
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase'
kubectl get pods -w                                 # watch: stream changes, doesn't exit
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get deploy,svc,ingress -n payments          # multiple resource types at once
```

- `-o wide`: adds columns useful for debugging (node placement, pod IP) without the verbosity of full YAML.
- `-o yaml`/`-o json`: full object, including server-populated fields (`status`, `resourceVersion`, `uid`) — use this to see what's actually running, not just what you applied.
- `-w`: keeps the connection open and streams ADD/MODIFY/DELETE events; useful for watching a rollout, but doesn't replace `kubectl rollout status` for actual completion tracking.

### describe

```bash
kubectl describe pod mypod
kubectl describe node worker-3
```

`describe` aggregates the object's spec, status, **and recent Events** related to it (scheduling failures, image pull errors, probe failures, OOMKills) — this is usually the first command to run when something's wrong, before `logs`, since Events often show the failure reason directly (`FailedScheduling: Insufficient cpu`, `Back-off pulling image`, `Liveness probe failed`).

### logs

```bash
kubectl logs mypod
kubectl logs mypod -c sidecar-container            # multi-container pod, must specify
kubectl logs mypod --previous                       # logs from the crashed/previous instance
kubectl logs -f mypod                                # follow (stream)
kubectl logs mypod --since=1h
kubectl logs mypod --tail=200
kubectl logs -l app=payments-api --all-containers --max-log-requests=10   # across matching pods
```

`--previous` is the single most useful flag for debugging CrashLoopBackOff — the current container instance's logs are often empty/just-started, while `--previous` shows why the last instance actually died.

### exec

```bash
kubectl exec -it mypod -- /bin/sh
kubectl exec -it mypod -c app -- env
kubectl exec mypod -- cat /etc/config/app.conf     # non-interactive, single command
```

`-it` = allocate a TTY (`-t`) and keep stdin open (`-i`) — needed for an interactive shell, not needed for a single non-interactive command.

### port-forward

```bash
kubectl port-forward pod/mypod 8080:80
kubectl port-forward svc/payments-api 8080:80
kubectl port-forward deploy/payments-api 8080:80    # picks one backing pod
```

Opens a local tunnel through the API server to the pod — useful for hitting an internal service (a database, an admin UI) from your laptop without exposing it via Ingress/NodePort. Local-only, dies when the terminal session ends, not a substitute for a real Service for anything other than ad-hoc debugging.

### apply, diff, edit

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/                       # whole directory
kubectl apply -k overlays/prod/                      # kustomize
kubectl apply -f deployment.yaml --dry-run=client -o yaml   # render locally, no API call at all
kubectl apply -f deployment.yaml --dry-run=server           # send to API server, run admission/validation, don't persist
kubectl diff -f deployment.yaml                      # show what apply would change, without applying
kubectl edit deployment payments-api                 # opens live object in $EDITOR, applies on save
```

- `--dry-run=client`: purely local rendering/validation — doesn't even contact the API server. Fast, good for generating YAML (`kubectl create deployment x --image=y --dry-run=client -o yaml > deploy.yaml`), but won't catch admission webhook rejections or quota violations.
- `--dry-run=server`: sends the request to the API server, runs it through validation and admission control (webhooks, quota checks, PodSecurity admission), but stops before persisting to etcd — catches issues client-side dry-run can't.
- `kubectl diff` requires `KUBE_EDITOR`/`diff` to be configured but works out of the box on most systems using standard `diff`; always run before `apply` on production objects you didn't author yourself.
- `kubectl edit` opens the **live** object (with server-populated fields) in your editor; saving triggers a `PATCH`. Changes are lost on the next `kubectl apply` from a manifest/Helm/Kustomize source that doesn't know about your edit (see server-side apply below) — `edit` is for emergency/exploratory changes, not standard workflow.

### diff and rollout for Deployments specifically

```bash
kubectl rollout status deployment/payments-api        # blocks until rollout completes or times out
kubectl rollout history deployment/payments-api
kubectl rollout undo deployment/payments-api
kubectl rollout undo deployment/payments-api --to-revision=3
kubectl rollout restart deployment/payments-api        # force new pods without a spec change (e.g., to pick up a ConfigMap/Secret update)
```

`rollout restart` is the standard way to force pods to restart after changing a mounted ConfigMap/Secret (which doesn't itself trigger a rollout — Kubernetes doesn't watch mounted volume content for changes by default).

## Label selectors and field selectors

**Label selectors** — match on `metadata.labels`, the primary mechanism connecting Services to Pods, Deployments to ReplicaSets, etc.

```bash
kubectl get pods -l app=payments-api
kubectl get pods -l 'app=payments-api,tier=backend'      # AND (comma = AND)
kubectl get pods -l 'environment in (prod,staging)'       # set-based
kubectl get pods -l 'tier notin (frontend)'
kubectl get pods -l 'app'                                  # key exists, any value
kubectl get pods -l '!app'                                 # key does not exist
kubectl label pod mypod tier=backend
kubectl label pod mypod tier=backend --overwrite           # required if key already exists
kubectl label pod mypod tier-                                # remove a label
```

Selectors on Deployments/Services (`spec.selector`) are **immutable after creation** for Deployments (changing `spec.selector` is rejected by the API server) — this is why you should choose selector labels carefully upfront (typically `app.kubernetes.io/name` + `app.kubernetes.io/instance` per the recommended labels convention) rather than relying on labels you might want to change later.

**Field selectors** — match on fields of the object itself (a much smaller supported set than labels, varies by resource type):

```bash
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase!=Running
kubectl get events --field-selector involvedObject.name=mypod
kubectl get pods --field-selector spec.nodeName=worker-3
kubectl get namespaces --field-selector metadata.name!=kube-system
```

Field selectors cannot do arbitrary field matching — only fields the resource's API explicitly supports (check with `kubectl explain` or API docs); unlike labels, you can't invent arbitrary field-selector keys.

## kubectl debugging commands

### kubectl debug — ephemeral containers and node debugging

```bash
# Attach a debug container to a running pod (doesn't restart it) — for distroless/minimal images with no shell
kubectl debug -it mypod --image=busybox:1.36 --target=app -- sh

# Create a copy of a pod with a debug container added, leaving the original untouched
kubectl debug mypod -it --image=busybox:1.36 --copy-to=mypod-debug --container=debugger -- sh

# Debug a node directly — schedules a privileged pod with the node's filesystem mounted at /host
kubectl debug node/worker-3 -it --image=busybox:1.36
```

- `--target` attaches the debug container to share the **process namespace** of an existing container in the pod — lets you `ps`/inspect the target process from the debug container even though the original image has no debugging tools installed (common for distroless/scratch production images).
- `--copy-to` is safer for production pods: it creates a **new** pod (a copy with the debug container injected) instead of mutating the live one, so you're not risking the running workload.
- `node/worker-3` debugging drops you into a pod with the host's `/` mounted at `/host` — from there `chroot /host` gives you an effective node shell without SSH access, useful when SSH is locked down or unavailable (managed node pools).

### Ephemeral containers directly

`kubectl debug --target` under the hood uses the **EphemeralContainers** subresource (stable since 1.25) — you can also add one manually via `kubectl edit pod mypod --subresource=ephemeralcontainers` or by constructing the API call, though `kubectl debug` is the practical interface. Ephemeral containers cannot be removed once added (no delete API) — the only way to get rid of them is to delete the pod.

### cp

```bash
kubectl cp mypod:/var/log/app.log ./app.log
kubectl cp ./config.json mypod:/etc/app/config.json -c app
```

Uses `tar` inside the container under the hood — fails silently or with confusing errors if `tar` isn't present in the image (common with distroless/scratch images); for those, use `kubectl exec ... cat file > local` as a workaround, or an ephemeral debug container with `cp`/`tar` present.

### top

```bash
kubectl top nodes
kubectl top pods -n payments
kubectl top pods --containers                        # per-container breakdown
```

Requires the **metrics-server** add-on to be installed in the cluster (not built into the control plane by default) — if `top` returns "metrics not available," metrics-server is either missing or its pods aren't Ready. `top` reflects real-time usage from cAdvisor via the Metrics API — distinct from Prometheus/long-term monitoring; it's a live snapshot only, not historical.

## API discovery

```bash
kubectl api-resources                              # every resource type: name, shortnames, apiGroup, namespaced?, Kind
kubectl api-resources --namespaced=false
kubectl api-resources --api-group=apps
kubectl api-versions                                # every apiGroup/version the API server currently serves
kubectl explain pod                                  # docs for the Pod schema, from the API server's OpenAPI
kubectl explain pod.spec.containers                 # drill into a nested field
kubectl explain pod.spec.containers.livenessProbe --recursive
kubectl explain deployment.spec.strategy.rollingUpdate
```

`kubectl explain` pulls live schema documentation directly from the connected cluster's API server (via its OpenAPI/discovery endpoint) — this means it's always accurate for that cluster's actual API version and any CRDs installed, unlike documentation that might be for a different Kubernetes version. Use it constantly instead of guessing field names/nesting from memory.

```bash
kubectl explain crd.spec                            # works for CRDs too, once installed
kubectl get --raw /openapi/v2 | jq .                # raw OpenAPI schema, rarely needed directly
```

## Editing live objects safely

### Server-side apply vs client-side apply

**Client-side apply** (the traditional `kubectl apply` behavior, still the default unless you pass `--server-side`): `kubectl` computes a three-way diff **locally** — last-applied-configuration (stored in an annotation, `kubectl.kubernetes.io/last-applied-configuration`), the local file, and the live object — then sends a PATCH. Problems:
- The "last applied config" is stored as JSON inside an annotation on the object itself, which has a size limit and gets unwieldy on large objects.
- Only one tool's view of "last applied" is tracked — if two tools (Helm and a manual `kubectl apply`, or two CI pipelines) both client-side apply to the same object, they silently fight, each overwriting fields the other set without any conflict signal.

**Server-side apply** (`kubectl apply --server-side`, GA since 1.22, becoming more of a default expectation in newer tooling): the API server itself tracks **field ownership** per "manager" (the name of the applying tool/client) via `metadata.managedFields`. Each field in an object is attributed to whichever manager last set it.

```bash
kubectl apply -f deployment.yaml --server-side
kubectl apply -f deployment.yaml --server-side --field-manager=ci-pipeline
kubectl get deployment payments-api -o yaml    # inspect metadata.managedFields to see per-field ownership
```

- If a second manager tries to change a field currently owned by a different manager, the API server returns a **409 Conflict** by default — an explicit signal instead of silent field-stomping.
- `--force-conflicts` forces the change and reassigns ownership of the contested field to the current manager: `kubectl apply -f deployment.yaml --server-side --force-conflicts`.
- Reading fields does not take ownership — only fields actually specified in the applied manifest are claimed; fields left unset (e.g., `status`, or fields another controller like an HPA manages, such as `spec.replicas` when an HPA is active) are left alone, avoiding the classic "Helm/kubectl apply keeps resetting the replica count the HPA just set" bug that plagues client-side apply setups.

### Practical guidance for live edits

- Prefer changing the source manifest (Helm values, Kustomize overlay, raw YAML in git) and re-applying over `kubectl edit`/`kubectl patch` directly on the cluster — direct edits create drift between git and the cluster that the next automated deploy will silently overwrite (or, worse, be silently overwritten by if you're on server-side apply with a different field manager already owning that field).
- If you must patch live for an emergency, use `kubectl patch` with a clear scope instead of `kubectl edit` in a pinch to keep the change scriptable/auditable:

```bash
kubectl patch deployment payments-api -p '{"spec":{"replicas":10}}'
kubectl patch deployment payments-api --type=json -p '[{"op":"replace","path":"/spec/replicas","value":10}]'
```

- Always follow an emergency live patch by updating the actual source of truth (git-tracked manifest/Helm values) as soon as the incident is over — otherwise the next `helm upgrade`/`kubectl apply`/Argo CD sync reverts your emergency fix without warning.
- When debugging "why did my change get reverted," check `metadata.managedFields` (or `kubectl get -o yaml` and look at annotations for client-side apply) to see which controller/tool currently owns the field in question — this is the single most useful piece of evidence for field-ownership conflicts between GitOps controllers, HPAs, and manual changes.
