# Fastly Traffic Steering

## The Problem Fastly Solves

Your origin servers sit in one region. Users are global. Every request travels far, slow, and expensive. Fastly puts a **global fleet of caching/routing nodes (POPs)** between users and your origin — each POP runs your config, caches responses, and only forwards to origin on cache misses. Beyond caching, Fastly lets you do **traffic steering at the edge**: route by path, headers, weight, or health — before traffic ever reaches your cluster.

---

## Core Concepts

### Service

**Why:** You need a single versioned unit of configuration — domains, routing rules, backends — that you can deploy and rollback atomically.

**Mental model:** Like a K8s `Deployment`. Each change creates a new version. Activating a version = deploying. Rolling back = activating a previous version. Every POP in the world runs the active version of your Service.

**Example:**
```
Service v12 (active)
  domains: api.example.com
  backends: stable, canary
  VCL: route /api/* to api_director
```

---

### Backends

**Why:** You need named upstream servers with connection settings and health checks — Fastly needs to know where to send traffic and whether those destinations are alive.

**Mental model:** Like K8s pod endpoints with a `livenessProbe`. Each backend has a host, port, SSL flag, and a probe that Fastly polls to determine health.

**Example:**
```vcl
backend stable {
  .host = "stable.example.com"; .port = "443"; .ssl = true;
  .probe = { .url = "/healthz"; .interval = 5s; .threshold = 2; .window = 5; }
}

backend canary {
  .host = "canary.example.com"; .port = "443"; .ssl = true;
  .probe = { .url = "/healthz"; .interval = 5s; .threshold = 2; .window = 5; }
}
```

---

### Directors

**Why:** A single backend is a single point of failure with no load distribution. Directors group backends and add a selection policy + automatic health-aware failover.

**Mental model:** Like a K8s `Service` in front of Pods. The Director picks which backend handles a request based on policy. If a backend fails its health probe, the Director skips it automatically. If **all** backends are unhealthy, Fastly routes to the "least broken" one rather than erroring.

**Policies:**

| Policy   | Use when                          |
|----------|-----------------------------------|
| `random` | Load balancing (weighted traffic) |
| `hash`   | Cache consistency (same URL → same backend) |
| `client` | Session affinity (same IP → same backend) |
| `chash`  | Consistent hash when backends change often |

**Example:**
```vcl
director api_director random {
  { .backend = stable; .weight = 80; }
  { .backend = canary; .weight = 20; }
}
```

> Always point `req.backend` at a Director, not a bare backend — Directors give you health-aware routing for free.

---

### VCL

**Why:** You need to hook into the request lifecycle to make routing decisions, manipulate headers, and control caching — declarative config isn't expressive enough.

**Mental model:** VCL is Fastly's DSL with lifecycle hooks (`vcl_recv`, `vcl_pass`, `vcl_deliver`). Think of `vcl_recv` as a middleware function that runs on every incoming request at the edge. You set `req.backend` here to pick the Director/backend for this request.

**Key variables:**
```vcl
req.url          // full URL including query string
req.url.path     // path only (use this for path matching)
req.http.X-Foo   // request header named X-Foo
req.backend      // set this to pick backend or director
```

**Gotcha:** `#FASTLY RECV` must stay in `vcl_recv` — Fastly injects its own required code at that comment. Remove it and things break silently.

**Example:**
```vcl
sub vcl_recv {
  #FASTLY RECV   // ← never remove this

  if (req.url.path ~ "^/api/") {
    set req.backend = api_director;
  }
}
```

---

### Conditions & Traffic Steering

**Why:** Different paths, headers, or client properties need different backends — you need expressive if/else routing logic at the edge.

**Mental model:** Like K8s Ingress rules — `if/elsif` chains in `vcl_recv`, **first match wins**. Put specific rules before general ones. Combine path routing + weighted Directors for canary deployments.

**Example — path routing + canary:**
```vcl
sub vcl_recv {
  #FASTLY RECV

  if (req.url.path ~ "^/api/") {
    set req.backend = api_director;      // 80/20 split, health-aware
  } elsif (req.url.path ~ "^/static/") {
    set req.backend = static_director;
  } else {
    set req.backend = default_director;
  }
}
```

---

## POPs (Points of Presence)

**Why:** Latency drops when the edge node is physically close to the user.

**Mental model:** A global fleet of nginx+cache nodes. Each POP independently caches responses and runs your Service config. Your origin only sees requests that missed the cache.

---

## Gotchas

1. `#FASTLY RECV` must stay in `vcl_recv` — Fastly injects its own code there
2. Use `req.url.path` for path matching, not `req.url` (which includes query string)
3. If all backends in a Director are unhealthy, Fastly routes anyway to the least broken one
4. Always use Directors over bare backends in production — health-aware routing is automatic
5. VCL conditions are first-match-wins — put specific rules before general ones
