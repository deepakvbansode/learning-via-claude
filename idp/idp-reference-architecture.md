# IDP Reference Architecture

## The Problem It Solves
Without an IDP, developers own platform concerns: writing K8s YAML, Terraform
modules, CI config, secrets wiring — repeated inconsistently across every
service. An IDP separates developer intent (what a service needs) from platform
implementation (how to provision it), letting developers stay in their domain
while the platform team owns the how.

---

## Core Concepts

### 1. IDP Layer Model

**Why:** Separates UI, orchestration, provisioning, and delivery into
independent layers so each evolves without breaking the others.

**Mental model:** Same as K8s — you declare desired state, the control
plane reconciles. IDP does this one level above K8s for your entire platform.

**Example:**

Developer Interface   → Backstage (portal) + Score (spec)
Platform Orchestrator → custom engine / Humanitec-style
Infrastructure Engine → Terraform
Delivery Engine       → ArgoCD
Runtime               → K8s



---

### 2. Backstage as Portal

**Why:** Developers need a single pane of glass — service catalog,
scaffolding, docs, status. Backstage is that UI layer, nothing more.

**Mental model:** Backstage is a React frontend backed by a plugin system.
It's a form + catalog, not an orchestrator. Like a kubectl UI — it fires
commands but doesn't implement control loops.

**Example:**
Developer fills "new service" form in Backstage
→ Scaffolder plugin creates git repo
→ Commits catalog-info.yaml + score.yaml to repo
→ Backstage discovers new service via catalog-info.yaml
→ Backstage's job is done. CI/orchestrator take over.



---

### 3. Score Workload Spec

**Why:** Eliminates per-environment config duplication. One spec
works for local (docker-compose), staging (K8s pod), prod (RDS).

**Mental model:** Score is a Go interface. The spec is the interface
definition. Each environment is a different implementation. The orchestrator
is the runtime that resolves the implementation.

**Example:**
```yaml
# score.yaml
apiVersion: score.dev/v1b1
metadata:
  name: my-service
containers:
  app:
    image: .
    variables:
      DB_URL: ${resources.db.connection_string}
resources:
  db:
    type: postgres   # WHAT, not HOW
```
### 4. Platform Orchestrator (RMCD)
Why: Something stateful must bridge Score specs to Terraform/K8s.
CI scripts are stateless and break at scale. ArgoCD deploys but doesn't
provision. Terraform provisions but doesn't know about workloads.

Mental model: A K8s controller, but for your entire platform.
Watches desired state (score.yaml), compares to actual state, reconciles.
Needs a state store to track resource allocations across workloads.

Example:


Read   → parse score.yaml + environment context (staging? prod?)
Match  → find resource driver: type:postgres → aws-rds Terraform module
Create → generate terraform.tfvars + K8s manifests
Deploy → trigger terraform apply + commit manifests to GitOps repo

### 5. The Wiring (Git as Event Bus)
Why: Every layer transition needs a trigger. Git push is the
universal event that connects all layers without tight coupling.

Mental model: Like a Unix pipe — each layer reads from stdin
(git/API), does its job, writes to stdout (git/API). No layer knows
about layers beyond its immediate neighbors.

Example:


git push
  → GitHub Actions: build image, call orchestrator API
  → Orchestrator: RMCD → terraform apply → capture outputs
                       → write K8s Secrets
                       → commit manifests to GitOps repo
  → ArgoCD: detects new manifests → syncs to K8s
  → Pod runs with injected secrets
Gotchas
Orchestrator needs a state store — tracking which workload owns which
resource instance (DB, S3 bucket) requires a DB. CI scripts can't do this.
Design the schema early.

Secrets never in manifests — Terraform outputs (DB URLs, S3 creds) go
into K8s Secrets, referenced by name in manifests. Never hardcoded. GitOps
repo must stay clean of plaintext secrets.

Bootstrapping problem — The orchestrator, ArgoCD, and the GitOps repo
must exist before the IDP works. Plan a "day 0" manual Terraform run to
stand up the platform itself.

GitOps repo structure is load-bearing — Define your envs/staging/,
envs/prod/ directory convention before writing the orchestrator. ArgoCD
Applications point at specific paths. Changing this later is painful.

Idempotency is non-negotiable — Running the orchestrator twice on the
same Score spec must produce identical output. Otherwise infra drifts
silently between deploys.

Quick Reference
IDP layers and their tools (your stack)


Portal         → Backstage
Spec           → score.yaml
Orchestrator   → custom (RMCD engine)
IaC            → Terraform
GitOps         → ArgoCD
Runtime        → K8s
RMCD pattern


Read    → parse Score + env context
Match   → find resource driver (Terraform module, K8s operator)
Create  → generate terraform.tfvars + K8s manifests + Secrets
Deploy  → terraform apply + commit to GitOps repo
GitOps repo layout (recommended)


gitops-repo/
  envs/
    staging/
      my-service/   ← orchestrator writes here
    prod/
      my-service/
Session Handoff Prompt
Copy this into a new Claude Code session when you're ready to build:

I know IDP reference architecture well. Quick context: An IDP separates developer intent (score.yaml) from platform implementation (orchestrator → Terraform + K8s manifests). Git is the event bus: CI triggers the orchestrator, which runs RMCD, commits to a GitOps repo, and ArgoCD syncs to K8s. Backstage is the portal layer only — scaffolding and catalog. I'm building: [describe your task here — e.g. "the custom platform orchestrator that reads score.yaml, runs Terraform modules, and commits K8s manifests to a GitOps repo"]

