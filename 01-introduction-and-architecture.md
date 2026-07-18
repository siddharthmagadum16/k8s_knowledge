# Kubernetes Introduction and Architecture

## 1. What Kubernetes Is and Why It Exists

Kubernetes (K8s) is a container orchestration system: it runs containers across a fleet of machines, keeps them running, and reconciles the running state of the cluster against a desired state that you declare.

### 1.1 The problem before Kubernetes

**VMs era**: Each application ran on its own VM (or bare metal). Provisioning was slow (minutes to hours), resource utilization was poor (a VM sized for peak load sits mostly idle), and scaling meant booting new VMs. Config drift between VMs was common because each was hand-maintained or driven by imperative config management (Puppet/Chef/Ansible scripts run in sequence).

**Plain Docker era**: Containers fixed packaging (an image bundles the app + deps + runtime, runs identically everywhere) and reduced overhead (containers share the host kernel, start in seconds, and are far denser per host than VMs). But Docker alone only runs containers on a single host. It does not answer:

- What happens when a container crashes at 3 AM? (no automatic restart across a fleet)
- What happens when a host dies? (containers on it are gone; nothing reschedules them elsewhere)
- How do you run 50 replicas of a service across 20 hosts and load-balance across them?
- How do you roll out a new version without downtime, and roll back if it's bad?
- How do containers on different hosts find and talk to each other?
- How do you attach persistent storage to a container that might move hosts?
- How do you enforce that a team's containers only get X CPU / Y memory, and isolate them from other teams?

Tools like Docker Swarm and Mesos attempted this before Kubernetes; Kubernetes (derived from Google's internal Borg/Omega systems, open-sourced in 2014) won because of its extensibility model (CRDs, controllers, the API machinery) and ecosystem.

### 1.2 What Kubernetes actually provides

- **Scheduling**: place containers (grouped as Pods) onto machines (Nodes) based on resource availability and constraints.
- **Self-healing**: restart failed containers, reschedule Pods from dead nodes, kill and replace containers that fail health checks.
- **Service discovery and load balancing**: stable DNS names and virtual IPs for a set of replicas, regardless of which physical Pods back them at any moment.
- **Declarative rollouts/rollbacks**: describe the desired version and replica count; Kubernetes performs the transition (e.g., rolling update) and can revert.
- **Storage orchestration**: attach network/cloud storage to Pods, following them across rescheduling within constraints.
- **Secret/config management**: inject configuration and sensitive data without baking it into images.
- **Horizontal scaling**: scale Pod replica counts (and, with more setup, node counts) based on load.
- **Bin packing / multi-tenancy**: many workloads share a pool of machines, each with resource requests/limits, improving utilization versus one-VM-per-app.

Kubernetes does **not** provide by itself: CI/CD pipelines, application-level build tooling, databases-as-a-service, or opinionated PaaS behavior — it's infrastructure plumbing that higher-level tools (Helm, Argo CD, Operators, managed DBs) build on top of.

## 2. Declarative vs Imperative and Desired-State Reconciliation

### 2.1 Imperative model

You tell the system a sequence of *actions*: "start this container", "now stop that one", "now start a replacement". The system's state is whatever the accumulated actions produced. If you did an action out of order or a script failed halfway, the actual state is unknown without inspection. `docker run`, `docker rm`, shell scripts calling the Docker API — all imperative.

### 2.2 Declarative model

You tell the system a *desired end state*: "there should be 3 replicas of this Pod template running, using this image." You submit this as an object (YAML/JSON) to the API. A **controller** (a control loop) continuously compares actual state to desired state and takes actions to close the gap. You never say "start container #4" — you say "3 replicas should exist" and the system figures out what to start or stop.

This is captured in Kubernetes's fundamental building block: every object has a `spec` (desired state, written by you) and a `status` (observed actual state, written by the system).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:          # <- desired state (you write this)
  replicas: 3
  ...
status:        # <- observed state (Kubernetes writes this)
  replicas: 3
  readyReplicas: 3
  availableReplicas: 3
```

### 2.3 Reconciliation loops (the control loop pattern)

Every Kubernetes controller runs the same abstract loop, forever:

```
for {
    observed := getCurrentState()
    desired  := getDesiredState()
    diff     := compare(observed, desired)
    if diff != none {
        takeAction(diff)   // create/delete/update objects to converge
    }
    wait for next trigger (watch event or resync period)
}
```

This is **level-triggered**, not **edge-triggered**: the controller doesn't care about the sequence of events that led to the current state; it only cares what the current state *is* versus what it *should be*, and it re-evaluates this repeatedly (on every relevant watch event, plus a periodic full resync, typically every 30s–10min depending on the controller). This makes the system self-healing and resistant to missed events, restarts, or partial failures — if a controller crashes mid-reconciliation and restarts, it just re-observes state and continues; there's no persisted "step 3 of 5" to recover.

This pattern is used everywhere: ReplicaSet controller (ensure N pod replicas exist), Deployment controller (ensure ReplicaSets match rollout state), Node controller (mark nodes NotReady, evict pods), and every custom Operator you or a vendor writes.

## 3. Full Architecture

A Kubernetes cluster has two categories of machines: **control plane** nodes (the "brain") and **worker nodes** (where your workloads actually run).

```
                         ┌─────────────────────────────────────────────┐
                         │              CONTROL PLANE                  │
                         │                                               │
   kubectl / clients ───▶│  kube-apiserver  ◀────────────────┐          │
                         │        │  ▲                       │          │
                         │        ▼  │                       │          │
                         │      etcd │                       │          │
                         │           │                       │          │
                         │  kube-scheduler   kube-controller- │          │
                         │                    manager          │          │
                         │                   cloud-controller- │          │
                         │                    manager (opt.)   │          │
                         └───────────┬───────────────────────┬┘          │
                                     │ watch/write via API    │
              ┌──────────────────────┼──────────────────────┼─────────┐
              │                      │                      │         │
        ┌─────▼─────┐          ┌─────▼─────┐          ┌─────▼─────┐
        │  Node 1    │          │  Node 2    │          │  Node 3    │
        │ kubelet    │          │ kubelet    │          │ kubelet    │
        │ kube-proxy │          │ kube-proxy │          │ kube-proxy │
        │ CRI runtime│          │ CRI runtime│          │ CRI runtime│
        │ (containerd│          │ (containerd│          │ (containerd│
        │  /CRI-O)   │          │  /CRI-O)   │          │  /CRI-O)   │
        │ Pods...    │          │ Pods...    │          │ Pods...    │
        └────────────┘          └────────────┘          └────────────┘
```

### 3.1 Control plane components

**kube-apiserver**
- The only component that talks to `etcd` directly. Everything else — controllers, scheduler, kubelets, `kubectl` — talks only to the API server.
- Stateless (in the sense that it holds no persistent data itself); you can run multiple replicas behind a load balancer for HA.
- Responsibilities: authentication (certs, tokens, OIDC), authorization (RBAC, ABAC, webhook), admission control (mutating/validating webhooks, e.g. Pod Security admission, resource quota checks), validation of object schemas, and serving the REST/watch API (`/api/v1/...`, `/apis/apps/v1/...`).
- Exposes a **watch** mechanism: clients (controllers, kubelets) open long-lived HTTP connections and receive a stream of change events instead of polling.

**etcd**
- A distributed, strongly consistent key-value store (Raft consensus). This is the **single source of truth** for all cluster state — every object (Pods, Services, Secrets, ConfigMaps, custom resources) is a key under a path like `/registry/pods/<namespace>/<name>`.
- Requires quorum (majority of members alive) to accept writes — typically run as 3 or 5 members for fault tolerance (3 tolerates 1 failure, 5 tolerates 2).
- Losing etcd = losing the cluster's state entirely. Back it up regularly (`etcdctl snapshot save`), and it should live on fast disks (low-latency writes matter for Raft) and ideally isolated from other workloads.
- Nothing else — not the scheduler, not kubelets — talks to etcd directly. This is a deliberate boundary: the API server is the sole gatekeeper, enforcing auth/validation/admission for every write.

**kube-scheduler**
- Watches for Pods with `spec.nodeName` unset (i.e., unscheduled Pods) and assigns them to a Node.
- Runs a two-phase algorithm per Pod:
  1. **Filtering (predicates)**: eliminate nodes that can't run the pod (insufficient CPU/memory, taints without matching tolerations, node selectors/affinity not satisfied, port conflicts, volume topology constraints).
  2. **Scoring (priorities)**: rank remaining feasible nodes (spread pods across nodes/zones, prefer nodes with more free resources or with the image already cached, respect pod anti-affinity, honor `topologySpreadConstraints`).
- Writes the decision back via the API server as a `Binding` (sets `pod.spec.nodeName`); the scheduler never directly tells a kubelet what to do — kubelets discover their assigned pods by watching the API server.
- Pluggable via the **scheduler framework** (extension points: PreFilter, Filter, PostFilter, Score, Reserve, Permit, Bind, etc.) and you can run multiple custom schedulers (`spec.schedulerName`).

**kube-controller-manager**
- Runs many independent controllers as a single process (historically for operational simplicity):
  - Node controller (detects unresponsive nodes, marks `NotReady`, eventually triggers pod eviction after a grace period)
  - ReplicaSet controller (maintains replica counts)
  - Deployment controller (manages ReplicaSet rollouts)
  - Job/CronJob controllers
  - Endpoint/EndpointSlice controller (keeps Service endpoints in sync with matching Pod IPs)
  - Namespace controller, ServiceAccount controller, PV/PVC binding controller, etc.
- Each controller only watches/writes via the API server, following the reconciliation loop pattern described above.

**cloud-controller-manager**
- Splits out cloud-provider-specific logic (AWS/GCP/Azure/etc.) from the core `kube-controller-manager`, so core Kubernetes stays cloud-agnostic.
- Runs: node controller (initialize node with cloud metadata like zone/instance type, detect node deletion from the cloud), route controller (configure inter-node routes for pod networking, on clouds that need it), and service controller (provision a real `LoadBalancer` — e.g., an AWS NLB/ELB — when a Service of `type: LoadBalancer` is created).
- On managed platforms (EKS/GKE/AKS) this is run by the provider and invisible to you; on bare-metal/self-hosted clusters it may be absent or replaced by things like MetalLB for LoadBalancer emulation.

### 3.2 Node (worker) components

**kubelet**
- The agent on every node. Watches the API server for Pods scheduled to its node (`spec.nodeName == <this node>`).
- Ensures the containers described in each Pod spec are actually running, by talking to the local container runtime through the **CRI (Container Runtime Interface)**.
- Runs **liveness/readiness/startup probes**, reports Pod and Node status back to the API server (`status` subresource), manages volume mounting for pods on its node, and reports node-level resource capacity/allocatable amounts.
- Also enforces resource limits by configuring cgroups per container based on the Pod's `resources` field.

**kube-proxy**
- Runs on every node, implements the Service abstraction at the network layer by programming rules (iptables or IPVS) that redirect traffic destined for a Service's ClusterIP to one of the backing Pod IPs.
- Watches Services and EndpointSlices via the API server and updates local rules on changes. Covered in depth in the networking doc.

**Container runtime (via CRI)**
- The actual component that pulls images and starts/stops containers/namespaces/cgroups: **containerd** or **CRI-O** are the common choices today (Docker Engine itself was deprecated as a direct Kubernetes runtime in v1.24 — `dockershim` was removed; containerd, which Docker itself uses under the hood, is used directly instead).
- CRI is a gRPC API (`RuntimeService`, `ImageService`) that decouples kubelet from any specific runtime implementation — kubelet calls `RunPodSandbox`, `CreateContainer`, `StartContainer`, etc., and any CRI-compliant runtime can serve those calls (containerd, CRI-O, gVisor/Kata via containerd shims for stronger isolation).
- The **pod sandbox** concept: before starting app containers, the runtime creates a "pause" container (or equivalent) that holds the shared network namespace (IP address) for all containers in the Pod.

**CNI plugin (not a "core" component but always present)**
- Handles actual Pod networking — assigning IPs, setting up virtual interfaces, and (for many plugins) programming routes/overlays for pod-to-pod connectivity across nodes. Examples: Calico, Cilium, Flannel, AWS VPC CNI. Invoked by the container runtime when a pod sandbox is created.

## 4. End-to-End Flow: `kubectl apply -f deployment.yaml`

Walking a Deployment creation through every hop:

1. **kubectl**: reads the YAML, converts to JSON, determines the target REST resource (`/apis/apps/v1/namespaces/<ns>/deployments`), and either POSTs (create) or PATCHes (if the object exists — `kubectl apply` computes a strategic merge patch against the last-applied-configuration annotation, this is how it differs from `kubectl create`/`replace`).

2. **kube-apiserver** receives the HTTPS request:
   - **Authentication**: validates the client cert / bearer token / OIDC token, determines the user/identity.
   - **Authorization**: checks RBAC (`Role`/`ClusterRole` + bindings) — does this identity have `create`/`patch` on `deployments` in this namespace?
   - **Admission control (mutating)**: mutating webhooks and built-in admission plugins run first (e.g., inject defaults, sidecar injection like Istio's, set default resource requests via `LimitRange`).
   - **Schema validation**: object is validated against the OpenAPI schema for `Deployment`.
   - **Admission control (validating)**: validating webhooks and plugins run (e.g., `ResourceQuota` check, `PodSecurity` admission, OPA/Gatekeeper policy checks).
   - **Persist to etcd**: the API server writes the object as a key under `/registry/deployments/<ns>/<name>` via etcd's Raft-replicated write. Only after etcd acknowledges the write (quorum) does the API server return success to `kubectl`.
   - The API server also emits a **watch event** ("Deployment X created") to every client with an open watch on that resource type.

3. **Deployment controller** (in kube-controller-manager) has a watch open on Deployments. It receives the create event, reconciles: sees no matching ReplicaSet exists for this Pod template hash, so it creates a new `ReplicaSet` object (again via a request to the API server, which goes through the same auth/admission/etcd-write pipeline).

4. **ReplicaSet controller** watches ReplicaSets. It sees the new ReplicaSet wants `replicas: 3` but 0 Pods exist matching its label selector, so it creates 3 `Pod` objects (via the API server). At this point each Pod object exists in etcd with `spec.nodeName` unset — it is "Pending" and unscheduled.

5. **kube-scheduler** watches for Pods with no `nodeName`. For each of the 3 Pods, it runs filtering + scoring across all nodes, picks a node, and issues a `Binding` API call, which sets `pod.spec.nodeName = <chosen node>`. This is itself a write to the API server → etcd.

6. **kubelet** on the chosen node has a watch open (scoped to pods on its own node). It sees a new Pod object bound to itself:
   - Requests the CNI plugin to set up a network namespace/IP for the pod sandbox.
   - Calls the CRI runtime (containerd/CRI-O) to pull the image (if not cached) and create/start the containers defined in the Pod spec, applying resource requests/limits as cgroup settings.
   - Starts running configured probes (startup, then liveness/readiness) against the containers.
   - Continuously reports Pod status (`Running`, container ready states, restart counts) back to the API server, which persists it to etcd.

7. **ReplicaSet/Deployment controllers** observe the Pods transitioning to `Running`/`Ready` via their watches and update `status.readyReplicas` etc. on the Deployment/ReplicaSet objects.

8. **Endpoint/EndpointSlice controller**, if a Service selects these Pods by label, notices new Ready Pod IPs and adds them to the Service's EndpointSlice.

9. **kube-proxy** on every node watches EndpointSlices and updates local iptables/IPVS rules so traffic to the Service's ClusterIP load-balances to the new Pod IPs.

Total round trip: `kubectl` → apiserver → etcd → (watch fanout) → Deployment controller → apiserver → etcd → ReplicaSet controller → apiserver → etcd → scheduler → apiserver → etcd → kubelet → CRI/CNI → running container → status flows back up the same path.

Every single arrow in that chain is a **watch event or an API call through kube-apiserver** — no component calls another directly (kubelet does not call the scheduler; the scheduler does not call kubelet). This is why the API server + etcd is correctly described as the nervous system of the cluster: everything is mediated through it.

## 5. Cluster Topology Options

### 5.1 Single control-plane-node cluster
- One node runs apiserver, etcd, scheduler, controller-manager (often as static pods). Worker nodes join it.
- Fine for learning/dev (`kubeadm`, `kind`, `minikube`, `k3s` single-node). **Not for production** — that one node is a single point of failure for the entire cluster's control operations (existing workloads keep running briefly if it dies, since kubelets/kube-proxy operate independently for a while, but nothing can be scheduled, healed, or changed, and eventually node heartbeats and lease renewals depend on the control plane).

### 5.2 HA control plane
- Run **3 (or 5) etcd members** for Raft quorum, and **≥2 (usually 3) kube-apiserver / scheduler / controller-manager instances**, typically colocated on 3 control-plane nodes.
- apiserver instances are stateless and sit behind a load balancer (cloud LB, or `kube-vip`/HAProxy+keepalived for on-prem) — clients and kubelets point at the LB's VIP, not any one apiserver.
- **Leader election**: scheduler and controller-manager use a `Lease` object in the API to elect a single active leader among their replicas at a time (only one should be actively scheduling/reconciling to avoid conflicting actions); the others sit hot-standby.
- etcd can be **stacked** (co-located on control-plane nodes) or **external** (its own dedicated cluster) — external etcd decouples etcd's lifecycle/scaling from control-plane node lifecycle, recommended for larger/more critical clusters.
- This is what `kubeadm` HA setups, and what managed offerings run internally, look like.

### 5.3 Managed vs self-hosted

**Managed (EKS, GKE, AKS, and similar)**
- The cloud provider runs and fully manages the control plane (apiserver, etcd, scheduler, controller-manager, upgrades, patching, HA, backups) — you don't see or administer those nodes at all; you get an API endpoint.
- You are responsible for: worker nodes (or use serverless node pools like GKE Autopilot/Fargate profiles on EKS to offload that too), workload manifests, RBAC, networking add-ons choice (though a default CNI is often preselected), and cluster version upgrade timing (within provider support windows).
- Cloud-controller-manager integration is built-in (LoadBalancer Services provision real cloud load balancers automatically, `StorageClass` for cloud block storage is preconfigured).
- Trade-off: less operational burden and battle-tested HA, but less control over control-plane configuration (e.g., custom admission plugins, non-default etcd tuning) and you pay a control-plane fee/overhead plus node costs.

**Self-hosted (kubeadm, k3s, RKE2, Talos, hard way)**
- You provision and manage every control-plane component yourself (or via a bootstrapper). Full control over versions, flags, custom builds, air-gapped environments, on-prem/bare-metal.
- `kubeadm`: the standard, relatively low-level bootstrapping tool — you still choose your own CNI, storage, LB solution for the apiserver VIP, etc.
- `k3s`/`RKE2`: lightweight distributions bundling sane defaults (SQLite or etcd datastore option for k3s, a default CNI, simplified install) — popular for edge/IoT and smaller ops teams.
- Talos: an immutable, API-managed OS purpose-built for running Kubernetes (no SSH, config via a declarative machine config), reduces node-level configuration drift.
- Trade-off: full control and no vendor lock-in/fees, but you own upgrades, etcd backups, certificate rotation, HA design, and security patching.

### 5.4 Node pools / multi-AZ considerations
- Production clusters typically spread worker nodes across multiple availability zones; combined with Pod `topologySpreadConstraints` or anti-affinity, this survives an AZ outage.
- Control-plane nodes/etcd members should also be spread across AZs (odd number, e.g. 3 AZs with 1 etcd member each) so a single AZ loss doesn't break Raft quorum.
- Heterogeneous node pools (e.g., a GPU pool, a spot/preemptible pool, a memory-optimized pool) are common; workloads target them via `nodeSelector`/affinity and taints/tolerations (see the scheduling doc).
