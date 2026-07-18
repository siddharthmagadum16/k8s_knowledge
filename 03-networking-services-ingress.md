# Networking, Services, and Ingress

## 1. The Kubernetes Networking Model

Kubernetes imposes a small set of fundamental requirements on any network implementation (the "Kubernetes networking model"), regardless of which CNI plugin provides it:

1. **Every Pod gets its own IP address** — no port-mapping/NAT needed to reach a Pod's containers from elsewhere in the cluster. Containers within a Pod share that one IP (they communicate via `localhost`).
2. **Pods can reach all other Pods' IPs directly, cluster-wide, without NAT** — a Pod on node A can talk to a Pod on node B using the Pod B's IP directly, as if they were on a flat LAN, regardless of physical node topology.
3. **Nodes can reach all Pods, and vice versa, without NAT** (for node-level agents/health checks).
4. The IP a Pod sees itself as (i.e., what it binds to) is the same IP others use to reach it — no additional translation layer inside the pod's own view.

This is a deliberate simplification versus Docker's default per-host NAT'd bridge network, where containers on different hosts can't reach each other by IP without explicit port publishing. Kubernetes pushes that complexity into the CNI plugin so application authors never think about NAT/port mapping.

### 1.1 How the flat network is actually achieved

CNI plugins implement this flat-network guarantee via one of a few strategies:
- **Overlay networking** (e.g., Flannel VXLAN, Calico IP-in-IP mode): Pod traffic between nodes is encapsulated in a tunnel protocol over the existing physical network. Simple, works over any underlying network, adds some encapsulation overhead.
- **Native L3 routing / BGP** (e.g., Calico BGP mode): each node advertises routes for its local Pod CIDR to the rest of the network (via BGP peering), so Pod-to-Pod packets are routed natively without encapsulation. Lower overhead, requires routable infrastructure.
- **Cloud VPC-native** (e.g., AWS VPC CNI, GCP VPC-native/alias IP): Pods get IPs directly from the cloud VPC's address space, so Pod IPs are natively routable within the VPC without any overlay — Pods look like first-class VPC citizens (has implications for IP address exhaustion planning at scale).

Each node is assigned a distinct **Pod CIDR block** (e.g., node1: `10.244.1.0/24`, node2: `10.244.2.0/24`) carved out of the cluster's overall `--pod-network-cidr`; the CNI plugin's job is making cross-node routing between these blocks work.

### 1.2 Why Pods are ephemeral and why Services exist

Pod IPs are **not stable** — a Pod that's deleted and recreated (by a Deployment during a rollout, or after a node failure) gets a brand-new IP. Hardcoding Pod IPs anywhere is a bug waiting to happen. The **Service** object exists to provide a stable virtual IP and DNS name in front of a changing set of Pod IPs.

## 2. Services

A Service is a stable abstraction — a virtual IP (VIP) plus a DNS name — that load-balances traffic across a dynamic set of backend Pods, selected by label.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web           # matches Pods with label app=web
  ports:
  - name: http
    port: 80            # Service's own port (what clients use)
    targetPort: 8080     # container port to forward to
  type: ClusterIP        # default
```

### 2.1 `ClusterIP` (default)

- Allocates a virtual IP reachable only **inside the cluster**, from any Pod/node.
- This is the building block every other Service type is built on top of.
- Use for internal service-to-service communication (the overwhelming majority of Services in a cluster: backend APIs, databases, caches).

### 2.2 `NodePort`

```yaml
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080     # optional; auto-assigned from 30000-32767 range if omitted
```

- Everything `ClusterIP` gives you, **plus**: the Service is additionally exposed on a static port (`nodePort`) on **every node's IP** in the cluster, whether or not a Pod for that Service is actually running on that particular node (traffic is forwarded internally to wherever the Pods are).
- Reaching `<any-node-ip>:30080` from outside the cluster hits the Service.
- Rarely used directly in production as the final entry point (raw node IPs are unwieldy, no TLS termination, no host/path routing) — mostly used as the underlying mechanism for `LoadBalancer` Services or in bare-metal setups without a cloud LB.

### 2.3 `LoadBalancer`

```yaml
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

- Superset of `NodePort`: additionally asks the **cloud-controller-manager** to provision an actual external load balancer (AWS NLB/ELB, GCP Network LB, Azure LB) that forwards to the NodePort on cluster nodes (or, with some CNI/cloud integrations, directly to Pod IPs).
- `status.loadBalancer.ingress` gets populated with the external IP/hostname once provisioning completes (`kubectl get svc` shows it under `EXTERNAL-IP`).
- On bare-metal/self-hosted clusters with no cloud integration, `EXTERNAL-IP` stays `<pending>` forever unless something like **MetalLB** is installed to emulate this (it assigns IPs from a configured pool and announces them via ARP/BGP).
- One `LoadBalancer` Service typically means one real cloud load balancer provisioned (and billed) per Service — this is why Ingress exists (see below), to avoid needing one LB per app.

### 2.4 `ExternalName`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  type: ExternalName
  externalName: db.on-prem.example.com
```

- No selector, no proxying, no virtual IP at all. Purely a **DNS CNAME** — anything in the cluster resolving `legacy-db.default.svc.cluster.local` gets back a CNAME to `db.on-prem.example.com`, which is then resolved normally.
- Use case: giving an external/legacy resource (an on-prem database, a third-party API endpoint) a cluster-internal-looking name, so application code doesn't need environment-specific hostnames, and you can swap the external target later without changing app config.

## 3. Endpoints / EndpointSlices

A Service's `selector` doesn't directly wire traffic — it's resolved into a list of backend IPs, tracked as a separate object:

- **Endpoints** (legacy, one object per Service, a single flat list of all backend IP:port pairs) — being phased out for large Services because a single Endpoints object grows unboundedly and every change requires rewriting/redistributing the whole object to every kube-proxy watcher.
- **EndpointSlices** (current default): the same information, but sharded into multiple smaller objects (default max 100 endpoints per slice), which scales far better for Services with thousands of backend Pods, and is extensible (carries additional per-endpoint fields like `topology`/zone hints).

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```

The **Endpoint/EndpointSlice controller** (part of kube-controller-manager) watches Pods matching a Service's selector and keeps the corresponding EndpointSlice(s) in sync — adding an entry when a Pod becomes `Ready`, removing it when the Pod is deleted or fails readiness.

Note: only Pods passing their **readiness probe** are included as endpoints — a Running-but-not-Ready Pod (e.g., still warming a cache) receives no traffic even though it's alive, avoiding sending requests to a Pod that isn't actually prepared to serve.

## 4. kube-proxy Modes

kube-proxy on every node watches Services + EndpointSlices and programs local packet-forwarding rules so traffic to a Service's ClusterIP reaches one of the backend Pod IPs. Two common modes:

### 4.1 iptables mode (long-time default)
- Programs Linux `iptables` NAT rules: a packet destined for the Service VIP gets its destination address rewritten (DNAT) to a randomly chosen backend Pod IP, weighted equally.
- Rule evaluation is a linear chain — with thousands of Services, iptables rule-matching becomes O(n)-ish per packet and rule-program updates get slow (every change touches large rule sets), a known scaling bottleneck at large cluster size.

### 4.2 IPVS mode
- Uses the Linux kernel's IP Virtual Server (a real, purpose-built L4 load balancer built into the kernel), backed by hash tables rather than sequential rule chains.
- Scales far better with large numbers of Services (near O(1) lookup) and supports more load-balancing algorithms (round robin, least connection, etc., vs iptables' random choice).
- Requires the `ip_vs` kernel modules present on nodes; increasingly the default recommendation for large clusters.

(A newer eBPF-based approach, used by Cilium's kube-proxy replacement, bypasses kube-proxy/iptables/IPVS entirely and programs eBPF hooks directly for even lower overhead — mentioned here as the direction the ecosystem is moving, not a kube-proxy "mode" per se.)

## 5. CoreDNS and Service Discovery

**CoreDNS** runs as a Deployment inside the cluster (in `kube-system`), and every Pod's `/etc/resolv.conf` is configured (by the kubelet, at Pod creation) to point at the CoreDNS ClusterIP as its nameserver, with a search path including the Pod's namespace.

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

DNS naming convention for a Service: `<service-name>.<namespace>.svc.<cluster-domain>` (cluster-domain defaults to `cluster.local`).

- `my-svc.my-namespace.svc.cluster.local` → resolves to the Service's ClusterIP (for a normal Service), or directly to backend Pod IPs in round-robin (for a **headless** Service, `clusterIP: None` — see the StatefulSet discussion in the workloads doc, where each Pod also gets its own record: `postgres-0.postgres-headless.default.svc.cluster.local`).
- Within the same namespace, the short name alone (`my-svc`) resolves due to the search path.
- Across namespaces, you need at least `my-svc.other-namespace`.
- Pods themselves also get DNS records if `subdomain`/`hostname` are set on the Pod spec combined with a headless Service, though this is less commonly used directly than the Service-level name.

`ndots:5` means any name with fewer than 5 dots is tried against each search domain before being tried as an absolute/external name — this is a well-known source of latency for external DNS lookups from inside pods (each external hostname lookup wastes several failed internal queries first) and is why heavily DNS-dependent apps sometimes tune `dnsConfig` or append a trailing dot to fully-qualified external names.

## 6. Ingress

### 6.1 The problem Ingress solves

A `LoadBalancer` Service gives you one external IP/LB per Service — fine for one or two Services, expensive and unwieldy for dozens of HTTP(S) apps that all want to live behind `https://example.com/...` on the same domain with different paths, or on different subdomains, all needing TLS termination. You don't want 30 cloud load balancers for 30 microservices.

**Ingress** is an L7 (HTTP/HTTPS) routing abstraction: a single entry point (usually one `LoadBalancer` Service backing an Ingress Controller) that then does host- and path-based routing to many backend Services inside the cluster, and can centralize TLS termination.

### 6.2 Ingress Controllers

An `Ingress` object by itself does nothing — it's just a declarative rule set. You need an **Ingress Controller** (a separately deployed piece of software, itself typically a Deployment + a `LoadBalancer`/`NodePort` Service) that watches `Ingress` objects and configures itself (a reverse proxy) accordingly. Kubernetes ships no default Ingress Controller — you install one.

Common choices:
- **ingress-nginx**: the most widely deployed, wraps NGINX, driven by templated config regenerated on Ingress changes.
- **Traefik**: config auto-discovery, native Let's Encrypt integration, popular in Docker/K8s hybrid shops.
- **HAProxy Ingress, Contour (Envoy-based), Kong, cloud-native ones** (AWS Load Balancer Controller which provisions an actual ALB per Ingress, GKE's native Ingress-to-HTTP(S)-LB integration).

### 6.3 IngressClass

Since a cluster can run multiple Ingress Controllers simultaneously, `IngressClass` tells Kubernetes (and disambiguates for the controllers themselves) which controller should handle which Ingress objects.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["shop.example.com"]
    secretName: shop-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```

### 6.4 Host-based and path-based routing

- **Host-based**: `shop.example.com` and `admin.example.com` route to entirely different backend Services, all through the same Ingress Controller/LB IP — DNS for both hostnames points at the same external IP; the controller inspects the HTTP `Host` header to decide where to send the request.
- **Path-based**: within one host, `/api` vs `/` route to different Services. `pathType: Prefix` matches by path segment prefix; `Exact` requires an exact match; `ImplementationSpecific` defers matching semantics to the controller (legacy, avoid where possible).

### 6.5 TLS termination

- `spec.tls` references a `Secret` of type `kubernetes.io/tls` (containing `tls.crt` + `tls.key`) — the Ingress Controller terminates HTTPS at the edge using this certificate, then typically forwards plain HTTP internally to the backend Service (re-encryption to the backend is also possible with annotations, if you need end-to-end TLS).
- In practice, **cert-manager** is layered on top to automate provisioning/renewal of these certs (e.g., via Let's Encrypt ACME), so you rarely hand-manage the Secret.

### 6.6 Gateway API (brief mention)

The newer **Gateway API** (`gateway.networking.k8s.io`) is the eventual successor to Ingress — more expressive (native support for TCP/UDP/gRPC routing, cross-namespace routing, richer traffic splitting for canaries), with clearer role separation (infra admins own `Gateway`, app teams own `HTTPRoute`). Ingress remains extremely widely deployed and is not going away soon, but new designs increasingly default to Gateway API where the controller supports it.

## 7. NetworkPolicy

By default, Kubernetes networking is **fully open**: any Pod can reach any other Pod (and be reached by any other Pod) cluster-wide, subject only to the flat-network model described above — there is no default isolation. `NetworkPolicy` objects let you restrict this, but **only take effect if the CNI plugin implements NetworkPolicy enforcement** (not all do — Flannel alone doesn't; Calico, Cilium, and most production-grade CNIs do).

A NetworkPolicy is **additive and default-deny-once-selected**: as soon as any NetworkPolicy selects a Pod for a given traffic direction (ingress/egress), that direction becomes deny-by-default for that Pod except for what's explicitly allowed — across all NetworkPolicies matching it (they're combined with OR semantics, not overridden).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend        # this policy applies to Pods labeled app=backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend    # only allow traffic from Pods labeled app=frontend
    ports:
    - protocol: TCP
      port: 8080
```

Effect: any Pod labeled `app=backend` in namespace `prod` now **only** accepts inbound TCP:8080 traffic from Pods labeled `app=frontend` (in the same namespace, unless a namespace selector is added) — all other inbound traffic to it is dropped. Its outbound (egress) traffic is unaffected, since `policyTypes` only lists `Ingress` here.

A common baseline pattern: a default-deny-all policy per namespace, then explicit allow rules per legitimate traffic path:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}          # selects ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```

Combine with explicit allow rules (like the one above, plus DNS egress to CoreDNS, which is easy to forget and breaks name resolution if omitted) to build a least-privilege network posture per namespace.
