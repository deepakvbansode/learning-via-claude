# Score Internals & Provisioners

## The Problem It Solves
`score.yaml` describes *what* a workload needs (postgres, dns) without specifying *how*
to provide it. Provisioners are the "how" — swappable per environment. Without this,
developers would hardcode infra details (RDS endpoints, hostnames) into app config,
coupling dev and prod.

---

## Core Concepts

### 1. The generate Pipeline

**Why:** Bridges developer intent (`score.yaml`) to deployable K8s manifests, with
persistent state so outputs stay stable across runs.

**Mental model:** Like `go build` but with a local cache (`.score-k8s/state.yaml`) that
remembers which provisioner handled each resource and what it produced.

**Steps:**
```
score-k8s generate
  1. READ   state.yaml         — existing workloads + resources from previous runs
  2. MERGE  score.yaml         — add/update workloads and resources
  3. MATCH  resource → provisioner — first match wins (lexicographic file order)
  4. EXECUTE provisioners      — runs provisioner logic, produces outputs + K8s manifests
  5. WRITE  state.yaml + manifests.yaml
```

**Key insight:** The matched provisioner is stored in `state.yaml`. On subsequent runs,
Score uses the stored provisioner — it does not re-match. Delete the resource entry from
`state.yaml` to force re-matching.

---

### 2. Provisioner Matching

**Why:** Different environments need different implementations of the same resource type.
Matching lets you swap implementations without touching `score.yaml`.

**Mental model:** Like Go interface dispatch — `score.yaml` declares the interface
(`type: postgres`), provisioners are concrete implementations, matching picks the struct.

**Match fields:**
```yaml
- uri: template://prod/postgres
  type: postgres      # required — must match resource type in score.yaml
  class: null         # null = match any class; "default" = match only class: default
  id: primary-db      # null = match any resource id; non-null = exact match only
```

**File loading order:** Lexicographic. Name custom files `10-` or `aa-` so they sort
before `zz-default.provisioners.yaml`.

```
.score-k8s/
  10-production.provisioners.yaml   ← loaded first, wins on conflict
  zz-default.provisioners.yaml      ← loaded last, fallback
```

**Override targeting:**

| Goal | Fields to set |
|------|---------------|
| All resources of a type | `type: postgres`, `id: null` |
| One specific resource | `type: postgres`, `id: primary-db` |
| One class only | `type: postgres`, `class: production` |

**Note:** The resource `id` is the key name in `score.yaml`, not a `params` field:
```yaml
resources:
  primary-db:     # ← this IS the resource id
    type: postgres
```

---

### 3. init → state → outputs → manifests Lifecycle

**Why:** Outputs must be stable across runs (env vars can't change each deploy).
The lifecycle separates ephemeral computation from persistent memory.

**Mental model:** Like a Go struct — `init` is the constructor (runs each time),
`state` is the struct's persistent fields (survives in `state.yaml`), `outputs` is
the public API, `manifests` is the side effect (K8s objects).

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌───────────┐
│  init   │────▶│  state  │────▶│ outputs │────▶│ manifests │
└─────────┘     └─────────┘     └─────────┘     └───────────┘
  fresh each      persisted       exposed as       K8s objects
  run, sets       in state.yaml   ${resources.x}   added to
  defaults        between runs    placeholders     manifests.yaml
```

**Example — stable hostname pattern:**
```yaml
init: |
  suffix: {{ randAlphaNum 8 | lower }}           # fresh each run

state: |
  hostname: {{ dig "hostname" (printf "dns%s.localhost" .Init.suffix) .State }}
  # ^^^ keep existing value; only use init's value on first run

outputs: |
  host: {{ .State.hostname }}                    # exposed as ${resources.dns.host}
```

**Key rule:** `init` is ephemeral — never rely on it for stable values. Use the
`dig "key" default .State` pattern in `state` to freeze values after first generation.

---

### 4. template vs cmd Provisioners

**Why:** Go templates can't make API calls or run complex logic. `cmd` provisioners
let you write provisioner logic in any language.

**Mental model:** `template` = synchronous handler in-process (everything in YAML);
`cmd` = separate process with JSON over stdin/stdout as the wire protocol.

**Template provisioner:**
```yaml
uri: template://my-provisioners/postgres
type: postgres
init: |
  {{ ... go template + sprig functions ... }}
state: |
  {{ ... }}
outputs: |
  {{ ... }}
manifests: |
  {{ ... }}
```

**Cmd provisioner:**
```yaml
uri: cmd://./.score-k8s/../provisioners/rds.sh#aws-rds
type: postgres
class: rds
```

**stdin/stdout contract:**
```json
// Score sends on stdin:
{
  "type": "postgres",
  "class": "rds",
  "id": "db",
  "guid": "abc-123",
  "state": {},
  "params": { "size": "db.t3.medium" }
}

// Script must return on stdout:
{
  "state": { "pipeline_run_id": "run-456" },
  "outputs": {
    "host": "myapp.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "name": "appdb",
    "username": "admin",
    "password": "..."
  },
  "manifests": []
}
```

---

### 5. Adding / Overriding Provisioners

**Why:** Default provisioners are dev stubs. Overrides let you inject production
implementations without modifying files you don't own.

**Mental model:** Like K8s admission webhooks over defaults — intercept specific
resource types, let everything else fall through.

**Three ways:**

1. Drop a file in `.score-k8s/` named to sort before `zz-`:
```
.score-k8s/10-prod.provisioners.yaml
```

2. Pull from remote at init time:
```bash
score-k8s init --provisioners https://example.com/my-provisioners.yaml
# also supports: git://, oci://, local paths
```

3. Minimal override example (fixed DNS hostname):
```yaml
# .score-k8s/10-prod.provisioners.yaml
- uri: template://prod-provisioners/dns
  type: dns
  outputs: |
    host: myapp.prod.example.com
```

**After adding:** Delete the resource entry from `state.yaml` to force re-matching.

---

## Real-World Pattern: Async Terraform Pipeline Provisioner

**Use case:** Provision RDS on AWS by triggering an existing Terraform CD pipeline.

**File layout:**
```
.score-k8s/
  10-rds.provisioners.yaml     ← provisioner declaration
provisioners/
  rds.sh                       ← script with the actual logic
```

**Provisioner YAML:**
```yaml
# .score-k8s/10-rds.provisioners.yaml
- uri: cmd://./.score-k8s/../provisioners/rds.sh#aws-rds
  type: postgres
  class: rds
```

**score.yaml resource:**
```yaml
resources:
  db:
    type: postgres
    class: rds          # targets the custom provisioner
```

**Async state machine across generate runs:**

```
Run 1: no pipeline_run_id in state
       → read params from stdin
       → update .tfvars
       → trigger CD pipeline via API
       → store run_id in state
       → return error "provisioning in progress"

Run 2: run_id in state, pipeline still running
       → check pipeline status via API
       → return error "still provisioning"

Run 3: run_id in state, pipeline complete
       → read RDS endpoint from Terraform outputs (SSM / TF output file)
       → return outputs (host, port, name, username, password)
       → store outputs in state for future runs

Run 4+: outputs already in state
       → return cached outputs immediately
```

**Three async strategy options:**

| Option | How it works | Best for |
|--------|-------------|----------|
| Wait | Trigger + block until pipeline completes | Simple setup, slow generate (10-20 min) |
| Checkpoint | Trigger, store run_id, fail; succeed on next run | Most real-world cases |
| Read-only | Pipeline runs out-of-band; script just reads outputs | Teams with existing infra automation |

---

## Gotchas

1. **Re-matching requires manual state edit**: After adding a new provisioner, delete
   the resource entry from `state.yaml` — Score uses the stored provisioner, not re-matches.

2. **`init` is not persistent**: If you need stable values (guids, hostnames), use the
   `dig "key" default .State` pattern in `state`, not in `init`.

3. **First match wins, silently**: Two provisioners matching the same resource — only
   the first runs. File naming (`10-` vs `zz-`) is your only ordering lever.

4. **cmd stdout must be valid JSON**: A crash or malformed output fails generate with
   a cryptic error. Always validate your script's output shape before wiring it in.

5. **Async pipelines need idempotency**: Always check `state.pipeline_run_id` before
   triggering — otherwise every `generate` run fires a new pipeline.

---

## Quick Reference

**Provisioner URI schemes:**
```
template://namespace/name       # Go template, all stages in YAML
cmd://./path/to/script#label    # external script, JSON stdin/stdout
cmd://bash#label                # run via bash
```

**Template context variables:**
```
.Type, .Class, .Id, .Uid, .Guid    # resource identity
.Init                               # output of init stage (ephemeral)
.State                              # persisted state from last run
.Shared                             # shared across all resources
.SourceWorkload                     # which workload declared this resource
.Params                             # resource params from score.yaml
```

**Stable value pattern:**
```yaml
state: |
  myValue: {{ dig "myValue" .Init.candidate .State }}
```
