# Kubernetes Networking Deep Dive

## 1. CNI Internals

### The invocation path

The CNI spec is deliberately dumb: it's an exec-based plugin interface, not a daemon protocol (though most implementations also run a daemon alongside the exec'd binary for state). The chain of custody for a pod's network setup:

```
kubelet -> CRI (containerd/CRI-O) -> creates pod sandbox (pause container, netns)
        -> CRI shells out to CNI binary via containerd's CNI plugin manager
        -> CNI binary reads config from /etc/cni/net.d/*.conf(list)
        -> CNI binary performs ADD, returns IP/routes as JSON on stdout
```

Concretely, when a Pod is scheduled, kubelet calls `RunPodSandbox` via the CRI gRPC interface. containerd creates the network namespace and the pause container that holds it open, then invokes the configured CNI plugin binary (e.g. `/opt/cni/bin/calico`) with:

- `CNI_COMMAND=ADD`
- `CNI_CONTAINERID=<sandbox id>`
- `CNI_NETNS=/var/run/netns/cni-<uuid>`
- `CNI_IFNAME=eth0`
- `CNI_PATH=/opt/cni/bin`
- stdin = the JSON config from `/etc/cni/net.d/`

The plugin does its work (create veth, assign IP, program routes/BPF/iptables) and prints CNI result JSON to stdout, which containerd hands back to kubelet, which uses the returned IP to populate `pod.status.podIP`.

The four commands every plugin must implement:

- **ADD** — create the pod's network interface, assign IP, wire routing. Called on sandbox creation, and also idempotently on kubelet reconciliation restarts (must be safe to call again against an existing setup — many plugin bugs live here).
- **DEL** — tear down. Called on pod deletion. Must be idempotent and must not fail hard if resources are already gone (e.g. node crashed and never got to call DEL, then a new command comes in against orphaned state).
- **CHECK** — introspect and verify the previously created network is still intact (kubelet calls this periodically as a health check for the pod network). Not all plugins implement it well; a common bug source is `CHECK` throwing errors that get treated as invalidating the whole pod.
- **VERSION** — report supported CNI spec versions.

### Config files: `/etc/cni/net.d/`

kubelet's `--cni-conf-dir` defaults to `/etc/cni/net.d`. The **lexically first file** (by filename) is the one used — this is a frequent footgun when two CNI installers drop configs into the same directory (e.g. Multus alongside Calico) and ordering breaks because of alphabetical sort, not install order.

Example single-plugin config (`10-calico.conflist`):

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1440,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
```

This is a **plugin chain** — the `plugins` array is executed in order, each one receiving the previous plugin's result as input (the "prevResult" field in the CNI spec). `calico` sets up the interface and routing; `portmap` adds hostPort iptables DNAT rules; `bandwidth` attaches tc qdiscs for rate limiting. Chaining is how CNI achieves modularity without every plugin reimplementing IPAM or interface creation.

### IPAM plugins

IPAM is itself pluggable and invoked as a sub-step inside ADD:

- **host-local** — allocates from a locally-configured CIDR range per node, tracks allocations in a file under `/var/lib/cni/networks/<network-name>/`. Simple, no coordination needed beyond the node getting a unique podCIDR from the controller-manager (`--allocate-node-cidrs`). Used by Flannel and (optionally) Calico in non-BGP setups.
- **calico-ipam** — talks to the Calico datastore (etcd or Kubernetes CRDs) to allocate from IP pools, supports per-namespace pools, borrows blocks of addresses per-node (a `/26` block, say) to avoid a datastore round trip on every pod creation. This block-borrowing is why you'll see Calico nodes hold onto more addresses than pods running on them — check with `calicoctl ipam show --show-blocks`.
- **whereabouts** — a cluster-wide IPAM plugin (from the Multus ecosystem) for cases where multiple nodes need to allocate from the *same* shared range without per-node carve-outs, backed by CRDs with lease-like locking to avoid double allocation. Common with Multus multi-homed pods on the same L2 segment/VLAN where per-node CIDR blocks don't make sense.

### veth pair mechanics

Every pod's primary interface is one end of a veth pair:

```
[ Pod netns ]                          [ Host root netns ]
  eth0@if7  <---- veth pair ---->  cali1234abcd@if2 (or vethXXXX)
  10.244.1.15/32                    (no IP, L2 only)
      |                                    |
      +------ single virtual wire ---------+
```

The CNI plugin creates the pair with `ip link add`, moves one end into the pod's netns (renamed to `eth0` there), and leaves the other end in the host namespace. What happens to that host-side end depends on the plugin's data plane mode:

- **Bridge mode** (classic bridge, old Flannel/CNI bridge plugin): host-side veth end is enslaved to a Linux bridge (`cni0`, `cbr0`, `docker0`-style). The bridge does L2 forwarding between all pods on the node. Traffic leaving the node goes through whatever routing/NAT the overlay backend does.
- **Routed mode** (Calico default, no bridge): host-side veth end gets **no bridge**. Instead a `/32` route is installed in the host's routing table pointing at that veth for the pod's IP, and the pod's default route inside its netns points at a link-local/point-to-point address on the host end. This is why `ip route` on a Calico node shows dozens of individual `/32` routes, one per local pod — it's pure L3 routing, not bridging. Confirm with:

```bash
ip route show | grep -E '/32'
# 10.244.1.15 dev cali1234abcd scope link
```

### Full packet path out of a pod (ASCII)

```
+-------------------------------------------------------------+
|                        Node (root netns)                     |
|                                                                |
|   eth0 (physical/cloud NIC)                                   |
|     ^                                                          |
|     | (routed / encapsulated / NAT'd depending on CNI mode)    |
|     |                                                          |
|   ip route lookup, iptables/nftables/eBPF hooks (FORWARD,      |
|   POSTROUTING, tc ingress/egress)                              |
|     ^                                                          |
|     |                                                          |
|  cali1234abcd@if2  <---veth pair--->  eth0@if7 (in pod netns)  |
|  (host end, no IP,                     10.244.1.15/32          |
|   /32 route or bridge port)            default via <link-local>|
|                                              ^                  |
|                                              |                  |
|                                       [ Pod's network namespace]|
|                                       [ /var/run/netns/cni-xxx ]|
+-------------------------------------------------------------+
```

For overlay modes, insert an encap/decap step between the veth and `eth0`:

```
pod eth0 -> veth -> host routing -> vxlan0/tunl0 (encapsulate) -> eth0 -> wire
```

### Calico: BGP mode vs overlay mode

**BGP mode (native, unencapsulated)** — each node runs Felix + BIRD (or now `bird` embedded via confd, in newer versions Calico uses its own lightweight BGP daemon). Every node peers with every other node (full mesh by default) or, at scale, peers with **route reflectors** to avoid O(n²) peering sessions. Each node advertises the `/26` (or configured block size) it owns for its local pods as a BGP route. Other nodes install that as a normal kernel route via their BGP-speaking process, pointing at the peer node's IP as next-hop. No encapsulation — packets are routed exactly like normal L3 traffic between hosts, just with pod IPs as endpoints. This is why BGP mode is the highest-throughput, lowest-CPU option — there is nothing extra happening to the packet, no encap/decap tax on every host.

Why it fails on cloud providers: AWS, GCP, Azure VPCs are configured to only accept traffic from the instance's own assigned IP (source/dest check) and don't run a BGP fabric you can peer into at the VPC network layer (no L2 adjacency guarantee between nodes either). Unless the cloud explicitly supports it (GCP allows disabling source/dest check per-instance and Calico can work in BGP mode there if you also handle route propagation; AWS requires disabling source/dest check AND doesn't propagate pod CIDR routes without a custom route table hack), plain BGP mode either silently blackholes traffic (packets sent to a pod IP the fabric doesn't know how to route) or gets dropped at the ENI level.

Peering topology check:

```bash
calicoctl node status
# Calico process is running.
# IPv4 BGP status
# +--------------+-------------------+-------+----------+-------------+
# | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
# +--------------+-------------------+-------+----------+-------------+
# | 10.0.1.5     | node-to-node mesh | up    | 03:14:22 | Established |
# | 10.0.1.6     | node-to-node mesh | up    | 03:14:25 | Established |
# +--------------+-------------------+-------+----------+-------------+
```

At scale (>50-100 nodes) full mesh BGP becomes expensive (every node holding a session and full routing table copy per peer). Route reflectors solve this: designate a small number of nodes (or dedicated RR instances) that every other node peers with, and disable full mesh (`nodeToNodeMeshEnabled: false` in the BGPConfiguration). RRs re-advertise routes between clients without requiring every client to peer with every other client — same pattern as internal BGP route reflection in traditional networking.

**Overlay mode (VXLAN/IPIP)** — used when the underlying fabric doesn't route pod CIDRs (cloud VPC default, or nodes without L3 adjacency). Every pod packet gets wrapped:

- **IPIP**: adds a 20-byte outer IP header. Simple, but IPIP doesn't carry a UDP/L4 header, which breaks some cloud load balancer / ECMP hashing that expects a 5-tuple, and some cloud security groups block protocol-4 (IPIP) entirely.
- **VXLAN**: adds outer IP + UDP + VXLAN header, roughly **50 bytes overhead**. More overhead than IPIP but survives cloud networks better because it's just UDP traffic on port 4789, which ECMP/hashing and security groups treat normally.

Overhead cost: every packet does an extra encap on egress and decap on ingress, at the kernel networking stack level for standard Calico (extra CPU per packet), pushing effective throughput down and adding a few microseconds latency per hop — usually not perceptible until you're pushing multi-Gbps sustained flows, where it shows up as elevated softirq CPU usage on the node (`mpstat -P ALL 1`, look at `%soft`).

Check which mode is active:

```bash
kubectl get ippool default-ipv4-ippool -o yaml | grep -E 'ipipMode|vxlanMode'
# ipipMode: Always
# vxlanMode: Never
```

### Cilium: eBPF-based dataplane

Cilium's core pitch is replacing the iptables/bridge-based dataplane with eBPF programs attached at `tc` (traffic control, ingress/egress qdisc hooks) and optionally `XDP` (earliest possible hook, before the kernel even builds an `sk_buff` in some driver modes — used for DDoS/L3-L4 filtering at line rate).

Instead of a packet traversing a chain of iptables rules or being routed conventionally, Cilium's eBPF program attached to the veth's `tc` hook does an in-kernel map lookup (identity, policy, service backend) and either forwards, drops, or redirects the packet — often via `bpf_redirect()`, skipping large parts of the normal kernel network stack traversal entirely (this is the basis of Cilium's "eBPF host-routing" which bypasses iptables and even some of the normal netfilter/routing-table lookups between veth and physical NIC).

Why it's faster:

- No sequential rule-list traversal (iptables' fundamental scaling problem — see kube-proxy section below); eBPF map lookups are hash-based, O(1)-ish regardless of the number of services/endpoints/policies.
- Fewer context switches / skb traversals — with native routing + eBPF host-routing, a packet can go from pod veth to physical NIC without passing through the normal per-hop netfilter hooks (PREROUTING/FORWARD/POSTROUTING) since the BPF program at `tc egress` handles the forwarding decision directly.
- Policy enforcement is a map lookup keyed by security identity (not IP — Cilium assigns each pod a numeric identity based on its labels, so policy computation happens once, not per-IP), which decouples policy cost from cluster size in a way IP-based iptables rules can't.

Inspect Cilium's BPF programs on a node:

```bash
cilium-dbg bpf endpoint list
cilium-dbg service list          # in-kernel service backend map, replaces kube-proxy's iptables/ipvs
tc filter show dev cali1234abcd ingress   # see the actual bpf program attached
bpftool prog show                # list all loaded eBPF programs cluster-node-wide
```

Cilium can run in **overlay mode** (VXLAN/Geneve, same MTU tax as Calico's overlay) or **native routing mode** (direct routing, requires the underlying network to route pod CIDRs — same constraint as Calico BGP mode, though Cilium can pair native routing with an external BGP speaker like MetalLB/FRR or its own `bgp-control-plane` to advertise routes without needing full Calico-style BGP mesh).

### Flannel (brief)

Flannel is intentionally minimal — no NetworkPolicy support, no BGP, just "give every node a subnet, get packets between nodes." Two common backends:

- **vxlan backend**: same encap overhead story as Calico VXLAN (~50 bytes), works anywhere including cloud, simplest possible overlay config (a single `flanneld` daemon per node writing `/run/flannel/subnet.env` and programming a `flannel.1` vxlan interface).
- **host-gw backend**: no encapsulation at all — programs the host's routing table directly with node-subnet-to-node-IP routes (`ip route add 10.244.2.0/24 via <node2-ip>`), functionally similar to Calico BGP's *result* but achieved by static route population rather than a dynamic routing protocol. Requires L2 adjacency between nodes (same broadcast domain / no router hop between them that would drop packets with the wrong next-hop), which rules it out on most cloud VPCs spanning multiple subnets without custom route table entries.

Flannel's appeal is operational simplicity for small/simple clusters; its cost is no first-class NetworkPolicy enforcement (see section 6) and no advanced traffic engineering.

---

## 2. kube-proxy Deep Dive

### iptables mode: chain structure

kube-proxy in iptables mode programs a layered chain structure in the `nat` table (and some rules in `filter`/`mangle` for things like `NodePort` and health checks). The layering exists so that adding/removing one service/endpoint only touches that service's own small chain, not a single monolithic rule list — but the *traversal* is still sequential per-chain.

Simplified real structure:

```
PREROUTING / OUTPUT
   -> KUBE-SERVICES                       (top-level dispatch, one rule per Service)
        -> KUBE-SVC-XPGD46QRK7WJZT7O      (per-Service chain, matches by dest VIP:port)
             -> KUBE-SEP-SXIVWICOYRO5RUAA (per-Endpoint chain #1, ~33% probability)
             -> KUBE-SEP-CNW6ZGMSHDJH3EYT (per-Endpoint chain #2, ~50% of remainder)
             -> KUBE-SEP-XXX              (per-Endpoint chain #3, remainder = 100%)
                  -> DNAT to pod IP:port
```

Real `iptables -t nat -L` style output:

```
Chain KUBE-SERVICES (2 references)
target                     prot opt source          destination
KUBE-SVC-XPGD46QRK7WJZT7O  tcp  --  0.0.0.0/0        10.96.0.10   /* kube-dns cluster IP */ tcp dpt:53
KUBE-SVC-4SW47YFZTEDKD3PJ  tcp  --  0.0.0.0/0        10.96.100.5  /* default/my-app cluster IP */ tcp dpt:80
... (one block per Service, in whatever order iptables restore last wrote them) ...

Chain KUBE-SVC-4SW47YFZTEDKD3PJ (1 references)
target                     prot opt source          destination
KUBE-SEP-SXIVWICOYRO5RUAA  all  --  0.0.0.0/0        0.0.0.0/0    statistic mode random probability 0.33332999982
KUBE-SEP-CNW6ZGMSHDJH3EYT  all  --  0.0.0.0/0        0.0.0.0/0    statistic mode random probability 0.50000000000
KUBE-SEP-XXX               all  --  0.0.0.0/0        0.0.0.0/0

Chain KUBE-SEP-SXIVWICOYRO5RUAA (1 references)
target     prot opt source          destination
KUBE-MARK-MASQ  all  --  10.244.1.7   0.0.0.0/0    /* mark hairpin traffic for masquerade */
DNAT       tcp  --  0.0.0.0/0        0.0.0.0/0    tcp to:10.244.1.7:8080
```

The `statistic mode random probability` trick is how iptables (which has no native concept of weighted load balancing) approximates uniform random distribution across N endpoints: rule 1 fires with probability `1/N`; if it doesn't match, rule 2 fires with probability `1/(N-1)` of the *remaining* traffic, and so on, until the last rule catches everything left. This is mathematically equivalent to uniform random selection across N backends, but it means for endpoint N in a list of N, the packet has already been tested against N-1 preceding rules before falling through.

### Why this doesn't scale

Every rule evaluation is a linear, sequential match test — there is no indexed/hashed lookup in iptables when traversing a chain (nftables improves this somewhat with concatenated set matching, but classic iptables mode in kube-proxy does not use that structure). Consequences at scale:

- With **N services**, `KUBE-SERVICES` alone has N sequential match rules — a packet destined for the *last* service in that chain has already been tested against N-1 preceding rules.
- Each service's endpoint fan-out adds more sequential rules.
- Clusters with a few thousand Services and tens of thousands of Endpoints commonly end up with **hundreds of thousands of iptables rules** cluster-wide, and single-digit-thousands of rules just in the per-node dataplane.
- Every packet that hits a Service VIP (not just new connections — conntrack helps for established flows, but every *new* connection setup pays this cost, and DNS-heavy workloads generate a lot of new UDP "connections") pays for this linear scan.
- Beyond the per-packet cost, **rule programming** cost matters operationally: `iptables-restore` (which is how kube-proxy atomically swaps in a new ruleset) becomes slow with huge rulesets — some clusters have measured `iptables-restore` taking 5-20+ seconds during a full resync, meaning **service/endpoint changes cluster-wide are delayed** during that window. This is the single biggest reason large clusters (GKE/EKS docs both call this out explicitly around ~5,000 Services) move off iptables mode.

Diagnose rule count and evaluate the blast radius:

```bash
iptables -t nat -L KUBE-SERVICES -n | wc -l
iptables -t nat -S | wc -l                     # total programmed rules on this node
conntrack -L -p udp --dport 53 | wc -l          # DNS-related conntrack pressure, see section 3
```

### IPVS mode

IPVS (IP Virtual Server, a Linux kernel L4 load balancer, the same tech backing traditional LVS deployments) replaces the per-service chain-of-rules model with **hash table lookups**. Each Service VIP becomes an IPVS virtual service; each endpoint becomes a real server entry under it — lookup to find which VIP a packet matches, and which backend to pick, is effectively O(1) regardless of how many services/endpoints exist. iptables is still used in IPVS mode, but only for a small, fixed set of things (masquerading rules for certain traffic patterns), not one rule per service.

Enable it: `kube-proxy --proxy-mode=ipvs` (and load `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, `ip_vs_sh` kernel modules on the node).

Scheduling algorithms available (`--ipvs-scheduler`):

- `rr` — round robin (kube-proxy's default in IPVS mode)
- `lc` — least connection: send to the backend with fewest active connections, useful when request durations are uneven
- `dh` — destination hashing: consistent hashing on destination IP, useful for cache-affinity type setups
- `sh` — source hashing: consistent hashing on source IP, gives you client-affinity without needing `sessionAffinity: ClientIP` semantics implemented elsewhere

Inspect with `ipvsadm` (needs `ipvsadm` installed on the node):

```bash
ipvsadm -Ln
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
# TCP  10.96.100.5:80 rr
#   -> 10.244.1.7:8080              Masq    1      2          14
#   -> 10.244.2.9:8080              Masq    1      1          9
#   -> 10.244.3.4:8080              Masq    1      0          11
```

`ipvsadm -Ln --stats` shows per-VS packet/byte counters, useful for confirming actual traffic distribution matches expectation (catches cases where an endpoint is silently receiving zero traffic due to a readiness probe issue upstream even though it's listed).

IPVS also needs a dummy interface (`kube-ipvs0`) to bind all Service VIPs to locally so the kernel's IPVS hooks see the traffic as locally destined:

```bash
ip addr show kube-ipvs0
# 10.96.0.1/32, 10.96.0.10/32, 10.96.100.5/32, ... one per Service ClusterIP
```

### eBPF: Cilium's full kube-proxy replacement

Cilium can disable kube-proxy entirely (`kubeProxyReplacement: true`/`strict`) and implement Service load balancing via eBPF at two possible layers:

- **Socket-level (sockops/`connect()`/`sendmsg` hooks)** — for pod-to-ClusterIP traffic, Cilium can intercept the `connect()` syscall itself and rewrite the destination to a chosen backend *before a packet is ever built*. This means for many east-west flows there is no DNAT step in the packet path at all — the application socket connects directly to the backend pod IP from the kernel's perspective. This is the fastest path since it avoids NAT/conntrack entry creation entirely for those flows.
- **tc-level (`bpf_redirect`, similar hook point as CNI use above)** — for traffic that doesn't go through the socket hooks (e.g. traffic arriving from outside the node, NodePort/LoadBalancer traffic, or hostNetwork pods), a `tc` ingress/egress eBPF program does the VIP-to-backend lookup via an eBPF hash map (`cilium_lb4_services`, `cilium_lb4_backends` — inspectable via `bpftool map dump`) and redirects/rewrites the packet, again without walking any iptables chain.

Net effect: no `KUBE-SERVICES`/`KUBE-SVC-*`/`KUBE-SEP-*` chains exist at all on a node running Cilium with kube-proxy replacement — `iptables -t nat -L` on such a node is nearly empty. Verify:

```bash
cilium-dbg status --verbose | grep -A5 "KubeProxyReplacement"
cilium-dbg service list
iptables -t nat -L KUBE-SERVICES 2>&1   # should error "chain doesn't exist" if replacement is active
```

### kube-proxy iptables traversal path (ASCII)

```
Packet to Service VIP 10.96.100.5:80
        |
        v
  PREROUTING (nat table)
        |
        v
  KUBE-SERVICES  ---- sequential match against every Service rule ----
        |  (match: dest == 10.96.100.5:80)
        v
  KUBE-SVC-4SW47YFZTEDKD3PJ
        |
        +--> [33% prob] KUBE-SEP-A --> DNAT to pod A:8080
        +--> [50% prob of remainder] KUBE-SEP-B --> DNAT to pod B:8080
        +--> [remainder] KUBE-SEP-C --> DNAT to pod C:8080
        |
        v
  POSTROUTING (KUBE-MARK-MASQ / MASQUERADE if needed for hairpin/SNAT)
        |
        v
  routed to pod via CNI's normal path (veth/bridge/overlay)
```

---

## 3. DNS Resolution Chain

### What's in the pod's resolv.conf

```bash
kubectl exec -it mypod -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

`ndots:5` means: if a queried name has **fewer than 5 dots**, the resolver will try appending each `search` domain first, in order, before trying the name as-is (absolute). Almost every hostname a pod resolves — `myservice`, `myservice.default`, `api.stripe.com` (2 dots, still under 5) — triggers this behavior.

### The real cost: query amplification

For `api.stripe.com` (an external, fully qualified name with 2 dots), glibc's resolver with `ndots:5` will try, **in order**:

1. `api.stripe.com.default.svc.cluster.local` — NXDOMAIN
2. `api.stripe.com.svc.cluster.local` — NXDOMAIN
3. `api.stripe.com.cluster.local` — NXDOMAIN
4. `api.stripe.com` (search list exhausted, try bare/absolute) — succeeds

That's **4 queries** (sometimes cited as 5 when both A and AAAA are attempted per search-domain step, which glibc does in parallel/sequentially depending on `options` and whether `single-request-reopen` is set) before the actual answer comes back. Every one of those failed lookups still costs a round trip to CoreDNS, and CoreDNS itself may have to check its `kubernetes` plugin zone data, fail, then fall through to its `forward` plugin to upstream resolvers for the final one — all of this multiplies both **latency** (each hop is a real network round trip, tens of ms adds up when serialized) and **load** on CoreDNS (an app doing lots of first-time external DNS lookups can 4-5x actual CoreDNS QPS versus what naive query-count expectations suggest).

Verify by tcpdumping the CoreDNS pod or the node's DNS traffic:

```bash
kubectl exec -it mypod -- sh -c 'nslookup api.stripe.com' 
# or better, capture actual query sequence:
kubectl debug node/<node> -it --image=nicolaka/netshoot -- \
  tcpdump -i any -n port 53
```

You'll see the `.default.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local` suffixed queries fire in sequence before the bare query succeeds.

### Fixes

1. **Fully qualify with a trailing dot**: `api.stripe.com.` (note trailing `.`) tells the resolver this is already absolute — skip the search list entirely, go straight to the single query. Cheapest fix but requires app/config changes everywhere a hostname is used.
2. **Set `ndots` per-pod via `dnsConfig`**:

```yaml
apiVersion: v1
kind: Pod
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
  dnsPolicy: ClusterFirst
```

Lowering `ndots` to something like `2` means fewer internal-name lookups accidentally skip the search path (weigh this against how many dots your actual in-cluster Service DNS names have — `myservice.default.svc.cluster.local` has plenty, but bare `myservice` calls still need the search list, so don't set it to `1` blindly).

3. **node-local-dns cache** — this is the standard production fix and doesn't require any app changes.

### CoreDNS caching and node-local-dns

CoreDNS's `cache` plugin caches responses in-process (default success TTL cap ~30s unless configured otherwise, negative/NXDOMAIN caching too), which helps for repeat queries hitting the *same* CoreDNS pod, but every pod's DNS query still traverses the network to a CoreDNS Service ClusterIP first — which means it goes through the same iptables/IPVS Service DNAT path (and conntrack) as any other Service traffic (see next section for the specific bug this causes with UDP).

**node-local-dns** runs a DNS caching daemon (`dnsmasq`/CoreDNS in cache-only role, actually upstream node-local-dns is itself a small CoreDNS binary) as a DaemonSet, listening on a link-local IP (`169.254.20.10` conventionally) bound via a dummy interface on **every node**. Pods are configured (via kubelet's `--cluster-dns` or by rewriting resolv.conf at pod creation, depending on install method) to query that local IP instead of the CoreDNS Service ClusterIP directly.

Why this specifically helps:

- **Avoids the Service VIP DNAT/conntrack path for cache hits** — traffic to `169.254.20.10` never goes through kube-proxy's Service NAT at all (it's a local process on the node, not a Service VIP), which eliminates the conntrack race described below for cached queries.
- **Reduces actual CoreDNS pod load** — most queries get served from the node-local cache, so CoreDNS pods see far fewer QPS, meaning fewer CoreDNS replicas needed and less risk of CoreDNS itself becoming a bottleneck/OOM target during query storms (e.g. a big deployment rollout of a service that does lots of first-connection external lookups).
- Still forwards cache misses to the real CoreDNS Service (or directly upstream, configurable), so correctness is unaffected.

Check it's running and pods are pointed at it:

```bash
kubectl get ds -n kube-system node-local-dns
kubectl exec -it mypod -- cat /etc/resolv.conf   # nameserver should read 169.254.20.10
```

### The classic UDP DNS conntrack race (5-second timeout bug)

This is one of the most infamous Kubernetes networking bugs and it's worth understanding at the kernel level.

The problem: two threads/processes in the same pod issuing **concurrent UDP DNS queries** (very common — glibc resolver often fires A and AAAA lookups in parallel) can race on **conntrack entry creation**. When both queries go out near-simultaneously from the same source port (UDP, low-numbered ephemeral port reuse, or specifically when using the same 5-tuple due to how some resolvers manage sockets) through the same DNAT'd Service VIP, there is a kernel-level race in `nf_conntrack` where **two packets try to insert a conntrack entry for what looks like the same tracked connection at the same time**. One of them loses the race, the DNAT rewrite for that packet is either skipped or applied inconsistently, and the response never routes back to the querying process correctly. The querying application then hits the standard UDP DNS resolver timeout, which historically defaults to **5 seconds** before falling back/retrying, producing the well-known "random 5 second delays on DNS lookups" symptom.

This was tracked upstream as a kernel/netfilter bug affecting any DNAT'd UDP service reached from the same source concurrently, not just Kubernetes-specific, but it manifests constantly in k8s because every pod's default DNS setup guarantees exactly this pattern (two near-simultaneous UDP queries through a DNAT'd VIP).

Diagnose:

```bash
# Look for repeated/duplicate insert attempts or drops:
conntrack -S | grep -i insert_failed
# insert_failed non-zero and climbing correlates directly with this symptom

# Confirm timeout pattern from the app side — look for exactly ~5s (or 2.5s x2 retries) stalls:
kubectl exec -it mypod -- sh -c 'time nslookup api.stripe.com'
```

Fixes/mitigations, in order of how commonly they're applied:

1. **node-local-dns** — sidesteps the issue for most traffic since queries to the node-local cache don't traverse a DNAT'd Service VIP at all.
2. Force single-request DNS resolution instead of parallel A/AAAA: `options single-request-reopen` or `single-request` in resolv.conf/dnsConfig — avoids the concurrent-query pattern that triggers the race.
3. Disable IPv6 lookups if you don't need AAAA at all (removes half the parallel query pattern).
4. Some environments patch/tune conntrack table sizing and hash table (`nf_conntrack_max`, `nf_conntrack_buckets`) — this doesn't fix the race itself but reduces the *frequency* by reducing table pressure/collisions under load.

---

## 4. Service Mesh Fundamentals

### Sidecar injection mechanics

When a Deployment's Pod spec doesn't explicitly define a proxy container, Istio (or Linkerd) adds one automatically via a **MutatingWebhookConfiguration** registered against `pods` create events. The flow:

```
kubectl apply (Deployment/Pod spec, no sidecar defined)
        |
        v
API server admission chain reaches MutatingWebhookConfiguration
        |
        v
Webhook (istiod / linkerd-proxy-injector) receives AdmissionReview,
        inspects namespace/pod labels (e.g. istio-injection=enabled),
        returns a JSON patch
        |
        v
API server applies the patch: adds
    - an init container (istio-init / linkerd-init) — runs iptables setup, then exits
    - the sidecar container (istio-proxy / linkerd-proxy) — runs for the pod's lifetime
        |
        v
Pod scheduled with the extra containers already present in its spec
```

The **init container** is what wires traffic redirection: it runs `iptables` (or, in ambient/no-sidecar-iptables modes, eBPF) commands inside the pod's own network namespace (it runs with `NET_ADMIN` capability) to install rules that:

- Redirect all inbound traffic on the pod's listening ports to the sidecar's inbound proxy port (typically 15006 in Istio).
- Redirect all outbound traffic from the app container to the sidecar's outbound proxy port (15001), excluding traffic to/from the proxy's own UID to avoid infinite redirect loops.

Example of what that init container actually programs (Istio's `istio-iptables` tool, simplified):

```bash
iptables -t nat -N ISTIO_REDIRECT
iptables -t nat -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-port 15001
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
iptables -t nat -A ISTIO_INBOUND -p tcp --dport 15006 -j RETURN
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT
iptables -t nat -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN   # proxy's own traffic bypasses redirect
iptables -t nat -A ISTIO_OUTPUT -j ISTIO_REDIRECT
```

The upshot: **every packet the app container sends or receives is transparently routed through Envoy first**, without the application knowing or needing any code changes. This is exactly why sidecar mesh adoption requires zero app-level SDK integration but does add per-packet iptables traversal + a full userspace proxy hop (parse, evaluate policy, re-serialize, send) on every request.

### mTLS mechanics

Both Istio and Linkerd implement automatic mutual TLS between meshed workloads using workload identity rather than IP-based trust:

- Each workload is issued a **SPIFFE-format identity** (`spiffe://cluster.local/ns/default/sa/my-app` in Istio's case, tied to the pod's ServiceAccount) baked into the SVID (SPIFFE Verifiable Identity Document) — effectively a short-lived X.509 certificate whose SAN encodes that identity URI.
- The mesh's control plane (istiod, or Linkerd's identity component) acts as the CA, issuing certs to each sidecar over a secure channel at proxy startup and **rotating them frequently** (Istio's default cert TTL is 24h with rotation well before expiry, Linkerd defaults even shorter — this short lifetime is a deliberate security property, limiting the blast radius of a leaked cert).
- The sidecar proxies perform the TLS handshake and validate the peer's SPIFFE identity against mesh policy (`PeerAuthentication`/`AuthorizationPolicy` in Istio) — application code never sees the certs, doesn't do the handshake, and doesn't need any TLS library changes. Traffic between app and its own local sidecar is plaintext (localhost, same netns) — only the sidecar-to-sidecar hop over the wire is encrypted.

Inspect a workload's cert in Istio:

```bash
istioctl proxy-config secret <pod-name> -n <namespace>
openssl x509 -in <(istioctl pc secret <pod> -o json | jq -r '...') -noout -text | grep -A2 "Subject Alternative Name"
```

### Why adopt it / why not

Reasons to adopt:

- **Traffic shifting / canary routing** (weighted `VirtualService` routing, `TrafficSplit` in SMI/Linkerd) without touching app deploy pipelines.
- **Retries, timeouts, circuit breaking, outlier detection** implemented at the proxy layer uniformly across every service, regardless of language — no more re-implementing a retry-with-backoff library in five different languages inconsistently.
- **Uniform observability** — every meshed service gets consistent request-level metrics (latency histograms, error rates, golden signals) and distributed tracing propagation without app instrumentation, because the proxy sees every request.
- **mTLS everywhere** without every team wiring up their own cert management.

Operational costs to weigh honestly:

- **Latency**: each hop now traverses two extra userspace proxies (client sidecar out, server sidecar in) — typically single-digit milliseconds added per hop in well-tuned setups, but this compounds across deep call chains (a request touching 8 services in-mesh pays this tax 8 times).
- **Resource overhead**: a sidecar per pod means CPU/memory reservations multiply by pod count, not by node count — at high pod density this is a material capacity planning line item, not a rounding error.
- **Operational complexity**: another control plane to upgrade, another set of CRDs to understand, and failure modes that are genuinely harder to debug (is this timeout the app, the network, or a misconfigured `DestinationRule` connection pool setting?).
- **Ambient mesh modes** (Istio ambient, Linkerd's lighter dataplane options) exist specifically to address the per-pod sidecar tax by moving some functionality to shared per-node proxies — worth evaluating before committing to full sidecar-per-pod if resource overhead is the main objection.

---

## 5. MTU Mismatches in Overlay Networks

### The mechanism

Physical NIC MTU is typically 1500 bytes (Ethernet default). Overlay encapsulation adds headers on top of every packet:

- VXLAN: outer IP (20) + outer UDP (8) + VXLAN header (8) = **50 bytes overhead**
- IPIP: outer IP header only = **20 bytes overhead**
- Geneve (Cilium alt.): variable, typically similar ballpark to VXLAN

If the CNI doesn't correctly lower the **pod-facing interface's MTU** to account for this overhead (e.g. pod MTU should be `1500 - 50 = 1450` for VXLAN, not left at 1500), any packet from the pod that's close to the full 1500 bytes will, after encapsulation, exceed the physical link's 1500 MTU. Normally IP handles this transparently via **fragmentation** or **Path MTU Discovery (PMTUD)** — the oversized packet gets an ICMP "Fragmentation Needed" (type 3, code 4) sent back to the source, prompting a resend at a smaller size.

The failure mode: many cloud security groups, on-prem firewalls, and some default `iptables`/network ACL configs **block ICMP** entirely (or specifically block type-3 messages), which silently disables PMTUD. The oversized encapsulated packet gets dropped somewhere in the middle of its path (often at the outer IP's exit point where the physical MTU is actually enforced) with **no error signaled back to the sender**. The sending TCP stack just... never gets an ACK, retransmits, times out.

### Symptoms

This produces a very specific and confusing pattern:

- `curl` of small endpoints works fine, `ping` works fine (ICMP echo is small, well under any MTU).
- The initial TCP handshake (SYN/SYN-ACK/ACK, all tiny packets) succeeds.
- Then the connection **hangs or resets** specifically when a large payload needs to cross — a **TLS handshake** (certificate chain in the ServerHello can easily be several KB, spanning multiple full-size TCP segments) or a large HTTP response body flowing at full MSS-sized segments.
- Because small requests/responses work and only specific larger payloads fail, this gets misdiagnosed as an application bug, a TLS library issue, or "flaky networking" for a long time before someone thinks to check MTU.

### Diagnostic technique

**Binary search the actual working MTU using `ping` with the Don't-Fragment bit set:**

```bash
# -M do sets DF bit (don't fragment) so any drop tells you the real limit
# -s <size> is the ICMP payload size; total packet = payload + 28 bytes (8 ICMP + 20 IP)
ping -M do -s 1472 <target-ip>     # 1472+28=1500, standard MTU test
# ping: local error: message too long, mtu=1450        <- immediate feedback if local iface MTU is already smaller

ping -M do -s 1422 <target-ip>     # 1422+28=1450, the VXLAN-adjusted expected size
# 1450 bytes from <target-ip>: icmp_seq=1 ttl=63 time=0.412 ms   <- success

ping -M do -s 1450 <target-ip>     # test right at the boundary you suspect
```

Binary search between the sizes that succeed and fail to find the exact effective MTU on the path, then compare that against what the pod's interface actually reports:

```bash
kubectl exec -it mypod -- ip link show eth0
# mtu 1500     <- BUG: should be 1450 if VXLAN overlay is in use
```

If the pod's interface MTU is set correctly (1450) but you're still seeing drops, the problem is more likely somewhere in the physical path (a tunnel/VPN hop, a cloud interconnect, or an intermediate device silently capping MTU below what you configured) rather than the CNI's own config.

**Packet capture inside the pod's netns** — three equivalent ways to get a tcpdump running inside a pod's actual network namespace:

```bash
# Option 1: kubectl debug (ephemeral container sharing the pod's netns — needs a distro image with tcpdump)
kubectl debug -it mypod --image=nicolaka/netshoot --target=mypod -- tcpdump -i eth0 -n -s0 host <peer-ip>

# Option 2: nsenter from the node, if you can identify the container's PID
CONTAINER_PID=$(crictl inspect <container-id> | jq '.info.pid')
nsenter -t $CONTAINER_PID -n tcpdump -i eth0 -n -s0

# Option 3: ip netns exec, if the CNI leaves a named netns under /var/run/netns
ip netns list
ip netns exec cni-<uuid> tcpdump -i eth0 -n -s0
```

Look specifically for: TCP segments sized near the MTU boundary being retransmitted repeatedly with no corresponding ICMP "frag needed" ever showing up in the capture — that absence of an ICMP response, combined with retransmits of exactly the same size, is the signature of "ICMP blackholed, PMTUD broken" as opposed to a genuine congestion/loss issue (which would show more varied retransmit sizes and other loss indicators like out-of-order segments unrelated to size).

### The fix

- Set the CNI's configured MTU correctly (`calicoctl`/Cilium's `tunnel-protocol` + `mtu` config, Flannel's `net-conf.json` `MTU` field) to account for encapsulation overhead — most CNIs will auto-detect and subtract if configured to do so, but manually verify rather than trust silently, especially after changing the underlying instance type/NIC (some cloud NICs support jumbo frames at 9000 MTU, which changes the math).
- Where you can't guarantee ICMP isn't filtered end-to-end (multi-cloud, VPN peering, etc.), don't rely on PMTUD at all — get the MTU numbers right at the source so oversized packets are never generated in the first place.
- If using IPIP/VXLAN across a path with an already-reduced MTU (e.g. underlying transit network itself is already sub-1500, common on some VPN/SD-WAN underlays), the overlay's already-reduced MTU calculation needs to subtract from *that* real path MTU, not naively from 1500.

---

## 6. NetworkPolicy Implementation Reality

### The API server does nothing

This is the single most important fact to internalize about `NetworkPolicy`: it is **just a CRD-like API object stored in etcd**. The API server validates the object's schema and persists it. That's it. There is no controller in the API server, no admission webhook by default, no built-in enforcement path. `kubectl apply -f netpol.yaml` succeeding with `networkpolicy.networking.k8s.io/deny-all created` tells you the object was **accepted and stored** — it tells you **nothing** about whether any enforcement is actually happening on the wire.

Enforcement is entirely the responsibility of whatever CNI plugin's node-agent is watching NetworkPolicy objects and translating them into actual dataplane rules:

- **Calico**: `felix` (the per-node agent) watches NetworkPolicy (and Calico's own richer `GlobalNetworkPolicy`/`NetworkPolicy` CRDs) via the Typha/datastore layer and programs iptables (or eBPF, in Calico's eBPF dataplane mode) rules to actually match and drop/allow traffic.
- **Cilium**: `cilium-agent` watches NetworkPolicy and translates it (along with `CiliumNetworkPolicy` for richer L3-L7 rules) into eBPF policy maps enforced at the `tc`/socket hooks already described.
- **Plain Flannel**: has **no agent that watches NetworkPolicy at all**. There is no Felix-equivalent, no eBPF policy engine, nothing.

### The footgun

If your cluster runs plain Flannel (or any CNI without a policy engine) and someone applies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

...the API server accepts it. `kubectl get networkpolicy -n production` shows it. Security tooling that only checks "does a default-deny policy exist" reports green. **And it does absolutely nothing.** Every pod in that namespace remains fully reachable from anywhere in the cluster (and depending on your CNI's egress handling, potentially still reachable from outside too), because there's no agent translating that object into an actual iptables/eBPF rule anywhere. There is no error, no event, no admission rejection, no status condition telling you this — the object just silently has zero effect. This is a real, common, and dangerous gap in clusters that were originally stood up with Flannel for simplicity and later had NetworkPolicy objects added by a security team or compliance tool without anyone verifying enforcement actually exists.

### Verifying your CNI actually enforces policy

Don't trust the manifest — test it:

```bash
# 1. Apply a deny-all policy in a throwaway namespace
kubectl create ns netpol-test
kubectl label ns netpol-test purpose=test
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
EOF

# 2. Run a target pod and a client pod
kubectl run target -n netpol-test --image=nginx --port=80
kubectl run client --image=busybox -it --rm -- wget -qO- --timeout=3 <target-pod-ip>

# 3. Interpret the result:
#    - timeout/connection refused  -> policy IS enforced, CNI has a working policy engine
#    - successful response         -> policy is NOT enforced, CNI is silently ignoring it
kubectl delete ns netpol-test
```

Also check directly whether the CNI's control components are even running:

```bash
# Calico
kubectl get pods -n calico-system -l k8s-app=calico-node
kubectl exec -n calico-system <calico-node-pod> -- calico-node -felix-live  # felix health check

# Cilium
cilium-dbg status | grep -i "Policy"
kubectl exec -n kube-system <cilium-agent-pod> -- cilium-dbg endpoint list | grep -i enforcement
```

If there's no Felix, no cilium-agent, no equivalent policy-enforcing daemon in your cluster's system namespaces at all, treat every existing `NetworkPolicy` object in that cluster as **decorative** until proven otherwise by the connectivity test above. This is exactly the kind of gap that passes a manifest-based audit ("yes we have default-deny policies defined") while providing zero actual isolation — verify on the wire, not in etcd.
