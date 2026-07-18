# Observability in Kubernetes

## 1. The three pillars, and why Kubernetes forces all of them

Traditional host-based monitoring assumes stable identity: a box has a hostname, an IP, a disk, and it lives for months. Nagios pings it, syslog writes to `/var/log/messages`, you SSH in when something breaks. That model quietly assumes three things that Kubernetes breaks on purpose:

- **Identity is stable.** In K8s, a Deployment rolling update replaces every pod. Pod IPs are allocated from a CNI pool and reused within minutes of a pod dying. A hostname-based alert ("host web-03 is down") is meaningless when web-03 never existed as a concept — there's a Deployment, and it currently happens to be running on pods `web-7d4f9c8b6-x2k9p` and `web-7d4f9c8b6-9mqwt`, neither of which existed an hour ago.
- **The filesystem is stable.** In K8s, unless you mount a persistent volume, the container filesystem — including anything a process wrote to a local log file — is gone the moment the container is recreated. `kubectl logs` after a restart won't show you the previous instance's output unless you pass `--previous`, and even that is only available while the old pod object still exists in etcd (it gets garbage collected).
- **One process per host.** In K8s, a single node runs dozens of unrelated containers, often multiple containers in a single pod (app + sidecar), each with separate resource accounting, separate log streams, separate lifecycle. `top` on the node tells you almost nothing about which container is responsible for a load spike.

Given that, here's why you need all three pillars and not two:

**Metrics without logs**: your dashboard shows p99 latency jumped from 40ms to 4s at 14:32 and error rate went from 0.1% to 12%. That tells you *something* broke and *when*. It does not tell you *why*. Metrics are aggregates — you cannot ask a counter "what was the actual error message for request X." You're stuck grep-ing through whatever logs you can find, hoping the failing pod hasn't been rescheduled and its logs haven't rotated away.

Concrete incident: a payment service's error-rate metric alarms at 03:14. The Grafana panel shows `rate(http_requests_total{status=~"5.."}[5m])` spiking. Nothing in the metrics tells you if this is one bad pod, a downstream dependency timing out, a config change, or a bad deploy. You have the "what" and "when" but not the "why" — you need logs to see the actual stack trace or error payload.

**Logs without metrics**: you have exhaustive text for every request, but no way to know the *shape* of the problem across the fleet. Is 1 pod out of 40 throwing errors, or all 40? Is this new in the last 10 minutes or has it been happening at low volume for a week? Grepping logs for "500" across 40 pods streamed into Loki/ES can approximate this, but it's expensive and slow compared to a pre-aggregated counter. Without metrics you also have no early-warning signal — you find out about degradation only when someone notices or a log-based alert (usually noisy and lagging) fires.

**Logs and metrics without traces**: this is the one that actually stalls senior engineers on real incidents. Say `checkout-service` calls `inventory-service` calls `pricing-service` calls a Postgres RDS instance. Latency alert fires on `checkout-service` p99. You go to `checkout-service` logs: nothing obviously wrong, it's just "waiting on downstream call, took 2.8s". You now have to guess which downstream call, open `inventory-service` logs, correlate by approximate timestamp (they don't line up exactly because of clock skew and buffering), find a slow call there, then repeat for `pricing-service`. In a system with 4-5 hops this manual correlation-by-eyeball can eat an hour of an incident, especially at 3am when the person on-call didn't build the request-hop map in their head. A trace collapses this to one click: open the trace ID from the checkout log line, see all 5 spans, see instantly that span 4 (Postgres query in pricing-service) took 2.6s of the 2.8s total. Diagnosis time: seconds, not an hour.

The rule that falls out of this: metrics tell you something is wrong and roughly how bad; logs tell you the specific detail of one instance of wrong; traces tell you where in a multi-hop call chain the time/error actually originated. K8s' ephemerality means you cannot fall back on "SSH in and look around" as a fourth pillar — the evidence you need has to already be shipped somewhere durable before the pod disappears.

## 2. Metrics pipeline deep dive

### 2.1 metrics-server

`metrics-server` is **not** a monitoring system. It's an in-memory aggregator with zero persistence and zero history. It scrapes every kubelet's `/metrics/resource` endpoint (older versions used `/stats/summary`) every ~60 seconds (`--kubelet-request-timeout`, `--metric-resolution` flags), holds the latest data point per pod/node in memory, and exposes it via the Kubernetes **aggregated API** at `metrics.k8s.io/v1beta1`.

Consumers:
- `kubectl top nodes` / `kubectl top pods` — reads directly from this API.
- The **HPA controller** — for `type: Resource` metrics (cpu/memory), it queries `metrics.k8s.io` through the same aggregated API path.

Why `kubectl top` sometimes shows nothing (or `<unknown>`) for a few minutes after metrics-server restarts: it holds no history. On restart, its in-memory cache is empty. It has to wait for at least one full scrape cycle across all nodes to populate data, and some client libraries/HPA logic want two consecutive samples to compute a rate — so you can see a real gap of 1-2 scrape intervals (often 30-60s, sometimes longer under load or if `--kubelet-insecure-tls` / certificate issues cause partial scrape failures against some nodes). Check with:

```bash
kubectl get apiservices | grep metrics
kubectl logs -n kube-system deploy/metrics-server
kubectl top nodes
```

If it never recovers, the usual cause is metrics-server can't reach kubelet's HTTPS endpoint (cert SAN mismatch, or the pod lacks `hostNetwork`/correct CA bundle) — check for `x509` errors in the metrics-server log.

### 2.2 Prometheus

Prometheus is fundamentally a **pull-based** time series database with real retention. Instead of processes pushing metrics somewhere, Prometheus scrapes HTTP `/metrics` endpoints on a schedule and stores every sample on local disk (default 15 days retention, configurable, or shipped via `remote_write` to long-term storage like Thanos/Mimir/Cortex).

Service discovery in K8s is what makes this scale without hand-maintained target lists. Prometheus's `kubernetes_sd_configs` can discover targets from four roles:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # only scrape pods that opt in via annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

`role: endpoints`, `role: service`, and `role: node` give you other views of the same discovery (endpoints role is common for scraping Services that fan out to multiple pod backends). The `prometheus.io/scrape: "true"` annotation convention above is how most Helm charts opt workloads into scraping without you hand-editing the scrape config every time a new service is deployed.

This is a fundamentally different tool from metrics-server: it has actual history (you can query "what was CPU usage 6 hours ago"), supports arbitrary custom application metrics (not just cpu/memory), and its query language (PromQL) lets you do rates, aggregations, joins across metrics. metrics-server cannot do any of that — it is deliberately minimal because its only job is to feed `kubectl top` and HPA cheaply.

### 2.3 kube-state-metrics — object state, not resource usage

This is the single most confused exporter for engineers new to K8s observability, so get this straight: **kube-state-metrics (KSM) talks to the kube-apiserver, not the kubelet, not cAdvisor.** It watches K8s object state (via list-watch, same as a controller) and turns object *fields* directly into metrics. It has no idea how much CPU a container is actually burning — it only knows what the API server's stored object says.

Concrete metrics and what they actually mean:

```
kube_pod_status_phase{pod="web-7d4f9c8b6-x2k9p", phase="Running"} 1
kube_pod_status_phase{pod="web-7d4f9c8b6-x2k9p", phase="Pending"} 0
kube_deployment_status_replicas_unavailable{deployment="web"} 2
kube_deployment_spec_replicas{deployment="web"} 10
kube_pod_container_status_restarts_total{pod="web-7d4f9c8b6-x2k9p", container="app"} 4
kube_pod_container_status_last_terminated_reason{pod="...", container="app", reason="OOMKilled"} 1
kube_node_status_condition{node="ip-10-0-1-23", condition="Ready", status="true"} 1
kube_node_status_condition{node="ip-10-0-1-23", condition="DiskPressure", status="true"} 1
kube_persistentvolumeclaim_status_phase{claim="data-pg-0", phase="Pending"} 1
```

If a pod is stuck in `Pending` because there's no node with enough allocatable CPU, KSM is what tells you that (`kube_pod_status_phase{phase="Pending"}` plus `kube_pod_status_unschedulable`) — cAdvisor/node-exporter have nothing to say here since the pod's container never started. Conversely, if you want to know "which container was OOMKilled and when," `kube_pod_container_status_last_terminated_reason` combined with `kube_pod_container_status_restarts_total` is your primary signal, because the Linux OOM killer event itself isn't a Prometheus metric anywhere else by default — KSM surfaces it because the API server records the termination reason in pod status.

### 2.4 cAdvisor — container resource usage

cAdvisor is built directly into the kubelet binary (as a library, not a separate process, since K8s 1.12+) and exposes container-level resource metrics scraped at `https://<node>:10250/metrics/cadvisor`. This is real usage data pulled from Linux cgroups — actual CPU time consumed, actual memory pages resident, actual filesystem bytes used per container.

```
container_cpu_usage_seconds_total{pod="web-7d4f9c8b6-x2k9p", container="app"} 143.2
container_memory_working_set_bytes{pod="web-7d4f9c8b6-x2k9p", container="app"} 187342848
container_fs_usage_bytes{pod="web-7d4f9c8b6-x2k9p", container="app"} 52428800
container_cpu_cfs_throttled_periods_total{pod="...", container="app"} 812
```

cAdvisor knows nothing about Deployments, ReplicaSets, or pod phase — it only knows cgroups on the node it runs on. It cannot tell you a pod is Pending (it never ran there) and it doesn't know "desired replica count." That's KSM's job.

### 2.5 node-exporter — host/OS-level metrics

node-exporter runs as a DaemonSet and exposes metrics about the underlying Linux host that cAdvisor does not — because cAdvisor's scope is cgroups/containers, not the whole machine.

```
node_filesystem_avail_bytes{device="/dev/nvme0n1p1", mountpoint="/"} 8589934592
node_memory_MemAvailable_bytes 2147483648
node_network_receive_bytes_total{device="eth0"} 98432104213
node_load1 4.32
node_disk_io_time_seconds_total{device="nvme0n1"} 1832.4
```

This is where you catch node-level problems that no per-container metric shows: the node's root disk filling up from container image layers or log accumulation (which triggers kubelet `DiskPressure` and evictions), a NIC saturating at the host level across all pods combined, host-level memory pressure from non-cgrouped processes (systemd services, kubelet itself, containerd).

### 2.6 Why you need all four, and what each answers

None of these substitute for another — they observe different layers (API object state vs. container cgroup vs. host OS) and answering a debugging question usually requires picking the right one, or joining across two.

| Question | Exporter | Example metric |
|---|---|---|
| Is this Deployment actually running the desired replica count? | kube-state-metrics | `kube_deployment_status_replicas_unavailable` |
| Why is this pod stuck Pending? | kube-state-metrics | `kube_pod_status_phase{phase="Pending"}`, events |
| Was this pod OOMKilled, and how many times? | kube-state-metrics | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` |
| How much memory is this specific container actually using right now? | cAdvisor | `container_memory_working_set_bytes` |
| Is this container being CPU throttled? | cAdvisor | `container_cpu_cfs_throttled_periods_total` |
| Is the node's root disk about to fill up? | node-exporter | `node_filesystem_avail_bytes` |
| Is there host-level memory pressure outside any single container's cgroup? | node-exporter | `node_memory_MemAvailable_bytes` |
| Is a node marked NotReady or under DiskPressure/MemoryPressure? | kube-state-metrics | `kube_node_status_condition` |
| What's the network throughput on the node's physical NIC? | node-exporter | `node_network_receive_bytes_total` |
| How many restarts has this container had over the last day? | kube-state-metrics | `kube_pod_container_status_restarts_total` |
| Is a PVC stuck unbound? | kube-state-metrics | `kube_persistentvolumeclaim_status_phase` |

A concrete failure mode this table prevents: an engineer sees pods restarting and tries to find "restart count" in cAdvisor metrics — it doesn't exist there, because cAdvisor doesn't track pod-level lifecycle, only current cgroup stats for whatever's running right now. It's a KSM metric because restart count is API object state, not a live resource reading.

## 3. PromQL essentials for Kubernetes

### 3.1 rate() and the counter-reset gotcha

`container_cpu_usage_seconds_total` and similar `_total` suffixed metrics are **counters** — monotonically increasing (in theory). Never graph or alert on a raw counter value; it's cumulative since the container process started and tells you nothing about current rate of change. Always wrap in `rate()` (for alerting/smoothed trends, over a window of at least 4x your scrape interval) or `irate()` (for graphing volatile fast-changing rates, uses only the last two samples).

```promql
# correct - CPU cores consumed per second, averaged over 5m
rate(container_cpu_usage_seconds_total{pod=~"web-.*"}[5m])

# wrong - meaningless, ever-increasing since container start
container_cpu_usage_seconds_total{pod=~"web-.*"}
```

The counter-reset gotcha: when a pod restarts (crash, OOM, redeploy), its counter resets to 0 — a brand new container process starts with a fresh cgroup and cAdvisor reports from zero. If you computed rate manually as `(value_now - value_5_min_ago) / 300`, a restart mid-window would give you a large *negative* number, which is nonsense. `rate()` in PromQL specifically detects counter resets (a sample lower than the preceding one) and handles it by treating the reset point as if the counter continued from 0, extrapolating correctly rather than returning garbage. This is why `rate()` exists as a dedicated function instead of you hand-rolling deltas — do not reimplement this logic yourself, and never use `rate()` on gauges (things like `container_memory_working_set_bytes` are gauges, not counters — using rate() on a gauge is a common mistake that produces meaningless output).

### 3.2 working_set_bytes vs usage_bytes — the OOM prediction trap

This is a real, common false alarm. `container_memory_usage_bytes` includes the page cache — file-backed pages the kernel is caching that can be reclaimed instantly under memory pressure (e.g. content read from disk, log buffers). `container_memory_working_set_bytes` is `usage_bytes` minus the kernel's `inactive_file` reclaimable pages — it's the number the **OOM killer and the kubelet eviction manager actually act on**.

Concrete scenario: a container has a memory limit of 512Mi. It's steadily writing ~150Mi of application heap but has also read ~300Mi of static asset files from disk over its lifetime, which the kernel is holding in page cache because there's spare memory and no reason to evict it yet.

```
container_memory_usage_bytes{pod="web-x"}          -> 460Mi   (looks like it's about to OOM!)
container_memory_working_set_bytes{pod="web-x"}     -> 165Mi   (actually nowhere near the 512Mi limit)
```

If you alert on `usage_bytes` approaching the limit, you get paged constantly for containers that are perfectly healthy — the kernel will happily drop that reclaimable cache the instant it needs the memory back, well before an OOM kill would ever happen. The correct query for OOM-risk prediction always uses `working_set_bytes`:

```promql
# correct - fraction of memory limit actually at OOM risk
container_memory_working_set_bytes{container!=""}
  /
container_spec_memory_limit_bytes{container!=""}
  > 0.9
```

### 3.3 CPU throttling — the graph lies

This is the gap that separates senior from junior debugging almost every time. A CPU usage graph can show a container comfortably under its CPU limit — say averaging 60% of its 500m limit — while the container is *still being throttled* and suffering real added latency. Here's why: the kernel's CFS (Completely Fair Scheduler) bandwidth controller enforces the CPU limit in fixed **100ms accounting periods** (`cfs_period_us`, default 100000). If your container's limit is 500m (0.5 cores), it gets 50ms of CPU time to spend within each 100ms period. If your workload is bursty — say it needs to do a burst of parallel work for 30ms out of every 100ms window, but that burst wants 3 cores' worth of parallel threads for those 30ms — it can blow through its 50ms allowance in the first 15ms of the period and then sit throttled, unable to run at all, for the remaining 85ms, even though its *average* usage over a full second looks like nothing to worry about. Averaging over 1s or 5s completely hides this 100ms-granularity bursting.

The two counters to compare:

```
container_cpu_cfs_periods_total{pod="...", container="app"}      # total 100ms accounting periods elapsed
container_cpu_cfs_throttled_periods_total{pod="...", container="app"}  # periods where the container hit its quota and was throttled
```

Throttling ratio:

```promql
rate(container_cpu_cfs_throttled_periods_total{container="app"}[5m])
  /
rate(container_cpu_cfs_periods_total{container="app"}[5m])
```

A value of 0.3 means the container was throttled in 30% of its scheduling periods over the last 5 minutes — real, user-facing added latency (requests queueing behind throttle waits) that will not show up as elevated average CPU usage. This is the classic "CPU graph looks fine but p99 latency is bad" incident. Fix options: raise the CPU limit (or remove it and rely on requests + node-level bin packing), reduce thread/goroutine parallelism inside the container so bursts don't need more cores than the quota allows within one period, or in extreme cases lower `cfs_period_us` isn't something you control per-container without node-level kubelet config changes, so in practice the fix is almost always "give it more CPU limit or fewer concurrent threads."

### 3.4 Worked alert rules

OOM risk (uses working_set, not usage):

```yaml
groups:
  - name: memory.rules
    rules:
      - alert: ContainerMemoryNearLimit
        expr: |
          container_memory_working_set_bytes{container!="", container!="POD"}
            /
          container_spec_memory_limit_bytes{container!="", container!="POD"} > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.pod }}/{{ $labels.container }} is at {{ $value | humanizePercentage }} of its memory limit"
          description: "Working set has been above 85% of the memory limit for 10m. OOM kill risk if this keeps climbing."
```

CPU throttling:

```yaml
      - alert: ContainerCPUThrottlingHigh
        expr: |
          (
            rate(container_cpu_cfs_throttled_periods_total{container!="", container!="POD"}[5m])
              /
            rate(container_cpu_cfs_periods_total{container!="", container!="POD"}[5m])
          ) > 0.25
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.pod }}/{{ $labels.container }} throttled {{ $value | humanizePercentage }} of periods"
          description: "This container is CPU throttled in over 25% of CFS periods. Average CPU usage graphs will look fine; latency is still degraded. Consider raising cpu limit or reducing concurrency."
```

## 4. Custom Metrics API and External Metrics API

HPA out of the box only scales on `type: Resource` (cpu/memory) via metrics-server. To scale on anything else — queue depth, requests-per-second, Kafka consumer lag, business metrics — you need an **adapter** that implements the Kubernetes custom-metrics or external-metrics aggregated API and translates HPA's queries into queries against your actual metrics backend.

**prometheus-adapter** is the standard implementation: HPA calls `custom.metrics.k8s.io` or `external.metrics.k8s.io`, prometheus-adapter receives that call, translates it into a PromQL query against Prometheus per a rules config you write, and returns the result in the shape HPA expects.

```
HPA controller --> custom.metrics.k8s.io / external.metrics.k8s.io (aggregated API)
                        |
                   prometheus-adapter (translates request -> PromQL)
                        |
                    Prometheus (executes the actual query)
```

`type: Pods` metrics are metrics tied to specific pods in the scaled workload (e.g. `http_requests_per_second` exposed per-pod and averaged). `type: External` metrics come from something outside any Kubernetes object entirely (an SQS queue depth, a Kafka topic lag, a metric from a SaaS API) — there's no pod to attribute it to, so HPA just takes the raw value.

External metric example — scale workers on SQS queue depth:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sqs-worker
  namespace: processing
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sqs-worker
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels:
              queue: order-processing
        target:
          type: AverageValue
          averageValue: "30"   # aim for ~30 messages in queue per replica
```

This assumes prometheus-adapter (or a dedicated SQS exporter feeding Prometheus, or KEDA's SQS scaler) is configured with a rule mapping `sqs_queue_depth{queue="order-processing"}` to a PromQL query against whatever metric your SQS exporter publishes (`aws_sqs_approximate_number_of_messages_visible`).

Pods metric example — scale on requests-per-second per pod:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend
  namespace: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"   # scale to keep ~100 req/s per pod
```

Matching prometheus-adapter rule (goes in the adapter's ConfigMap, not the HPA):

```yaml
rules:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "http_requests_total"
      as: "http_requests_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

Note the failure mode people hit here: if the adapter rule's `metricsQuery` window (`[2m]` above) doesn't line up with your actual traffic patterns, HPA can see a stale or spiky signal and thrash replicas up and down. Also remember `type: Pods` metrics are averaged across all pods currently in the target — a single misbehaving pod skews the average and can mask a per-pod problem that Pods-type autoscaling won't catch.

## 5. Logging architecture

### 5.1 Node-level agent model

The standard pattern is a **DaemonSet** (Fluent Bit, Fluentd, or Vector) running one instance per node, which tails log files on the node's disk — it does **not** connect to the container's stdout stream directly, and it does not talk to the container runtime's API to pull logs live. Here's the actual path:

```
container process writes to stdout/stderr
   -> container runtime (containerd/CRI-O) captures it
   -> written to /var/log/pods/<namespace>_<pod>_<uid>/<container>/<n>.log (JSON-lines format for containerd)
   -> symlinked at /var/log/containers/<pod>_<namespace>_<container>-<containerID>.log
   -> Fluent Bit DaemonSet tails /var/log/containers/*.log from the HOST filesystem
        (via a hostPath volume mount, NOT a socket to the container)
   -> parses, enriches with k8s metadata (namespace, pod, labels via the kubernetes filter
        which calls the API server / kubelet to attach metadata)
   -> ships to Loki / Elasticsearch / a SaaS log backend
```

This matters operationally: because the agent reads from disk, if the node itself goes down hard (kernel panic, host crash) before the agent flushes its last read/send cycle, you can lose the last few seconds/KB of logs that were written but never shipped. It also means logs are inherently subject to the node's own log rotation — which is the next problem.

### 5.2 Log rotation and the silent-loss trap

Kubelet manages container log rotation itself via flags (or the equivalent `KubeletConfiguration` fields):

```yaml
# kubelet config (or /var/lib/kubelet/config.yaml)
containerLogMaxSize: "10Mi"
containerLogMaxFiles: 5
```

This means each container gets at most 5 rotated files of 10Mi each — 50Mi of log retention on disk, total, per container, regardless of how chatty it is. If a container logs heavily (e.g. debug-level logging turned on during an incident, or a retry loop spamming errors), it can burn through 50Mi in well under a minute. If your shipping agent's tail/read interval, or a temporary backlog (agent restart, backend outage, network partition to your log backend) takes longer than that to catch up, the oldest rotated file gets deleted before Fluent Bit ever reads it — **the logs are gone, with no error, no warning, nothing in `kubectl logs`.** This is one of the more insidious silent failure modes in K8s logging: everything looks like it's working (agent is running, no crash, no alert) and yet a chunk of logs from exactly the window you care about (the incident) never made it out.

Diagnosis when you suspect this happened: check `containerLogMaxSize`/`containerLogMaxFiles` on the node, check Fluent Bit's own metrics (`fluentbit_input_files_read_total`, retry/error counters, `fluentbit_output_retries_total`) around the incident window, and check if the backend ingestion had a gap or backpressure event at the same time. The fix is either increasing rotation size/file count (trades node disk usage) or, better, reducing agent read/flush latency and adding buffering (Fluent Bit's `storage.type filesystem` for its own local buffer) so a transient backend outage doesn't cause loss upstream of rotation.

### 5.3 Structured (JSON) logging

Emitting logs as JSON lines instead of free-text is not a style preference at scale — it's what makes Fluent Bit/Fluentd parsing reliable instead of brittle. A printf-style log line:

```
2026-07-18 03:14:22 ERROR Failed to process order 48213 for user 9931: timeout after 3 retries
```

requires a regex or grok pattern to extract `order_id`, `user_id`, `retry_count` — and that regex breaks the moment a developer tweaks the message wording, adds a field, or a message contains an unexpected character (embedded newlines from a stack trace being the classic killer — it silently splits into multiple "log lines" downstream unless multiline parsing is explicitly configured). The structured equivalent:

```json
{"timestamp":"2026-07-18T03:14:22Z","level":"error","msg":"failed to process order","order_id":48213,"user_id":9931,"retry_count":3,"trace_id":"4bf92f3577b34da6a3ce929d0e0e4736"}
```

parses deterministically regardless of message text changes, indexes cleanly as separate fields in Loki/Elasticsearch (so you can filter `order_id=48213` directly instead of full-text-searching), and — critically for the next section — carries a `trace_id` field as a first-class value rather than something a regex has to fish out of free text.

### 5.4 Correlating logs across services

Once you have more than one service in the request path, you need a shared identifier threaded through every log line touched by that request. This is normally a `trace_id` (if using OpenTelemetry/W3C trace context) or a hand-rolled `request_id` generated at the edge (ingress/gateway) and propagated via a header (`X-Request-ID`) that every downstream service reads out of the incoming request context and re-emits into its own logs and re-forwards on its own outbound calls.

```
edge/ingress generates X-Request-ID: 7e4a...
  -> service A logs {"request_id":"7e4a...", "msg":"received request"}
  -> service A calls service B, forwarding header X-Request-ID: 7e4a...
  -> service B logs {"request_id":"7e4a...", "msg":"querying db"}
```

With this in place, `{request_id="7e4a..."}` in Loki (or equivalent) pulls every log line across every service and pod involved in that one request, in order, regardless of which pod handled which hop. Without it, you're back to correlating by approximate timestamp across services with independent clocks and independent log buffering delays — unreliable and slow, exactly the "logs without traces" problem from section 1.

## 6. Distributed tracing basics

Once a request crosses more than one service boundary, per-service logs and metrics stop being sufficient to answer "where did the time/error actually happen" without manual reconstruction. Tracing exists specifically to make the call graph and per-hop timing explicit and queryable.

**OpenTelemetry (OTel)** is the current standard. Instrumentation comes in two forms:

- **Auto-instrumentation**: language-specific agents (Java agent, Python `opentelemetry-instrument` wrapper, .NET profiler) hook into common libraries (HTTP clients/servers, gRPC, common DB drivers) and create spans automatically for those calls, with zero code changes. This gets you spans for "HTTP call to service B" and "SQL query to Postgres" for free.
- **Manual instrumentation**: you explicitly create spans in your business logic using the OTel SDK (`tracer.start_span("validate_order")`) to capture logic that auto-instrumentation can't see — a loop doing 3 sequential validation steps, a cache lookup, a custom retry wrapper.

Trace context propagation is what actually links spans across service boundaries into one trace. The W3C standard header is `traceparent` (plus optional `tracestate` for vendor-specific extensions):

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ version
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ trace-id (32 hex chars, shared across the whole request)
                                                  ^^^^^^^^^^^^^^^^ parent-span-id (16 hex chars)
                                                                     ^^ trace-flags (01 = sampled)
```

Zipkin's older **B3** propagation format (`X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled`, or the single-header `b3` variant) is still common in systems that predate W3C standardization or use Zipkin-native tooling; OTel SDKs can be configured to emit and accept either.

Service mesh sidecars (Istio's Envoy proxy, Linkerd's proxy) sit in the data path for every inbound/outbound call and **can** automatically propagate an existing `traceparent` header end to end, and can even generate their own spans representing the network hop (proxy-to-proxy latency, TLS handshake time, retries the mesh itself performed). This is genuinely useful — you get network-level spans without touching app code. But there's a hard limit: **the mesh cannot see inside your process.** It has no idea that your handler spent 200ms doing a database query, calling a cache, and running business logic before responding — from the mesh's point of view that's one opaque span "time spent waiting for a response from this pod." If you don't instrument your own code (auto or manual), your traces will show accurate network-hop timing between services but a black box for everything that happens inside each service. Teams sometimes deploy Istio, expect end-to-end traces "for free," and are surprised when the trace shows 3 seconds unaccounted for inside a single service span with nothing underneath it — the mesh did its part; the app was never instrumented.

## 7. SLOs, SLIs, and alerting on symptoms not causes

An **SLI** (Service Level Indicator) is a precisely defined, measurable ratio — not a vague notion of "is it fast." A usable SLI definition:

> The proportion of HTTP requests to `checkout-service` that complete in under 300ms AND do not return a 5xx, measured over a rolling 28-day window.

As PromQL (assuming a histogram `http_request_duration_seconds_bucket` and a counter `http_requests_total`):

```promql
# good events: fast AND not 5xx
sum(rate(http_request_duration_seconds_bucket{service="checkout-service", le="0.3"}[5m]))
  -
sum(rate(http_request_duration_seconds_bucket{service="checkout-service", le="0.3", status=~"5.."}[5m]))
  /
sum(rate(http_requests_total{service="checkout-service"}[5m]))
```

(In practice you'd track "fast" and "not error" as two separate SLIs, or precompute a single "good/total" ratio from an app-level metric, since combining a latency histogram and status code cleanly in one PromQL expression like this is fiddly — the point is the SLI is a precise good-events-over-total-events ratio, not a feeling.)

An **SLO** sets a target on that SLI: "99.9% of requests over 28 days meet the SLI." That gives you an **error budget**: 0.1% of requests over 28 days are allowed to fail the SLI before you've burned the whole budget. If your traffic is ~10M requests/day, 0.1% over 28 days is roughly 280,000 "bad" request-equivalents you can spend before you've blown the SLO for the period — burn it too fast and you should be freezing risky deploys, not shipping more features.

**Burn rate alerting** (the Google SRE workbook multi-window multi-burn-rate approach) is the practical alerting mechanism built on top of an error budget. The idea: alert with high urgency when you're burning the budget fast enough to exhaust it in hours, and alert with lower urgency when you're burning it slowly enough that it'll take days — using two time windows per severity to avoid both false positives (a 2-minute blip) and false negatives (a slow leak that a short window never crosses threshold on).

```yaml
groups:
  - name: checkout-slo-burn-rate
    rules:
      # fast burn: would exhaust a 30-day budget in ~2 hours if sustained
      # requires both a 5m AND a 1h window to both show high burn rate (reduces flapping)
      - alert: CheckoutSLOFastBurn
        expr: |
          (
            checkout_error_budget_burn_rate_5m > 14.4
            and
            checkout_error_budget_burn_rate_1h > 14.4
          )
        labels:
          severity: page
        annotations:
          summary: "checkout-service burning error budget 14.4x normal rate - budget exhausted in ~2h if sustained"

      # slow burn: would exhaust the budget in ~3 days if sustained
      - alert: CheckoutSLOSlowBurn
        expr: |
          (
            checkout_error_budget_burn_rate_30m > 6
            and
            checkout_error_budget_burn_rate_6h > 6
          )
        labels:
          severity: ticket
        annotations:
          summary: "checkout-service burning error budget 6x normal rate - budget exhausted in ~3d if sustained, investigate during business hours"
```

(`checkout_error_budget_burn_rate_5m` etc. would be recording rules precomputing `(1 - SLI) / (1 - SLO)` over each window — the constants 14.4 and 6 come directly from the SRE workbook's standard table for a 30-day, 99.9% SLO with 2%/5% budget consumption targets.)

The core principle this all serves: **alert on symptoms the user experiences, not on internal causes.** A pod restarting, a node's CPU at 90%, a single replica being briefly unavailable — these are *possible causes* of a symptom, and pods restart routinely as a completely normal, self-healing part of how Kubernetes operates (rolling deploys, a node draining, a liveness probe catching a transient hang). None of that is inherently harmful to a user, and none of it is actionable at 3am beyond "go look at the dashboard" which the burn-rate alert already tells you to do only when it actually matters.

**Bad alert:**

```yaml
- alert: PodRestarted
  expr: increase(kube_pod_container_status_restarts_total[5m]) > 0
  labels:
    severity: page
```

This pages a human every time any pod restarts for any reason, including a completely benign rolling deploy or a liveness probe correctly killing a hung process before it caused user impact. It has no relationship to whether users were actually affected, it will page constantly in a healthy cluster doing routine deploys, and it trains the on-call engineer to ignore pages — the worst outcome for an alerting system.

**Good alert:** the burn-rate rule above. It only fires when the actual user-facing SLI (latency + error rate on real traffic) is degrading badly enough, for long enough, to matter against the commitment you made. A pod restart that doesn't move the SLI doesn't page anyone; a pod restart that does (e.g. it's restart-looping and taking real capacity out from under real traffic) will show up as increased latency/errors and get caught by the symptom-based alert anyway — you don't need a separate cause-based alert to catch it, and you get the benefit of not being paged for the 95% of restarts that never affected a single user.
