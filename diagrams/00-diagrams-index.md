# Kubernetes Architecture Diagrams

Seven Mermaid diagrams that map the whole Kubernetes system, told as one worked example: **“ShopFast”**, an e-commerce company running its app (frontend + api + redis + postgres) on **AWS EKS**. Every component carries a **hover tooltip** explaining what it is and does, and the dynamic diagrams are **numbered** so you can follow the flow step by step.

## Fastest way to view it (recommended)

Open **[`../k8s-architecture-map.html`](../k8s-architecture-map.html)** in any browser (double-click it). It renders all seven diagrams on one page with:

- guaranteed **hover tooltips** on every box,
- numbered-flow narration above each diagram,
- a light/dark theme toggle and a table of contents.

Needs an internet connection the first time (it loads the Mermaid library from a CDN).

## The seven diagrams

| # | File | What it shows | Read alongside |
|---|------|---------------|----------------|
| 1 | [`01-master-architecture.mmd`](01-master-architecture.mmd) | **The big picture** — every subsystem at once: users, CI/CD supply chain, cloud, control plane, add-ons, worker nodes. Static structure + key relationships. | [01](../01-introduction-and-architecture.md), [11](../11-etcd-and-control-plane-internals.md) |
| 2 | [`02-kubectl-apply-flow.mmd`](02-kubectl-apply-flow.mmd) | **`kubectl apply` control flow** — auth → admission → etcd → controllers → scheduler → kubelet, the desired-state loop. | [01](../01-introduction-and-architecture.md), [11](../11-etcd-and-control-plane-internals.md) |
| 3 | [`03-pod-scheduling-startup.mmd`](03-pod-scheduling-startup.mmd) | **Scheduling + container startup** — filter/score/bind, then sandbox → CNI → init → app → probes. Shows exactly where **Docker vs containerd** fit. | [05](../05-scheduling-and-resource-management.md), [12](../12-advanced-scheduling.md), [06](../06-health-checks-and-lifecycle.md) |
| 4 | [`04-networking-traffic-flow.mmd`](04-networking-traffic-flow.mmd) | **Request traffic** — north-south (user → LB → Ingress → Service → kube-proxy → pod) and east-west (pod → CoreDNS → Service → pod). | [03](../03-networking-services-ingress.md), [10](../10-networking-deep-dive.md) |
| 5 | [`05-storage-provisioning.mmd`](05-storage-provisioning.mmd) | **Dynamic storage** — PVC → StorageClass → CSI → cloud volume → PV bound → mounted. | [04](../04-configuration-storage.md) |
| 6 | [`06-cicd-gitops-flow.mmd`](06-cicd-gitops-flow.mmd) | **CI/CD + GitOps** — build/scan/sign/push, config-repo bump, Argo CD sync, admission, rolling update, rollback. | [08](../08-helm-and-package-management.md), [19](../19-production-operations-and-incident-response.md), [14](../14-security-hardening.md) |
| 7 | [`07-scaling-observability.mmd`](07-scaling-observability.mmd) | **Autoscaling + observability** — HPA pod scaling, Cluster Autoscaler node scaling, metrics/logs/alerts pipeline. | [15](../15-observability.md), [18](../18-performance-capacity-and-cost.md) |

## Importing the `.mmd` files elsewhere

Each `.mmd` file is standalone Mermaid source. To view or edit one:

- **[mermaid.live](https://mermaid.live)** — paste the file contents. Note: to get the hover tooltips there, open **Config** and set `securityLevel` to `loose` (the default `strict` disables the `click`/tooltip directives). The graph itself renders either way.
- **VS Code** — install the “Markdown Preview Mermaid Support” or “Mermaid Editor” extension and open the `.mmd`.
- **GitHub** — renders Mermaid inside ```` ```mermaid ```` fenced blocks in Markdown, but **disables `click`/tooltips** for security. The diagram shows; tooltips won’t.
- **Eraser / Excalidraw / Draw.io** — import via their Mermaid-import option (tooltip support varies by tool).

Because tooltip support is inconsistent across platforms, the bundled **`k8s-architecture-map.html`** is the canonical, fully-featured view.

## Legend (used across all diagrams)

- `───▶` solid arrow = live data / traffic path
- `· · ▶` dotted arrow = watch / control (reconcile) relationship
- `①②③ / ⒶⒷ / ⓍⓎ` = ordered steps within a flow
- `( )` cylinder shape = a datastore (etcd, registry, cloud disk)
