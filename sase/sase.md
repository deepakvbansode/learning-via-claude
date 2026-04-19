# SASE — Deep Reference

## The Problem It Solves

Traditional security assumed users and apps lived inside a corporate network — one trusted perimeter protected everything. When workforces went remote and apps moved to the cloud, that perimeter dissolved. Traffic had to backhaul through HQ for inspection, VPN credentials gave attackers full network access, and 5+ siloed tools created policy gaps and blind spots. SASE collapses networking and security into a single cloud-native framework delivered from edge PoPs — close to the user, consistently enforced, with one control plane.

---

## Core Concepts

---

### 1. Zero Trust Philosophy

**Why it exists:** The castle-and-moat model fails the moment credentials are stolen — the attacker is "inside" and can reach everything. Zero Trust eliminates the concept of a trusted "inside." Every request is verified regardless of where it comes from.

**Three pillars:**

- **Verify explicitly** — authenticate and authorize based on identity, device health, location, and behavior — every request, every time
- **Least privilege** — grant access only to what's needed, nothing more
- **Assume breach** — design as if the attacker is already inside; contain blast radius

**Analogy:** Like Kubernetes admission controllers — every pod creation is validated against RBAC, PodSecurityPolicy, and webhooks regardless of origin. Being "inside the cluster" grants nothing by itself.

```text
Attacker has stolen password:
  ✓ Password correct
  ✗ Device not enrolled in MDM
  ✗ Location: unusual country, 3am
  → ACCESS DENIED — credentials alone are never enough

Even if all checks pass:
  → Access granted to ONE specific app only (least privilege)
  → Blast radius contained
```

---

### 2. SASE Architecture

**Why it exists:** 5+ siloed tools (VPN, firewall, SWG, CASB, SD-WAN) with separate dashboards, separate policies, and gaps between them created inconsistency, blind spots, and operational overhead. A single control plane enforces consistent policy everywhere and gives full correlated visibility.

**The formula:**

```text
SASE = SD-WAN (networking) + SSE (security)
SSE  = ZTNA + SWG + CASB + FWaaS + DLP
```

**Analogy:** Like a CDN — instead of routing all traffic to one origin, deploy PoPs globally so users hit the nearest one. Each PoP runs the full security stack in a single inspection pass. One control plane = one policy enforced everywhere, like K8s RBAC across all nodes.

```text
OLD WORLD:
User (Bangalore) → VPN → HQ (Mumbai) → Firewall → SWG → App
                   [backhauled across the planet, high latency]

SASE:
User (Bangalore) → Nearest PoP (Singapore) → [auth + inspect + route] → App
                   [one hop, one inspection pass, minimal latency]
```

Single policy applied at every PoP worldwide:

```text
Policy: "user:deepak, device:trusted, location:india → allow:staging-app, inspect:all-traffic"
→ Enforced identically at every PoP
→ All logs flow to one control plane — full correlated visibility
```

---

### 3. ZTNA — Zero Trust Network Access

**Why it exists:** VPN gives full network access after one login — stolen credentials expose the entire internal network. ZTNA gives access to specific apps only. The app itself is dark to the internet (no public IP, no open port), so there is nothing to scan or attack directly.

**VPN vs ZTNA:**

```text
VPN:  User → login once → full network access (can scan 10.0.0.0/8)
ZTNA: User → request App-X → verify identity + device + location
           → proxied session to App-X only
           → user never sees App-X's real IP or the network behind it
           → next request to App-Y → verify again from scratch
```

**Analogy:** Like a Kubernetes NetworkPolicy with default-deny — no connection is allowed unless explicitly permitted. The app's connector dials _out_ to the ZTNA broker (no inbound ports open), like a K8s operator polling the API server rather than exposing an endpoint.

```text
App side:  Connector → dials OUT to ZTNA broker (no inbound ports)
User side: Request app → broker verifies identity + device + location
                      → stitches user session to connector
                      → user sees app, never sees its IP or network

VPN: contractor can scan entire 10.0.0.0/8, find every internal service
ZTNA: contractor sees exactly one app — nothing else exists
```

**Two deployment modes:**

- **Agent-based** — client app on device; better device posture checking
- **Agentless** — browser-based, no install; easier for contractors/BYOD, weaker device checks

---

### 4. SD-WAN — Software-Defined Wide Area Network

**Why it exists:** MPLS circuits are rigid and expensive — all traffic backhauled to HQ even when SaaS is reachable directly from the branch. SD-WAN uses multiple links simultaneously and steers each flow to the best path in real time based on what the application requires.

**The core decision:** For every traffic flow, SD-WAN asks — _what does this app require?_ — and routes accordingly based on latency, packet loss, bandwidth, and cost.

**Analogy:** Like Fastly/Cloudflare traffic steering — route each request to the best origin based on real-time health checks. SD-WAN does this for WAN links instead of origin servers.

```text
Links: Broadband 200Mbps (cheap, variable) | MPLS 50Mbps (stable) | LTE (backup)

Zoom calls  → MPLS      (stable, latency-sensitive — SLA critical)
Salesforce  → Broadband (SaaS tolerates variability, save cost)
DB backups  → Broadband (latency irrelevant)

Broadband fails mid-backup:
  → Auto-failover to MPLS in <1 second, no manual intervention
  → SD-WAN deprioritizes backup traffic to protect Zoom quality
  → Zoom and backup now share 50Mbps — Zoom wins
```

---

### 5. SSE — SWG + CASB

**Why it exists:** ZTNA controls _who_ accesses _what app_, but not whether the traffic content is safe or policy-compliant. A legitimate user with valid credentials can still download malware, visit phishing sites, or exfiltrate sensitive data to personal cloud storage. SSE inspects the content of all traffic at the PoP.

**Two components, two different inspection layers:**

|               | SWG                                 | CASB                                       |
| ------------- | ----------------------------------- | ------------------------------------------ |
| **Scope**     | All web traffic (HTTP/HTTPS)        | SaaS apps specifically                     |
| **Awareness** | Protocol-aware                      | Application-semantics-aware                |
| **Catches**   | Malware, phishing, bad domains, DLP | Policy violations, bulk exports, shadow IT |

**Analogy:** SWG is like a Kubernetes NetworkPolicy for web traffic — allow/deny by domain, category, or content signature. CASB is like an admission webhook — it understands the _semantic meaning_ of the request, not just the protocol.

```text
Employee uploads customer DB (names, emails, phones) to personal Google Drive:

SWG:  drive.google.com → not malicious → ALLOW
CASB: Google Drive Upload API + PII detected → BLOCK

You need both:
  SWG  → handles threats (malware, phishing, bad domains)
  CASB → handles compliance (what data goes where on legitimate services)
```

**CASB two modes:**

- **Inline** — sits in the traffic path, blocks in real time
- **API-based** — connects to SaaS APIs after the fact, discovers historical violations and shadow IT

---

### 6. FWaaS — Firewall as a Service

**Why it exists:** Traditional hardware firewalls sit at HQ — all traffic must backhaul for deep packet inspection, eliminating the latency advantage of edge PoPs. Not all traffic is HTTP/HTTPS either — SWG can't inspect raw TCP, UDP, or exploit payloads in non-web protocols. FWaaS runs full network-level inspection at every PoP, cloud-native, with automatic threat signature updates.

**What FWaaS inspects that SWG doesn't:**

```text
SWG:   Layer 7, HTTP/HTTPS only — "is this web content safe?"
FWaaS: Layers 3-7, all protocols  — "are these packets themselves an attack?"

FWaaS catches:
  - IPS signatures: TCP payload matches Log4Shell exploit pattern → DROP
  - Port/protocol rules: block unexpected protocols
  - DNS security: block queries to known C2 (command-and-control) domains
  - Non-HTTP protocols: SSH, FTP, custom TCP — SWG cannot touch these
```

**Analogy:** Like moving from a self-hosted Nginx on one VM to a globally distributed load balancer — same function, delivered at the edge. IPS signatures update automatically (hourly vs. weeks-long manual patch cycles on hardware).

```text
Legitimate user (passed ZTNA, clean device) → carries Log4Shell exploit payload:

ZTNA:  identity ✓, device ✓ → ALLOW
SWG:   HTTP domain legitimate → ALLOW
FWaaS: TCP payload matches Log4Shell IPS signature → DROP + ALERT
```

**Hardware vs FWaaS:**

| Aspect            | On-Prem Firewall         | FWaaS                     |
| ----------------- | ------------------------ | ------------------------- |
| Deployment        | Physical appliance at HQ | Cloud-native at every PoP |
| Signature updates | Manual (weeks behind)    | Automatic (hourly)        |
| Scaling           | Buy bigger box           | Elastic, auto-scales      |
| Latency           | Backhaul to HQ           | Inspected at nearest PoP  |

---

## Gotchas

1. **SASE is a framework, not a product.** You can't install "SASE 5.0." Vendors (Cloudflare, Zscaler, Palo Alto) approximate it — evaluate which components are native vs. bolted on.

2. **SSL inspection requires a corporate root cert on every device.** SWG must decrypt HTTPS to inspect it. Employees' encrypted traffic becomes readable by the company. Needs legal review and employee disclosure before enabling.

3. **CASB inline vs. API — you need both.** Inline blocks violations in real time. API-based discovers historical violations and shadow IT. One without the other leaves gaps.

4. **SD-WAN controller is a single point of failure.** Always deploy HA. Edge devices fall back to static routes if the controller goes down — traffic keeps flowing but loses intelligent steering.

5. **FWaaS ≠ ZTNA.** ZTNA controls access (who reaches what app). FWaaS inspects payload (are the packets an attack). A legitimate user with valid access can carry an exploit — only FWaaS catches it.

6. **ZTNA doesn't replace network segmentation.** ZTNA is app-level access control. Databases and internal APIs still need firewall rules. Defense-in-depth: both layers, not either/or.

---

## Quick Reference

**Acronyms:**

| Acronym | Full Form                           |
| ------- | ----------------------------------- |
| SASE    | Secure Access Service Edge          |
| SSE     | Security Service Edge               |
| ZTNA    | Zero Trust Network Access           |
| SD-WAN  | Software-Defined Wide Area Network  |
| SWG     | Secure Web Gateway                  |
| CASB    | Cloud Access Security Broker        |
| FWaaS   | Firewall as a Service               |
| DLP     | Data Loss Prevention                |
| IPS     | Intrusion Prevention System         |
| PoP     | Point of Presence                   |
| VPN     | Virtual Private Network             |
| MPLS    | Multiprotocol Label Switching       |
| MDM     | Mobile Device Management            |
| MFA     | Multi-Factor Authentication         |
| SSO     | Single Sign-On                      |
| HA      | High Availability                   |
| C2      | Command and Control                 |
| PII     | Personally Identifiable Information |
| WAN     | Wide Area Network                   |

**The SASE equation:**

```text
SASE = SD-WAN + SSE
SSE  = ZTNA + SWG + CASB + FWaaS + DLP
```

**What each layer answers:**

```text
ZTNA    → Who can access which app?
SD-WAN  → What path does traffic take?
SWG     → Is the web content/domain safe?
CASB    → Is SaaS usage policy-compliant?
FWaaS   → Are the packets themselves an attack?
```

**Component scope:**

| Component | Protocol scope | What it catches                                          |
| --------- | -------------- | -------------------------------------------------------- |
| ZTNA      | All            | Unauthorized access, unverified identity/device          |
| SWG       | HTTP/HTTPS     | Malware, phishing, bad domains, outbound DLP             |
| CASB      | SaaS APIs      | Data exfiltration, shadow IT, compliance violations      |
| FWaaS     | All (L3–L7)    | Exploit payloads, IPS signatures, non-HTTP threats       |
| SD-WAN    | All            | Suboptimal routing — not a security tool, pairs with SSE |

**Zero Trust decision flow:**

```text
Request arrives:
  1. Identity verified (SSO + MFA)?         → No  → DENY
  2. Device healthy (MDM, patches)?          → No  → DENY
  3. Location/behavior normal?               → No  → DENY
  4. User authorized for this specific app?  → No  → DENY
  5. Traffic content safe? (SWG + FWaaS)     → No  → DROP
  6. SaaS action policy-compliant? (CASB)    → No  → BLOCK
  → ALLOW (scoped to this app only)
```

---
