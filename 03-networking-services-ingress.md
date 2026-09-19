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

**What "virtual IP" actually means:** a ClusterIP is not a real address bound to any actual machine or process — it's purely a routing target that exists in the cluster's networking rules (programmed by kube-proxy on every node), not something you'd find "listening" anywhere.

**Where it lives:** nowhere in particular — it's not "located" on any single node, it's a virtual address recognized cluster-wide. Every node's kube-proxy has the same programmed rules (iptables/IPVS) that say "packets addressed to this ClusterIP should be rewritten (DNAT'd) to one of these real backend pod IPs." So the ClusterIP itself has no physical home; it becomes real only at the moment some node's kube-proxy intercepts traffic addressed to it.

**Its role, concretely:** a caller (e.g. the Ingress Controller in the traced journey in §6.7) doesn't know or care which specific pod will actually handle the request — it just sends the packet to the Service's ClusterIP, a single stable address that doesn't change even as backend pods are created, deleted, or rescheduled with new IPs. That stability is the entire point: it decouples "who's calling" from "which exact pod answers." The actual pod selection happens one step later, when kube-proxy (on whichever node the packet currently passes through) intercepts that ClusterIP-addressed packet and rewrites its destination to one specific healthy backend pod IP, picked from the Service's EndpointSlice (§3).

So: ClusterIP = the stable, virtual "front door" address for a Service; kube-proxy = the mechanism that turns a packet addressed to that front door into a packet addressed to one real pod behind it.

**Clarification: does Ingress determine whether a Service is headless?** No — unrelated. Headless-ness is set purely by `clusterIP: None` on the Service itself; Ingress is an L7 router that sits in front of a Service and doesn't create or require headlessness either way. The actual relationship: Ingress normally targets a **normal ClusterIP Service** as its backend (one routable address in, load-balanced to pods behind it). A **headless** Service (§5, used for StatefulSets) is rarely put behind an Ingress, precisely because the two solve opposite problems — Ingress wants one address to route to, headless exists specifically to expose individual pods instead — but nothing technically prevents it.

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

### 5.1 Clarification: `/etc/resolv.conf` vs CoreDNS — who does what

It's easy to assume `/etc/resolv.conf` and CoreDNS are two separate resolution steps doing redundant work. They're not — they're two different *kinds* of thing entirely:

- **`/etc/resolv.conf` does no DNS resolution itself.** It's purely a config file read by the pod's resolver library (glibc's stub resolver, invoked by any app via `getaddrinfo`/Go's `net.Dial`/etc.). It has zero DNS logic — it just tells that library **where to send queries** (`nameserver 10.96.0.10`) and **what suffixes to try first** (the `search` list, governed by `ndots`).
- **CoreDNS is where actual resolution happens.** It's a real DNS server process that receives the query sent to that nameserver IP, checks it against its live view of Services/Pods (via its watch on the API server), and either answers directly (cluster-local name) or forwards the query upstream to the real internet DNS resolver and relays the answer back (external name).

So the flow per lookup is:
1. App code asks to resolve a hostname (e.g. `orders-db`).
2. The resolver library reads `/etc/resolv.conf` to find out which server to ask and what suffixes to append first, given `ndots`.
3. It sends an actual DNS query **over the network** to that nameserver IP — CoreDNS's ClusterIP.
4. CoreDNS does the real lookup and returns the answer (or forwards upstream first, for non-cluster names).

There's no double resolution — the pod-side step is just "read config, then send one query"; CoreDNS is the only thing actually resolving anything. Without CoreDNS running, `/etc/resolv.conf` would just be pointing at an IP with nothing listening — it wouldn't resolve anything on its own.

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

### 6.7 Traced example: a pod calls another service's *external* hostname

A instructive (and common-mistake) scenario: a pod behind `service-a-svc` (in the same cluster) makes a call to `https://service-b.com/api/orders` — i.e., it addresses the *public* Ingress hostname of another service in the same cluster, instead of that service's internal cluster-DNS name. Tracing the full round trip shows exactly why that's expensive, and where each hop actually happens:

**Request path:**

1. **App code makes the call** — a pod behind `service-a-svc` does `http.Get("https://service-b.com/api/orders")`.
   📍 *Located: inside the pod, application layer.*

2. **DNS resolution starts** — the pod's resolver reads `/etc/resolv.conf`. Since `service-b.com` has fewer than 5 dots, `ndots:5` makes it first try `service-b.com.default.svc.cluster.local`, then `.svc.cluster.local`, then `.cluster.local` — all fail (NXDOMAIN), wasting 3 queries — before finally trying it as an absolute external name.
   📍 *Located: pod's stub resolver + CoreDNS.*

3. **CoreDNS forwards upstream** — `service-b.com` isn't a cluster-local Service, so CoreDNS's `forward` plugin passes the query to the real upstream DNS resolver (cloud VPC resolver → public DNS).
   📍 *Located: CoreDNS (kube-system).*

4. **External DNS resolves it** — `service-b.com` resolves to the public IP of **service-b's Ingress Controller external LoadBalancer**.
   📍 *Located: outside the cluster — public DNS.*

5. **Request leaves the cluster** — the pod's packet is routed out through the node's network, NAT'd to a public IP, and travels over the internet to that LB IP — even though the destination is running in the *same cluster* as the caller.
   📍 *Located: CNI networking + node's egress path.*

6. **Cloud Load Balancer receives it** — forwards to a node's NodePort, or directly to an Ingress Controller pod IP.
   📍 *Located: cloud LB (AWS NLB/ALB, etc.) — outside K8s, managed by cloud-controller-manager.*

7. **Ingress Controller reads the `Host` header** — sees `service-b.com`, matches the corresponding host block in its Ingress rules (host-based routing, §6.4).
   📍 *Located: Ingress Controller pod, inside the cluster.*

8. **Path matching** — controller checks the URL path (`/api/orders`) against that host's path rules, matches the relevant prefix, picks the target Service (`service-b-api-svc`).
   📍 *Located: Ingress Controller.*

9. **TLS termination** — if HTTPS, the controller decrypts here using the cert from the referenced Secret; traffic onward is typically plain HTTP unless re-encryption is configured (§6.5).
   📍 *Located: Ingress Controller.*

10. **Forwarded to the backend Service, and a specific pod is picked** — the Ingress Controller sends the request to `service-b-api-svc`'s ClusterIP (one stable virtual IP). But no real machine is listening on that virtual IP — it's not a live process, just a routing target. So kube-proxy (running on every node) intercepts the packet and rewrites its destination (DNAT) to one **actual pod IP**, picked from the list of currently-healthy pods behind that Service (its EndpointSlice, §3) — usually at random (iptables mode) or via round-robin/least-connection (IPVS mode). It's not "the same pod every time" and not based on which pod is "closest" — just whichever healthy pod the load-balancing rule currently points to for this packet. (Note: this DNAT decision is made by exactly **one** kube-proxy — not broadcast to every node's kube-proxy — see the clarification just below the pointwise list.)
    📍 *Located: kube-proxy (every node) + the target pod.*

11. **Pod processes the request** — the actual backend pod handles the request and returns a response.
    📍 *Located: destination pod.*

**Response path** is the exact reverse: pod → kube-proxy DNAT unwound → Ingress Controller → cloud LB → internet → back into `service-a`'s pod's original connection.

**Why this matters:** steps 4 through 10 are a full round trip *out of the cluster and back in* through a public load balancer, purely because the pod addressed the external hostname instead of internal Service DNS. If `service-a` and `service-b` are both inside the same cluster, the correct call is directly to `service-b-api-svc.default.svc.cluster.local` (or just `service-b-api-svc` within the same namespace) — resolved straight to the internal ClusterIP by CoreDNS in one hop, skipping Ingress, the cloud LB, and the internet entirely. Calling a service's public hostname for internal service-to-service traffic is a common but real anti-pattern: it adds latency, cost (LB data-processing charges), and an unnecessary external dependency for traffic that never needed to leave the cluster.

**Clarification: does step 10 hit every node's kube-proxy, or just one?**

Just one — the DNAT decision in step 10 is never broadcast to every node's kube-proxy. Here's exactly what happens:

1. The Ingress Controller pod sends the packet to `service-b-api-svc`'s ClusterIP, from wherever that Ingress Controller pod itself is currently running — say Node A.
2. That packet has to leave the pod's network namespace, which means it passes through **Node A's own kube-proxy rules first** — the ones already programmed onto Node A's kernel (iptables/IPVS).
3. Node A's kube-proxy intercepts it right there and rewrites the destination (DNAT) to one specific backend pod IP, chosen from its local copy of the EndpointSlice list.
4. The packet is now addressed directly to that one pod's IP, and gets routed (via the CNI) straight to whatever node that pod lives on (could be Node A itself, or Node B, C, etc.) — as a single point-to-point packet, not a broadcast.

So it's not "sent to every kube-proxy" — it only ever passes through **the kube-proxy on the node where the packet currently is**. Every node's kube-proxy independently holds the *same* full list of backend pod IPs (they all watch the same EndpointSlices from the API server), so any node's kube-proxy is equally capable of making this DNAT decision on its own — but only the one node actually handling that packet at that moment does it.

Think of it like a signpost at every fork in a road network, all showing the same set of destinations — a car doesn't get copied to every signpost simultaneously; it just reads whichever signpost it's currently passing and picks one path. That's exactly why there's no duplication risk: the DNAT decision happens exactly once, at the first node the packet touches, and after that it's a normal routed packet headed to one specific pod IP — no other node's kube-proxy ever sees or acts on that same packet again.

**Clarification: where does `NodePort` fit into this journey?**

`NodePort` is a **Service type** (§2.2), not an independent hop in the traced journey above — it doesn't appear as its own numbered step; it's the mechanism *underneath* the `LoadBalancer` Service the Ingress Controller sits behind.

Here's exactly where it fits:

- The Ingress Controller (e.g. ingress-nginx) is itself deployed as a Deployment + a `type: LoadBalancer` Service.
- A `LoadBalancer` Service is a **superset of `NodePort`** (§2.3) — creating it also opens a static port (e.g. `30080`) on **every node's IP** in the cluster, whether or not an Ingress Controller pod happens to be running on that specific node.
- The cloud load balancer (step 6 above — "Cloud Load Balancer receives it") doesn't talk to pod IPs directly in the classic path; it forwards incoming traffic to `<any-node-ip>:30080` — that NodePort — on one of the cluster's nodes.
- From there, kube-proxy's rules on that node take over: they see traffic hit the NodePort and forward it (via DNAT) to one of the actual Ingress Controller pod IPs, wherever it's running, even on a different node if needed.

So in this scenario: **NodePort is the handoff point between "outside the cluster" and "inside the cluster."** The cloud LB doesn't know or care about pod IPs — it just needs *a node's IP + a fixed port* to send traffic to. NodePort is what makes every node uniformly capable of receiving that traffic and routing it onward to wherever the Ingress Controller pod actually lives.

One nuance worth flagging: on many cloud setups (e.g. the AWS Load Balancer Controller for ALBs, or an NLB in IP-target mode), the LB is configured to target pod IPs directly instead of going through NodePort — so whether NodePort is actually used as a real hop depends on the specific cloud/controller integration. But the "plain" LoadBalancer-Service-over-NodePort path described above is the general-case mechanism, and that's the one where NodePort genuinely sits in the path as the node-level entry point.

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
