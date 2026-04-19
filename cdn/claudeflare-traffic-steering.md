# Cloudflare CDN Traffic Management

## The Problem It Solves

Your origin sits in one region. Users are global. Cloudflare puts a global edge network between users and your origin — handling routing, failover, and edge logic before traffic reaches your cluster. Unlike Fastly (pure CDN via CNAME), Cloudflare combines DNS authority + CDN proxy in one platform.

---

## Core Concepts

### Zone + DNS Proxy (Orange Cloud)

**Why:** Cloudflare needs to be in the traffic path to apply routing, WAF, caching. It achieves this by becoming your authoritative DNS server and answering queries with its own edge IPs instead of your origin IP.

**Mental model:** In Fastly you add a CNAME per hostname in NS1 — NS1 stays in charge. In Cloudflare you transfer nameservers for the whole domain to Cloudflare. Then per DNS record you toggle orange cloud (proxied through Cloudflare) or gray cloud (DNS-only, Cloudflare invisible). Orange cloud = on switch for everything.

**Exception:** Business/Enterprise plans support CNAME setup — keeps NS1 as authority, just CNAME individual hostnames to Cloudflare. Identical to Fastly's model.

```
Zone: example.com
  ├── api.example.com  → [orange] → Load Balancer → Pools → Origins
  └── smtp.example.com → [gray]  → your mail server IP directly
```

---

### Origins, Pools & Monitors

**Why:** You need named upstream servers grouped for health-aware routing, with reusable health check configs.

**Mental model:** Origins = Fastly Backends. Pools = Fastly Directors (local steering within the group). Monitors = Fastly Probes, but decoupled — defined once, attached to a Pool, Cloudflare probes each origin in the pool using that Monitor's config.

```
Monitor: healthcheck-v1  (path: /healthz, interval: 60s)

Pool: api-eu  ← Monitor attached here
  ├── origin: eu1.example.com
  └── origin: eu2.example.com
```

---

### Load Balancer

**Why:** You need a single entry point per hostname that holds the pool list and global routing policy.

**Mental model:** The Load Balancer IS the DNS record (orange cloud created automatically). It's the decision-maker for which Pool serves a request. One Load Balancer = one hostname — unlike Fastly Service which can serve multiple domains.

```
Load Balancer: api.example.com
  ├── Steering policy: Geo
  ├── Pool priority: [eu-pool, us-pool]
  ├── Default pool: us-pool
  └── Fallback pool: us-pool  ← last resort, serves even if unhealthy
```

---

### Traffic Steering Policies

**Why:** Different traffic patterns need different routing strategies — geography, load, latency, weight.

**Mental model:** Global steering (Load Balancer → Pool) replaces Fastly's `vcl_recv` logic. Local steering (Pool → Origin) replaces Fastly Director's policy. Two levels, split responsibilities.

**Global steering policies:**

| Policy | Use when |
|--------|----------|
| Off (Failover) | Always route to first healthy pool in priority order |
| Geo | Route by user's country/region to mapped pool |
| Random | Weighted split across pools (canary deployments) |
| Dynamic (Argo) | Route to lowest-latency pool, real-time measured |
| Proximity | Route to geographically nearest pool |
| Least Outstanding Requests | Route to pool with fewest in-flight requests |

**Failover chain:** When mapped pool is unhealthy → Default pool → next healthy pool in priority order → Fallback pool (serves even unhealthy) → 503.

---

### Workers

**Why:** Declarative steering policies can't handle header-based routing, cookie A/B testing, URL rewriting, or async external lookups before routing.

**Mental model:** VCL `vcl_recv` is a DSL hook. Workers is a full JavaScript/WASM handler — like a Go `net/http` middleware but running at the edge with async support. Runs before the Load Balancer.

```javascript
export default {
  async fetch(request) {
    if (request.headers.get("X-Beta-User") === "true") {
      // route to canary via its load balancer hostname
      return fetch("https://canary-lb.example.com" + new URL(request.url).pathname);
    }
    return fetch("https://api-lb.example.com" + new URL(request.url).pathname);
  }
}
```

---

## Gotchas

1. **Orange cloud is the master switch** — if it's off for a record, no Load Balancing, no Workers, no WAF works for that hostname
2. **One Load Balancer = one hostname** — unlike Fastly Service, you can't serve multiple domains from one LB
3. **Monitor attaches to Pool, not Origin** — Cloudflare probes each origin in the pool using the pool's monitor config
4. **Workers bypass the Load Balancer** — `fetch(request)` goes direct to origin; explicitly route to your LB hostname to keep it in chain
5. **Geo steering needs a Default pool** — without it, unmatched regions and failed geo-pools have nowhere to fall
6. **Dynamic/Argo steering is paid** — Failover and Random are available on most plans; Geo, Dynamic, Proximity require the Load Balancing add-on
7. **CNAME setup (keep NS1) requires Business/Enterprise** — free/Pro requires full nameserver transfer

---

## Quick Reference

**Request flow (orange cloud on):**
```
User → Cloudflare POP → Worker (if route matches) → Load Balancer → Pool → Origin
```

**Pool failover chain:**
```
Mapped pool (unhealthy) → Default pool → next healthy in priority list → Fallback pool → 503
```

**When to use what:**
```
Failover      // active-passive, simple primary/secondary
Geo           // route users to nearest region
Random        // weighted canary deployments
Dynamic/Argo  // lowest latency, real-time (paid)
Workers       // header/cookie routing, A/B testing, async logic
```

---

### Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

```
I know Cloudflare CDN traffic management well. Quick context:
Cloudflare is DNS authority for your zone. Orange cloud proxies traffic
through Cloudflare's edge. Load Balancers sit above Pools (groups of
Origins with health Monitors). Traffic Steering policies on the LB
decide which Pool gets a request — Geo, Random, Failover, Dynamic.
Workers intercept before the LB for header/cookie/async routing logic.

I'm building: [describe your task here]
```
