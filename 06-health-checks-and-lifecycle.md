# Health Checks and Pod Lifecycle

## Why probes exist

Kubernetes cannot infer application health from process state alone. A process can be running (PID exists, container "Running") while the application inside is deadlocked, still booting, or has lost its database connection. Probes let you tell the kubelet how to check actual application health, and let the kubelet act on that information — restart the container, or pull it out of Service load balancing.

Three probe types exist, each answering a different question and triggering a different action:

| Probe | Question it answers | Action on failure | Checked by |
|---|---|---|---|
| **livenessProbe** | Is the app in a state it can never recover from without a restart (deadlock, hung thread pool)? | kubelet kills and restarts the container (subject to `restartPolicy`) | kubelet |
| **readinessProbe** | Can the app currently serve traffic? | Pod is removed from Service/Endpoints (and from Ingress backends); container is NOT restarted | kubelet, reflected in Endpoints/EndpointSlice controller |
| **startupProbe** | Has the app finished its (possibly slow) initial boot? | Container is killed and restarted (liveness/readiness are not even evaluated until startup succeeds) | kubelet |

Precise mental model:

- **startupProbe**, if defined, runs first. While it is running, `livenessProbe` and `readinessProbe` are **disabled** (not evaluated at all). Only after startupProbe succeeds once do liveness/readiness begin being polled.
- **livenessProbe** and **readinessProbe** run in parallel for the lifetime of the container once startup has succeeded (or immediately if no startupProbe is defined).
- Readiness failures are far more common in practice than liveness failures, and readiness is what actually protects users — it's the mechanism that keeps traffic away from pods that are up but not ready (e.g., still warming a cache, or a dependency is down).

### Consequences of misconfiguration

**Liveness probe too aggressive (short timeout/low failureThreshold) on a slow app:**
- App gets killed mid-request under load (e.g., GC pause, slow downstream call) → kubelet restarts it → CrashLoopBackOff-like restart cycling even though the app was never actually broken.
- This is the single most common production incident caused by probes: "the app restarts under load" is almost always an overly strict liveness probe, not an actual bug.
- Fix: liveness probes should check only "is the process fundamentally alive," not full dependency health. Never point livenessProbe at an endpoint that calls a database or downstream service — a downstream outage would then kill every pod in the deployment simultaneously (a self-inflicted total outage).

**Readiness probe too aggressive or checking the wrong thing:**
- If readiness checks a downstream dependency and that dependency blips, every pod goes NotReady at once → Service has zero endpoints → total outage for a component that was otherwise healthy.
- If readiness probe has `initialDelaySeconds` too low for a JVM/Node app with a slow cold start, the pod is marked Ready before it can actually handle requests → first requests during rollout fail or are slow (5xx spike right after deploy).
- If readiness is missing entirely, Kubernetes assumes the pod is ready the instant the container process starts (after `initialDelaySeconds`, default 0) — traffic hits it before the app finished loading configuration, warming connection pools, etc.

**Rolling updates stall when readiness never succeeds:**
- A Deployment rolling update waits for new ReplicaSet pods to become `Ready` before scaling down old pods (governed by `maxUnavailable`/`maxSurge`). If the readiness probe on new pods never passes (bad config, missing secret, wrong probe path), the rollout **hangs indefinitely** — it does not roll back automatically. `kubectl rollout status` will block; you must `kubectl rollout undo` manually.
- This is why CI/CD pipelines should always run `kubectl rollout status deployment/x --timeout=120s` and treat a timeout as a failed deploy.

## Probe types

### httpGet

Most common for web services. Kubelet issues a GET request to the pod IP (not through a Service) and expects a status code between 200 and 399.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Probe
      value: kubelet
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

- `port` can be a name (from container's `ports:`) or a number.
- `scheme: HTTPS` supported; kubelet does not verify certs by default.
- Response body is ignored — only status code matters.

### tcpSocket

Just opens a TCP connection to the port. Useful for non-HTTP protocols (databases, gRPC without health service, raw TCP servers) where you only need to know "is something listening."

```yaml
readinessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 3
```

Weak signal — a TCP accept queue can respond even if the app behind it is stuck. Prefer httpGet or exec when the app can expose real health logic.

### exec

Runs a command inside the container's namespace. Exit code 0 = success, anything else = failure.

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "pg_isready -U postgres || exit 1"
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

- Runs as a new process inside the container — has CPU/memory overhead if invoked frequently (relevant for very short `periodSeconds` at scale, thousands of pods).
- No network hop needed; useful for CLI-based health tools (e.g., `mysqladmin ping`, `redis-cli ping`).

### grpc

Native since Kubernetes 1.24 (GA in 1.27). Requires the app to implement the [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md) (`grpc.health.v1.Health` service).

```yaml
livenessProbe:
  grpc:
    port: 9090
    service: "" # optional; empty = overall server health
  initialDelaySeconds: 10
  periodSeconds: 10
```

- No need to vendor a shell or curl into the image just to run health checks — good fit for distroless/scratch images.
- If your gRPC server doesn't implement the health service, this probe will fail; use `tcpSocket` as a fallback for distroless gRPC apps that haven't added it.

## Tuning fields

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10  # wait before first probe
  periodSeconds: 10        # how often to probe
  timeoutSeconds: 1        # probe must respond within this
  successThreshold: 1      # consecutive successes to go Healthy (must be 1 for liveness)
  failureThreshold: 3      # consecutive failures before action
```

- **initialDelaySeconds**: grace period before the first probe fires. Set to your app's typical cold-start time (measure it, don't guess). Too low → false failures during startup, especially for JVM/Spring Boot apps (5-30s startup is common). Prefer using a `startupProbe` instead of inflating this, since a fixed delay is either too short under load or wastes time in the common case.
- **periodSeconds**: how often to re-check. Default 10s. Very low values (1-2s) increase load on the app and the kubelet; fine for lightweight tcpSocket checks, wasteful for exec probes forking processes.
- **timeoutSeconds**: if the probe doesn't respond in this window, it counts as a failure. Must be less than `periodSeconds`. Too low causes false failures when the app is momentarily slow (GC pause, CPU throttling under a CPU limit).
- **successThreshold**: consecutive successes required to flip from Unhealthy → Healthy. Must be `1` for liveness and startup probes (API server rejects other values). For readiness, raising this above 1 prevents flapping (pod bouncing in/out of Service endpoints) at the cost of slower recovery.
- **failureThreshold**: consecutive failures before action (restart for liveness/startup, remove-from-endpoints for readiness). Higher values tolerate transient blips at the cost of slower detection of real failures.

**Startup probe sizing example** — a Java app that takes up to 90s to start under worst-case load:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5
  failureThreshold: 18   # 5s * 18 = 90s total allowance
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3     # only kicks in after startupProbe succeeds
```

This gives the app up to 90 seconds to boot without the liveness probe killing it, while liveness stays tight (30s to detect a real hang) once the app is confirmed up.

## Full pod lifecycle

### Phases

`Pending` → `Running` → `Succeeded`/`Failed` (pod-level `status.phase`). Within `Running`, each container independently moves through `Waiting` → `Running` → `Terminated`.

### Init containers

Defined under `spec.initContainers`. Run **sequentially, in the order listed**, each to completion (exit code 0) before the next starts. All init containers must succeed before any app container starts. If an init container fails, the kubelet retries it according to `restartPolicy` (the whole pod is restarted from the first init container on `Always`/`OnFailure`, not just the failed one).

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  - name: run-migrations
    image: myapp-migrate:1.4
    command: ['./migrate', 'up']
  containers:
  - name: app
    image: myapp:1.4
```

- Use cases: schema migrations, waiting for a dependency, populating a shared `emptyDir` volume with config fetched at startup, permission-fixing on mounted volumes (`chown` a volume before the main container runs as non-root).
- Init containers do not support readiness/liveness probes (they're expected to run-to-completion, not stay running), but do support `startupProbe`... actually no — init containers support none of the three probe types at all in practice; their "health" is just their exit code.
- Resource requests/limits on init containers are effectively sequential — the highest single init container request, not summed, is what matters for scheduling (compared against summed app container requests).

### postStart and preStop hooks

```yaml
containers:
- name: app
  image: myapp:1.4
  lifecycle:
    postStart:
      exec:
        command: ["/bin/sh", "-c", "echo Started at $(date) >> /var/log/lifecycle.log"]
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 5 && /app/graceful-shutdown.sh"]
```

- **postStart**: runs immediately after the container is created. Runs asynchronously with the container's ENTRYPOINT — there is **no ordering guarantee** that postStart finishes (or even starts) before your app's main process begins executing. Do not use it for "setup that must happen before the app starts" — use an init container for that instead.
  - If postStart hangs or returns non-zero, the container is killed and restarted — this is a common source of confusing CrashLoopBackOff where the app logs look fine but the pod still restarts.
- **preStop**: runs **before** the container receives SIGTERM. Kubernetes waits for preStop to finish (or timeout) before sending SIGTERM. This is the correct place to implement graceful drain logic that must run before the app starts shutting down, e.g.:
  - `sleep N` to give kube-proxy/Endpoints time to propagate pod removal before the app stops accepting connections (see below — this is the most important preStop pattern).
  - Calling an app-specific shutdown endpoint.
- Both hooks support `exec` and `httpGet` (not `tcpSocket`).
- If a hook errors, times out, or hangs, an event is recorded (`FailedPostStartHook` / `FailedPreStopHook`) and the container is terminated as `Error`.

### Termination sequence (the part everyone gets wrong)

When a pod is deleted (`kubectl delete pod`, rollout, scale-down, node drain):

1. Pod's `status.phase` becomes `Terminating`; the pod is **immediately removed from Service Endpoints/EndpointSlices** by the endpoints controller. This removal is asynchronous relative to step 2 — there is a race.
2. If `preStop` is defined, kubelet runs it now, in parallel with step 1.
3. Kubelet sends `SIGTERM` to PID 1 in each container (either after preStop completes, or immediately if there is no preStop hook).
4. The kubelet waits up to `terminationGracePeriodSeconds` (default 30s) total — this budget covers **both** the preStop hook and the time after SIGTERM.
5. If the container is still running when the grace period expires, kubelet sends `SIGKILL`.

**The race in step 1**: Endpoint removal (informing kube-proxy / Ingress controllers) happens concurrently with SIGTERM delivery, not strictly before it. On a busy cluster, propagating the "pod is gone" update to every kube-proxy and load balancer can take a few seconds. If your app stops accepting connections the instant it receives SIGTERM, you will drop in-flight and freshly-routed requests during that propagation window. This is why a `preStop` hook with a plain `sleep 5`–`sleep 10` is a standard, load-bearing pattern:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```

This delays SIGTERM delivery by 5s, giving the cluster's data plane time to stop sending new traffic to this pod, while the app continues serving the in-flight (and any late-arriving) requests during that window. Combine with an app that gracefully drains on SIGTERM.

### terminationGracePeriodSeconds

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    image: myapp:1.4
    lifecycle:
      preStop:
        exec:
          command: ["/app/drain.sh"]  # must finish within the 60s budget
```

- Default is 30s. Must be large enough to cover: preStop execution + time for the app to finish in-flight requests/transactions after SIGTERM.
- Can be overridden per-delete: `kubectl delete pod x --grace-period=60`.
- `kubectl delete pod x --grace-period=0 --force` skips graceful termination entirely (sends SIGKILL immediately, bypasses the API server's normal deletion confirmation) — only for stuck/unresponsive pods, never for routine restarts, since it causes hard connection drops and can corrupt state for stateful workloads.
- Set high enough for stateful workloads (databases doing checkpoint/flush on shutdown) — some databases need minutes for a clean shutdown under load; undersizing this causes SIGKILL to hit mid-flush and can corrupt data files depending on the storage engine.

### SIGTERM handling — what the app must do

The app's PID 1 must:
1. Catch SIGTERM (default action for un-handled SIGTERM is immediate process termination — no cleanup).
2. Stop accepting new connections/work (close the listening socket, or Deregister first if using a service mesh sidecar).
3. Finish in-flight requests within the remaining grace period.
4. Close DB connections/flush buffers.
5. Exit with code 0.

**Common pitfall — PID 1 signal handling in shell wrappers.** If your container's entrypoint is a shell script (`CMD ["sh", "-c", "node server.js"]`), SIGTERM goes to the shell, not to `node`, and shells commonly do not forward signals to child processes. The app never sees SIGTERM, ignores it entirely until SIGKILL at the grace period boundary — every rollout/scale-down then hard-kills every pod, dropping in-flight requests and adding grace-period-length latency to every deploy.

Fixes:
```dockerfile
# Bad: shell swallows signals
CMD ["sh", "-c", "node server.js"]

# Good: exec form runs the process as PID 1 directly
CMD ["node", "server.js"]
```
Or use a minimal init system that forwards signals and reaps zombies, e.g. `tini`:
```dockerfile
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

### Container restart policy and backoff

`spec.restartPolicy` (pod-level, applies to all containers): `Always` (default, used by Deployments/StatefulSets/DaemonSets), `OnFailure` (used by Jobs — restart only on non-zero exit), `Never` (Jobs that should not retry the same pod).

- On repeated crashes, kubelet applies exponential backoff: 10s, 20s, 40s ... capped at 5 minutes, reset after the container has run successfully for 10 minutes. This is `CrashLoopBackOff` — it is a **status describing the backoff timer**, not the failure itself; check `kubectl describe pod` events and `kubectl logs --previous` for the actual cause.

## Common lifecycle pitfalls — checklist

- **Liveness probe hits a dependency** (DB, downstream API) → one downstream outage kills every replica simultaneously. Liveness should only check the process's own health.
- **No readiness probe** → traffic sent to pods before they've finished warming up; visible as 502/503 spikes right after every deploy or scale-up.
- **Readiness probe too strict / checks external dependency** → cascading total outage when a dependency blips, instead of graceful degradation.
- **`initialDelaySeconds` too short for slow-starting apps, no startupProbe** → restart loop during every startup, worse under load (cold JIT, cache warm-up, config fetch).
- **App doesn't handle SIGTERM** (shell-form CMD swallowing signals) → every scale-down/rollout hard-kills connections at the grace-period boundary, dropping requests and adding latency.
- **No preStop delay before SIGTERM** → requests dropped during the endpoint-propagation race window even if the app handles SIGTERM correctly.
- **`terminationGracePeriodSeconds` too low for the workload** → SIGKILL hits before in-flight work (or a stateful flush/checkpoint) completes.
- **Rolling update readiness never passes** (bad probe path/port after a config change) → rollout hangs forever with `kubectl rollout status` blocking; must `kubectl rollout undo` — nothing auto-rolls-back.
- **postStart used for required setup** → race condition since postStart has no ordering guarantee relative to the main process start; use an init container instead.
- **Probe path collides with an authenticated route** — e.g. `/health` behind an auth middleware that requires a JWT — every probe fails with 401, causing a restart loop; probe endpoints must be unauthenticated (but should not leak sensitive internals).
