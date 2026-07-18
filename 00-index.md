# Kubernetes: Complete Knowledge Base

A from-scratch to senior-infra-engineer reference. Part 1 covers what every Kubernetes user needs to know. Part 2 covers what separates a senior infra/DevOps engineer from everyone else — internals, failure modes, and production judgment.

## 📊 Visual architecture map

Start here for the mental model: **[k8s-architecture-map.html](k8s-architecture-map.html)** — seven interactive Mermaid diagrams (one worked example: an e-commerce company on EKS) with **hover tooltips on every component** and numbered flows. Open the HTML in a browser for the full experience; the standalone `.mmd` sources and details are in [diagrams/](diagrams/00-diagrams-index.md).

## Part 1 — Fundamentals

1. [Introduction and Architecture](01-introduction-and-architecture.md) — why Kubernetes exists, control plane and node components, request flow
2. [Pods and Workloads](02-pods-and-workloads.md) — Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs/CronJobs
3. [Networking, Services, Ingress](03-networking-services-ingress.md) — Service types, kube-proxy, CoreDNS, Ingress, basic NetworkPolicy
4. [Configuration and Storage](04-configuration-storage.md) — ConfigMaps, Secrets, Volumes, PV/PVC, StorageClasses, CSI
5. [Scheduling and Resource Management](05-scheduling-and-resource-management.md) — requests/limits, QoS classes, ResourceQuotas, HPA basics
6. [Health Checks and Lifecycle](06-health-checks-and-lifecycle.md) — probes, init containers, graceful shutdown
7. [Namespaces, RBAC, Security Basics](07-namespaces-rbac-security-basics.md) — namespaces, ServiceAccounts, RBAC, Pod Security Standards
8. [Helm and Package Management](08-helm-and-package-management.md) — charts, templating, releases, Kustomize
9. [kubectl and Cluster Interaction](09-kubectl-and-cluster-interaction.md) — kubeconfig, command patterns, server-side apply

## Part 2 — Senior / Infra-DevOps Depth

10. [Networking Deep Dive](10-networking-deep-dive.md) — CNI internals, kube-proxy internals, DNS chain, service mesh, MTU issues
11. [etcd and Control Plane Internals](11-etcd-and-control-plane-internals.md) — Raft, backup/restore, API server request flow, admission control, reconciliation loops
12. [Advanced Scheduling](12-advanced-scheduling.md) — affinity/anti-affinity, taints/tolerations, topology spread, priority/preemption, descheduler
13. [Troubleshooting and Debugging](13-troubleshooting-and-debugging.md) — systematic methodology, CrashLoopBackOff, OOMKilled, Pending pods, Service unreachable, Node NotReady
14. [Security Hardening](14-security-hardening.md) — Pod Security Admission, securityContext, seccomp/AppArmor, NetworkPolicy patterns, secrets maturity, supply chain security
15. [Observability](15-observability.md) — metrics pipeline, PromQL for K8s, logging architecture, tracing, SLOs
16. [Custom Resources and Operators](16-custom-resources-and-operators.md) — CRDs, operator pattern, controller-runtime, reconcile loop design
17. [Cluster Lifecycle, Upgrades, DR](17-cluster-lifecycle-upgrades-and-dr.md) — version skew, upgrade strategy, PDBs, cluster autoscaler, disaster recovery
18. [Performance, Capacity, Cost](18-performance-capacity-and-cost.md) — VPA, HPA internals, bin packing, multi-tenancy, cost optimization
19. [Production Operations and Incident Response](19-production-operations-and-incident-response.md) — GitOps, deployment strategies, CI/CD, on-call reality, war stories, chaos engineering

## How to use this

Read Part 1 in order if you're new to Kubernetes — each file builds on the last. Part 2 is not sequential — read whichever topic matches what you're working on, but read [13 Troubleshooting](13-troubleshooting-and-debugging.md) and [11 etcd/control plane internals](11-etcd-and-control-plane-internals.md) early since they inform how you read everything else.
