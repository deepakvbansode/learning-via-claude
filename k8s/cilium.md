# Cilium — eBPF-based K8s Service Mesh

## The Problem It Solves

Traditional K8s networking relies on iptables (O(n) rule chains, brittle at scale) and sidecar proxies (50–100MB RAM per pod, 2 extra hops per request, 1000 pods = 1000 Envoys). Cilium replaces both by pushing networking, security policy, and observability into the Linux kernel via eBPF programs — making it faster, leaner, and fully observable without touching application code.

---

## Packet Journey: Normal VM → K8s Pod → eBPF → Cilium

Understanding where Cilium hooks in requires understanding the full packet path — from a bare VM up through K8s pods.

---

### 1. Packet Journey on a Normal VM

```text
INGRESS (external → app on VM):

  [ Physical NIC ]
       │  packet arrives
       ▼
  [ Driver / Ring Buffer ]       — NIC driver hands packet to kernel
       │
       ▼
  [ Kernel Network Stack ]
       │
       ├── Netfilter / iptables  — PREROUTING → FORWARD/INPUT → POSTROUTING
       │   (rule chain walked linearly — O(n) per packet)
       │
       ▼
  [ Routing Table ]              — where does this packet go?
       │
       ▼
  [ Socket Layer ]               — delivers to process listening on port
       │
       ▼
  [ Application ]


EGRESS (app on VM → external):

  [ Application ]
       │
       ▼
  [ Socket Layer ]
       │
       ▼
  [ Routing Table ]
       │
       ▼
  [ Netfilter / iptables ]       — OUTPUT → POSTROUTING
       │
       ▼
  [ Physical NIC ]               — packet leaves the machine
```

Key point: iptables sits in the hot path of every packet. Each rule is evaluated sequentially — 500 rules = 500 comparisons per packet, every time.

---

### 2. Packet Journey in K8s (Pod-to-Pod, same node)

K8s adds virtual networking on top of the VM stack. Each pod lives in its own network namespace, connected via virtual ethernet pairs.

```text
INGRESS (external → pod):

  [ Physical NIC (eth0) ]
       │
       ▼
  [ Kernel Network Stack ]
       │
       ▼
  [ iptables / kube-proxy ]      — DNAT: Service ClusterIP → Pod IP
       │                            (walks iptables rules to find backend pod)
       ▼
  [ Node Routing Table ]
       │
       ▼
  [ CNI Bridge (cni0 / cbr0) ]   — virtual L2 bridge on the node
       │
       ▼
  [ veth pair ]                  — virtual ethernet cable
       │  (host side: vethXXXXXX)
       │  (pod side:  eth0 inside pod netns)
       ▼
  [ Pod Network Namespace ]
       │
       ▼
  [ Application ]


EGRESS (pod → external):

  [ Application ]
       │
       ▼
  [ Pod Network Namespace (eth0) ]
       │
       ▼
  [ veth pair ]                  — exits pod netns
       │
       ▼
  [ CNI Bridge (cni0) ]
       │
       ▼
  [ iptables / kube-proxy ]      — SNAT: Pod IP → Node IP (masquerade)
       │
       ▼
  [ Physical NIC (eth0) ]
```

Key point: kube-proxy maintains iptables rules for every Service and Endpoint. At 1000 Services × 5 pods each = 5000+ iptables rules walked on every packet.

---

### 3. How eBPF Fits In

eBPF lets you attach programs to kernel hooks that fire in the packet path — before iptables, before routing, as early as when the packet hits the NIC.

```text
INGRESS with eBPF:

  [ Physical NIC ]
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  XDP Hook  (eXpress Data Path)          │  ← fires BEFORE sk_buff allocation
  │  • Fastest interception point           │    kernel hasn't touched the packet yet
  │  • Actions: DROP, PASS, REDIRECT, TX    │    — perfect for DDoS mitigation
  └─────────────────────────────────────────┘
       │  PASS
       ▼
  [ sk_buff allocated ]                        ← kernel allocates memory for packet
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  TC Ingress Hook (Traffic Control)      │  ← fires after sk_buff, before routing
  │  • Full packet context available        │    can inspect headers + payload
  │  • Actions: DROP, REDIRECT, modify      │
  └─────────────────────────────────────────┘
       │
       ▼
  [ Netfilter / iptables ]                     ← traditional path (can be bypassed)
       │
       ▼
  [ Routing → Socket → Application ]


EGRESS with eBPF:

  [ Application → Socket ]
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  TC Egress Hook (Traffic Control)       │  ← fires before packet leaves node
  │  • Stamp identity into packet header    │
  │  • Apply egress policy                  │
  └─────────────────────────────────────────┘
       │
       ▼
  [ Netfilter / iptables ]                     ← can be bypassed
       │
       ▼
  [ Physical NIC ]


BPF Map — the O(1) lookup table:

  XDP Hook ──┐
             ├──► [ BPF Map ]   { identity → policy decision }
  TC Hook  ──┘     hash table in kernel memory, updated by Cilium agent
                   lookup is O(1) vs iptables O(n)
```

---

### 4. How Cilium Fits In (Full Picture)

Cilium replaces kube-proxy and the CNI bridge model with eBPF programs loaded by a DaemonSet pod on every node.

```text
CILIUM INGRESS (external → pod, cross-node):

  [ Physical NIC (eth0) ]
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  XDP Hook                               │  ← Cilium: early DROP for denied identities
  └─────────────────────────────────────────┘
       │
       ▼
  [ sk_buff allocated ]
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  TC Ingress Hook                        │  ← Cilium: read identity from packet header
  │                                         │    check BPF map → ALLOW or DROP
  │                                         │    replaces kube-proxy DNAT (BPF does it)
  └─────────────────────────────────────────┘
       │
       ▼
  [ iptables BYPASSED ]                        ← Cilium skips Netfilter entirely
       │
       ▼
  [ veth pair → Pod Network Namespace ]
       │
       ▼
  [ Application ]


CILIUM EGRESS (pod → external, cross-node):

  [ Application ]
       │
       ▼
  [ Pod Network Namespace ]
       │
       ▼
  ┌─────────────────────────────────────────┐
  │  TC Egress Hook (on pod's veth)         │  ← Cilium: stamp numeric identity
  │                                         │    into packet header
  │                                         │    apply egress CiliumNetworkPolicy
  └─────────────────────────────────────────┘
       │
       ▼
  [ iptables BYPASSED ]
       │
       ▼
  [ Physical NIC ]
       │  (WireGuard/IPSec encryption applied here for cross-node mTLS)
       ▼
  [ Wire → Destination Node ]


CILIUM NODE ARCHITECTURE:

  ┌──────────────────────────────────────────────────────┐
  │  Node                                                │
  │                                                      │
  │   Pod A          Pod B          Pod C                │
  │   [app]          [app]          [app]                │
  │     │              │              │                  │
  │   veth           veth           veth                 │
  │     └──────────────┴──────────────┘                  │
  │                    │                                 │
  │              eBPF programs                           │
  │         (no bridge, no kube-proxy)                   │
  │                    │                                 │
  │   ┌────────────────────────────────────┐             │
  │   │  Cilium Agent (DaemonSet pod)      │             │
  │   │  • loads eBPF programs into kernel │             │
  │   │  • syncs BPF maps from K8s state   │             │
  │   │  • manages WireGuard keys          │             │
  │   │  • runs node-local Envoy (L7 only) │             │
  │   │  • feeds Hubble ring buffer        │             │
  │   └────────────────────────────────────┘             │
  │                    │                                 │
  │              Physical NIC                            │
  └──────────────────────────────────────────────────────┘
```

---

## Core Concepts

---

### 1. eBPF in Cilium's Context

**Why:** iptables rules are O(n) and require full chain rewrites on every change. At scale (1000+ Services), this is a performance and operational bottleneck for both routing and policy enforcement.

**Mental model:** iptables is a long `if-else` chain evaluated at runtime. Cilium's BPF maps are like a Go map — O(1) lookup, pre-compiled, evaluated directly in the kernel datapath before the packet touches the networking stack.

**Example:**

```text
Traditional (kube-proxy + iptables):
  packet → walk 5000 iptables rules → find destination pod → forward

Cilium (eBPF):
  packet → XDP/TC hook → BPF map lookup (O(1)) → forward
```

---

### 2. Identity-based Networking

**Why:** Pod IPs are ephemeral — pod restarts, reschedules, scales up and gets a new IP. IP-based firewall rules break silently and require constant maintenance.

**Mental model:** Like JWT tokens for pods. The identity (a numeric ID derived from K8s labels) travels in the packet header — not the IP. Think of K8s RBAC `RoleBinding` but for network traffic: you bind rules to label selectors, Cilium resolves which identities match at runtime. Identities are distributed cluster-wide via etcd/CRDs so every node can enforce policy locally.

**Example:**

```text
{ app: payments, env: prod } → identity 1042

TC egress hook stamps 1042 into packet header as pod sends it.
Receiving node's TC ingress hook reads 1042, looks up BPF map:
  1042 → allowed to reach db:5432 → ALLOW

Pod restarts, gets new IP 10.0.0.9, same labels → still identity 1042.
Policy unchanged. No rules to update.
```

---

### 3. CiliumNetworkPolicy (L3/L4/L7)

**Why:** Standard K8s `NetworkPolicy` only sees IP + port (L3/L4). You can't prevent a compromised `payments` pod from calling `DELETE /users` even though it legitimately needs `GET /users`. L7 policy adds request-level control.

**Mental model:** Like Fastly VCL rules — L3/L4 is IP-based ACLs (who can connect where), L7 is request routing and inspection (what they can do once connected).

| Layer | What it sees | Example rule |
| ----- | ------------ | ------------ |
| L3 | Pod identity | `payments` identity can reach `db` identity |
| L4 | Port + protocol | only on TCP port 5432 |
| L7 | Request content | only HTTP `GET` to `/api/*` |

**Example:**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: payments-to-db
  namespace: finance
spec:
  endpointSelector:
    matchLabels:
      app: db                      # policy applies TO db pods
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: payments              # L3: only from payments identity
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP              # L4: only this port
      rules:
        http:                      # L7: only these requests
        - method: GET
          path: /api/health

# Namespace isolation — all pods in hr cannot reach any pod in finance:
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: isolate-finance
  namespace: finance
spec:
  endpointSelector: {}             # applies to ALL pods in finance namespace
  ingress:
  - fromEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: finance  # only finance pods allowed in
```

---

### 4. Sidecarless Service Mesh

**Why:** Istio/Linkerd inject an Envoy sidecar per pod — 50–100MB RAM per pod, 2 extra hops per request, 1000 pods = 1000 Envoy processes to manage with version mismatches, injection failures, and startup ordering bugs.

**Mental model:** Istio puts a traffic cop inside every car. Cilium embeds the traffic cop into the road — one eBPF program per node (via DaemonSet), shared by all pods on that node. For L7 features, one Envoy per *node* (not per pod): 20 nodes = 20 Envoys instead of 1000.

**Example:**

```text
ISTIO:
  [App Pod]
      └── [Envoy Sidecar]  ← per-pod proxy, userspace, 50-100MB RAM
              │
      [Kernel network stack]

CILIUM:
  [App Pod]
      │
  [Kernel network stack]
      │
  [eBPF program]           ← one per node, kernel-space, near-zero overhead
      │
  [BPF maps]               ← policy + identity + metrics

Cilium DaemonSet (one pod per node):
  ├── loads eBPF programs into kernel
  ├── mTLS via WireGuard/IPSec between nodes (transparent to app)
  └── one node-local Envoy only if L7 policy is applied on that node
```

---

### 5. Hubble Observability

**Why:** Traditional observability requires SDK instrumentation in your app. But eBPF programs intercept every packet in the kernel — Hubble taps that stream to give you full flow visibility with zero app changes. It catches misbehaving pods that wouldn't emit traces anyway.

**Mental model:** Distributed tracing at the kernel level — like having a wiretap on every network interface in the cluster, aggregated into a live service map. Unlike Jaeger/Zipkin which require SDK headers, Hubble sees all traffic unconditionally.

**Example:**

```text
Architecture:
  Hubble Agent (per node, reads eBPF ring buffer)
      │
  Hubble Relay (aggregates across all nodes, gRPC)
      │
  Hubble UI (live service map) + CLI + Prometheus metrics


CLI:
  hubble observe                              # live flow log, all namespaces
  hubble observe -n finance                  # flows in finance namespace
  hubble observe --verdict DROPPED           # debug what's being denied
  hubble observe --from-label app=payments   # flows from specific workload
  hubble observe --protocol http             # HTTP flows only

Sample output:
  payments → db:5432     ALLOW   (identity 1042 → 2081)
  hr → finance:8080      DROPPED (policy)
  payments → DNS:53      ALLOW   query: payments-db.svc.cluster.local
```

---

## Gotchas

1. **L7 policy always costs an Envoy** — even a single HTTP rule on one pod spins up a node-local Envoy. Don't apply L7 policy unless you actually need request-level control; pure L3/L4 is zero-proxy eBPF.

2. **XDP is not always available** — some cloud providers or virtual NIC drivers don't support native XDP mode. Cilium falls back to TC-only silently. Check `cilium status` for the XDP mode reported.

3. **Identity distribution lag** — new pod identity propagates via etcd/CRDs. During fast scale-up, there's a brief window where policy may not apply to brand-new pods. Design for this in zero-trust environments.

4. **Hubble ring buffer overwrites** — fixed size per node, old data is lost. Great for live debugging, not for compliance audit logs. Export flows to Loki/Elasticsearch for retention.

5. **Replacing kube-proxy is optional but important** — Cilium can run alongside kube-proxy, but you only get full eBPF performance (no iptables in the hot path) by disabling kube-proxy and letting Cilium handle Service load balancing entirely via BPF maps.

6. **L7 Kafka/gRPC policy requires protocol parsing** — Cilium must decrypt and re-encrypt TLS for L7 inspection. This works transparently but adds CPU overhead and requires careful consideration for high-throughput topics.

---

## Quick Reference

**Key CLI commands:**

```text
cilium status                          # health, XDP mode, encryption, BPF map usage
cilium endpoint list                   # all pods + their numeric identity
cilium monitor                         # raw eBPF event stream (low-level debug)
hubble observe                         # live flow logs
hubble observe --verdict DROPPED       # debug policy drops
hubble observe -n <namespace>          # namespace-scoped flows
hubble observe --from-label app=X      # flows from specific workload
```

**Policy selectors:**

```yaml
endpointSelector: {}                                          # all pods in namespace
matchLabels: { app: payments }                                # specific workload
matchLabels: { k8s:io.kubernetes.pod.namespace: finance }    # all pods in namespace
```

**Packet hook points:**

```text
NIC → [XDP] → sk_buff → [TC ingress] → stack → [TC egress] → NIC
       ↑                    ↑                       ↑
  before sk_buff      identity check           identity stamp
  fastest drop        policy enforce           egress policy
```

**L7 protocol support:**

| Protocol | Cilium rule granularity |
| -------- | ----------------------- |
| HTTP/1.1, HTTP/2 | method, path, headers |
| gRPC | service name, method name |
| Kafka | topic, role (produce/consume) |
| DNS | FQDN-based egress allowlist |

**Identity flow:**

```text
Pod labels → numeric identity (e.g. 1042)
           → distributed to all nodes via etcd/CRDs
           → stamped into packet header at TC egress
           → checked against BPF map at TC ingress of receiving node
           → ALLOW or DROP without touching iptables
```

**Sidecar comparison:**

| | Istio | Cilium |
| --- | --- | --- |
| Enforcement | Envoy sidecar per pod | eBPF program per node |
| mTLS | Sidecar handles | WireGuard/IPSec at node level |
| L7 proxy | 1 per pod | 1 per node (only if L7 policy used) |
| Observability | Sidecar emits traces | Kernel eBPF ring buffer |
| App changes needed | None (injection) | None (kernel-level) |
| RAM overhead | 50–100MB per pod | Near-zero (kernel programs) |

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

> I know Cilium well. Quick context: Cilium replaces kube-proxy and sidecar proxies by pushing networking, policy enforcement, and observability into the Linux kernel via eBPF programs hooked at XDP (pre-sk_buff) and TC (traffic control) points. Pods get label-derived numeric identities embedded in packet headers — not IPs — so policy survives pod churn. One Cilium DaemonSet pod per node handles everything: mTLS via WireGuard, L3/L4/L7 CiliumNetworkPolicy, and Hubble flow observability without touching app code. kube-proxy is replaced by BPF maps (O(1) vs O(n) iptables).
> I'm building: [describe your task here]
