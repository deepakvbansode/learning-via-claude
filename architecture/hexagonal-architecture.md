​# Hexagonal Architecture — Deep Reference

## The Problem It Solves

Business logic gets tangled with infrastructure (databases, APIs, frameworks, workflow engines). When your service layer imports `database/sql` or your domain struct has `gorm` tags, you can't test business rules without spinning up infrastructure. You can't swap Postgres for DynamoDB without rewriting business logic. You can't reuse domain rules across different entry points (HTTP, gRPC, CLI, Temporal workers).

Hexagonal architecture isolates your domain so it depends on nothing external. Everything external depends on it through well-defined interfaces (ports). Dependencies always point **inward** — outer layers know about inner layers, inner layers know nothing about the outside.

```
┌─────────────────────────────────────────────┐
│              Adapters (outer)               │
│  HTTP Handler, gRPC, Temporal Worker,       │
│  Postgres Repo, Fastly Client, Email Sender │
├─────────────────────────────────────────────┤
│           Use Case Layer (middle)           │
│  CreateService, AddBackend, DeleteService   │
├─────────────────────────────────────────────┤
│           Domain Layer (inner)              │
│  Entities, Value Objects, Aggregate Roots,  │
│  Domain Services, Business Rules            │
└─────────────────────────────────────────────┘

Dependencies flow INWARD only:
  Adapter → Use Case → Domain
  Never: Domain → Use Case or Domain → Adapter
```

---

## Core Concepts

---

### 1. Dependency Inversion Principle

**Why it exists:** Without it, your domain layer imports infrastructure packages (`database/sql`, `temporal-sdk`, `fastly-go`). This creates coupling — you can't test domain logic without real infrastructure, and swapping infrastructure means rewriting business code.

**The rule:** The domain defines interfaces (ports) that say "I need something that can do X." Infrastructure implements those interfaces. The domain never imports infrastructure — it's the other way around. The `main` package (or a wire-up layer) connects them.

**Analogy:** A Kubernetes Pod spec declares it needs a volume. It doesn't care if that volume is EBS, NFS, or hostPath. The Pod defines the contract, the infrastructure fulfills it.

```go
// Domain defines the port (interface) — lives in domain/ports package
type ServiceRepository interface {
    FindByName(ctx context.Context, name ServiceName) (*Service, error)
    Save(ctx context.Context, service *Service) error
}

// Adapter implements it — lives in adapter/postgres package
// This package imports domain. Domain never imports this package.
type PostgresServiceRepo struct {
    db *sql.DB
}

func (r *PostgresServiceRepo) FindByName(ctx context.Context, name ServiceName) (*Service, error) {
    // SQL query, scan into domain struct
}

// main.go wires them together
func main() {
    db := connectPostgres()
    repo := postgres.NewServiceRepo(db)          // adapter
    useCase := usecase.NewCreateService(repo)     // inject via port
    handler := http.NewServiceHandler(useCase)    // adapter
}
```

**What this gives you:**
- Unit test domain logic with mock repositories — no database needed
- Swap Postgres for DynamoDB by writing a new adapter — domain code unchanged
- Same domain logic works behind HTTP, gRPC, CLI, or Temporal workers

---

### 2. Domain Entities vs Value Objects

**Why it matters:** Not everything in your domain model is the same kind of thing. Some objects have identity and a lifecycle (created, modified, deleted). Others are just a bag of attributes with no identity. Mixing them up leads to confusion about where IDs, validation, and equality checks belong.

#### Entities

An entity has a **unique identity** that distinguishes it from other entities, even if all other attributes are identical. It has a lifecycle — it is created, changes over time, and can be deleted. You track it by its identity.

- Two `Backend` entities with the same name, weight, and health check in different directors are **different backends**
- A `Service` is still the same service even after you add or remove directors

#### Value Objects

A value object has **no identity**. It is defined entirely by its attributes. Two value objects with the same attributes are interchangeable. They are immutable — you don't modify a value object, you replace it with a new one.

- `HealthCheck{Path: "/health", Interval: 30s, Threshold: 3}` has no ID. Any health check with those same values is the same thing.
- `Money{Amount: 100, Currency: "USD"}` — no identity, purely defined by its values.
- `Address{Street: "123 Main", City: "NYC"}` — you compare by values, not by some ID.

**Analogy:** A Kubernetes Pod is an entity — it has a UID, and two Pods with identical specs are still different Pods. A `ResourceRequirements` (`cpu: 500m, memory: 128Mi`) is a value object — no ID, two identical ones are the same thing.

#### How to decide

Ask: **"Can I have two of these with identical attributes that are still different things?"**
- Yes → Entity (needs identity)
- No → Value Object (defined by attributes)

```go
// Entity — has identity, has lifecycle
type Service struct {
    Name          ServiceName     // identity
    PortfolioName PortfolioName   // reference
    ProviderName  ProviderName
    Directors     []Director
}

type Director struct {
    Name     DirectorName        // identity within the aggregate
    Backends []Backend
}

type Backend struct {
    Name        BackendName      // identity within the aggregate
    Weight      int
    HealthCheck HealthCheck      // value object — no identity
}

// Value Object — no identity, defined by attributes, immutable
type HealthCheck struct {
    Path      string
    Interval  time.Duration
    Threshold int
}

type DomainName string  // value object — "ab.com" is "ab.com"
```

#### Real-world example: Health checks across providers

In Fastly, health checks are directly attached to backends. In Cloudflare, they're separate entities called "monitors" that you associate with pools. But in **your** domain model, you decided a health check is a value object — just configuration attached to a backend. Your domain model doesn't have to mirror the infrastructure's model. The adapters handle the translation.

---

### 3. Entity Identity & IDs

**Why this is confusing:** Engineers instinctively add `ID uint` or `ID uuid.UUID` to every struct because that's what the database needs. But domain identity and database identity are different concerns.

#### Domain identity vs database identity

- **Domain identity** = how your business identifies something. A portfolio is identified by its domain name (`ab.com`). An order by its order number (`ORD-2024-0042`). This has business meaning.
- **Database identity** = a primary key (auto-increment `int`, `uuid`). Your Postgres table needs it. Your domain doesn't care about it.

**Analogy:** A Docker container has a container ID (`sha256:abc123`) — infrastructure assigns this. It also has a name (`my-redis`) — you chose this, it's meaningful. Both identify the container, but they serve different purposes.

#### Rules for domain IDs

**Rule 1: The domain decides what identifies an entity, not the database.**

```go
// BAD — domain coupled to database identity
type Portfolio struct {
    ID        uint      // who decides this? the database
    Domain    string
    Providers []Provider
}

// GOOD — domain owns its identity
type Portfolio struct {
    Name       PortfolioName   // this IS the identity — business-meaningful
    DomainName DomainName
    Providers  []Provider
}
```

**Rule 2: Wrap IDs in domain types.** Don't use raw `string` or `int`. This prevents accidentally passing a `CustomerID` where an `OrderID` is expected. The Go compiler catches it.

```go
type ServiceName string
type PortfolioName string
type DirectorName string
type BackendName string

// Compiler catches this mistake:
func FindService(name ServiceName) { ... }
FindService(PortfolioName("oops"))  // compile error
```

**Rule 3: Who generates the ID?**
- Business-meaningful (domain name, user-chosen name) → the user/domain generates it
- Synthetic (UUID) → generate it in the use case layer **before** calling the repository, not inside the database
- The domain should know the entity's identity at creation time

#### Can identity change?

If the user-friendly name IS the identity and the user can rename it, you have a problem — everything referencing that entity by name now has a stale reference. Two options:

1. **Name is immutable** — users pick it once, never change it. Name is the identity. Simple.
2. **Name can change** — introduce a stable synthetic ID (UUID) as the real identity. The name becomes just a display attribute. The domain still owns and generates the UUID.

This is a domain decision, not a technical one.

---

### 4. Entity Relationships & Aggregates

**Why aggregates exist:** Without clear boundaries, you either load the entire database for every operation, or make partial updates that break business rules. Aggregates define what gets loaded, validated, and saved as a unit.

#### What is an aggregate?

A cluster of entities and value objects treated as a **single unit** for data changes. The top-level entity is the **aggregate root** — the only entity that outside code can reference directly.

**Analogy:** A Kubernetes Deployment owns ReplicaSets, which own Pods. You interact with the Deployment — you don't directly modify a ReplicaSet. When you delete the Deployment, everything underneath goes with it. The Deployment is the aggregate root.

#### Aggregate rules

1. **Outside code only references the aggregate root** — never reaches inside to grab a child directly
2. **The root enforces all invariants** (business rules) for the whole cluster
3. **Load and save the entire aggregate as a unit** — your repository loads and saves the whole thing, not pieces
4. **Cross-aggregate references use IDs, not embedding** — aggregates point to each other by identity

#### You do NOT need separate structs for entity and aggregate root

The aggregate root IS just the top-level entity. It's a role, not a type. There's no `ServiceAggregateRoot` wrapper around `Service`. The `Service` struct is both an entity and the aggregate root.

```go
// Service IS the aggregate root. No separate wrapper needed.
type Service struct {
    Name          ServiceName       // identity
    PortfolioName PortfolioName     // reference to another aggregate (by ID)
    ProviderName  ProviderName
    Directors     []Director        // owned entity
}

type Director struct {
    Name     DirectorName
    Backends []Backend             // owned entity
}

type Backend struct {
    Name        BackendName
    Weight      int
    HealthCheck HealthCheck        // owned value object
}
```

#### The root is the gatekeeper

All mutations go through the aggregate root's methods. Nobody directly modifies child entities:

```go
// WRONG — bypasses business rules
service.Directors[0].Backends = append(service.Directors[0].Backends, newBackend)

// RIGHT — root enforces rules
err := service.AddBackendToDirector(directorName, newBackend)
```

The root's methods enforce invariants:

```go
func (s *Service) AddBackendToDirector(dirName DirectorName, backend Backend) error {
    dir := s.findDirector(dirName)
    if dir == nil {
        return ErrDirectorNotFound
    }
    if dir.hasBackend(backend.Name) {
        return ErrDuplicateBackend
    }
    // validate weight rules, etc.
    dir.Backends = append(dir.Backends, backend)
    return nil
}

func (s *Service) RemoveBackend(dirName DirectorName, beName BackendName) error {
    dir := s.findDirector(dirName)
    if len(dir.Backends) <= 1 {
        return ErrDirectorMustHaveBackend  // invariant enforced
    }
    dir.removeBackend(beName)
    return nil
}
```

#### How to decide aggregate boundaries

Ask: **"What must be consistent in a single transaction?"**

- If changing a backend's health check must be consistent with the director's state → same aggregate
- If a portfolio can exist without its services being fully loaded → separate aggregates
- If different users/processes modify different parts independently → split them

**Practical rule:** If the data is always modified together, it's one aggregate. If different parts are modified independently, split them.

#### Cross-aggregate references

When aggregates are separate, they reference each other by ID, not by embedding:

```go
// Portfolio aggregate — does NOT embed services
type Portfolio struct {
    Name       PortfolioName
    DomainName DomainName
}

// Service aggregate — references portfolio by ID
type Service struct {
    Name          ServiceName
    PortfolioName PortfolioName   // reference, not embedded Portfolio
    ProviderName  ProviderName
    Directors     []Director      // owned, within this aggregate
}
```

#### "Isn't loading the whole aggregate expensive?"

For most aggregates, no. A service with 10 directors and 20 backends each is maybe 200 objects — a few kilobytes. Trivially small.

**If your aggregate is too expensive to load fully, your aggregate boundary is too large.** Split it into smaller aggregates.

#### Repository save strategy

**Start with the replace strategy (simple):**

```go
func (r *ServiceRepo) Save(service *Service) error {
    // single transaction:
    // DELETE backends WHERE director_id IN (SELECT id FROM directors WHERE service_name = ?)
    // DELETE directors WHERE service_name = ?
    // INSERT directors ...
    // INSERT backends ...
}
```

No diffing logic. Delete children, re-insert. Works perfectly for small aggregates. Only add dirty-tracking or event-based diffing when you **measure** a real performance problem.

---

### 5. Use Case Layer (Application Services)

**Why it exists:** Orchestration logic — load aggregate, validate, save, trigger side effects — doesn't belong in the domain (domain doesn't know about repositories) or in adapters (HTTP handlers shouldn't contain business flow). The use case layer is the thin coordinator.

#### One use case = one user intention

Each use case represents a single thing a user wants to do. Not a CRUD service with `Create`, `Update`, `Delete` in one struct:

```go
// Each use case is its own struct with one Execute method
type CreateService struct {
    portfolioRepo PortfolioRepository
    serviceRepo   ServiceRepository
    workflowStart WorkflowStarter
}

type AddBackendToDirector struct {
    serviceRepo ServiceRepository
}

type DeleteService struct {
    serviceRepo ServiceRepository
}
```

**Analogy:** Kubernetes has one controller per concern — Deployment controller, ReplicaSet controller, not one giant controller doing everything.

#### The flow: load → call domain → save

```go
func (uc *AddBackendToDirector) Execute(cmd AddBackendCmd) error {
    // 1. Load aggregate from repository
    service, err := uc.serviceRepo.FindByName(cmd.ServiceName)
    if err != nil {
        return err
    }

    // 2. Call domain method — aggregate enforces rules
    err = service.AddBackendToDirector(cmd.DirectorName, cmd.Backend)
    if err != nil {
        return err  // domain rejected it (duplicate, invalid weight, etc.)
    }

    // 3. Save whole aggregate
    return uc.serviceRepo.Save(service)
}
```

#### What the use case does NOT do

- **Does not contain business rules** — if you see `if weight > 100` in a use case, move it to the entity
- **Does not know about HTTP, gRPC, or Temporal** — it takes a command struct, not `*http.Request`
- **Does not query the repository for validation** — the aggregate is loaded, it has all the data it needs to validate itself

```go
// BAD — business rule leaked into use case
func (uc *AddBackend) Execute(cmd AddBackendCmd) error {
    existing, _ := uc.serviceRepo.FindBackendByName(cmd.BackendName)
    if existing != nil {
        return ErrDuplicateBackend  // this check belongs in the domain
    }
    // ...
}

// GOOD — domain handles it
func (uc *AddBackend) Execute(cmd AddBackendCmd) error {
    service, _ := uc.serviceRepo.FindByName(cmd.ServiceName)
    err := service.AddBackendToDirector(cmd.DirectorName, cmd.Backend)
    // AddBackendToDirector checks for duplicates internally
    if err != nil {
        return err
    }
    return uc.serviceRepo.Save(service)
}
```

#### Layer responsibilities

| Layer | Does | Doesn't |
|-------|------|---------|
| **Domain** | Enforces business rules, holds state, validates invariants | Call repositories, know about persistence or infrastructure |
| **Use Case** | Orchestrates flow, calls ports, coordinates | Contain business rules, know about HTTP/gRPC/Temporal |
| **Adapter** | Translates external formats (HTTP request → command, DB row → entity) | Contain flow logic or business rules |

---

### 6. Domain Rules: Where Logic Lives

**Why this matters:** This is the #1 source of messy hexagonal codebases. Engineers know "business logic goes in the domain" but struggle with which logic goes where. You end up with fat use cases doing validation that belongs in entities, or entities doing orchestration that belongs in use cases.

#### Three categories of rules

**Category 1: Invariant rules → Entity / Aggregate methods**

Rules that must **always** be true for the aggregate to be valid. Enforced inside the aggregate, on every mutation, no exceptions.

- "Backend weight must be 0-100"
- "Director must have at least one backend"
- "No duplicate backend names within a director"

```go
func (s *Service) AddBackendToDirector(dirName DirectorName, b Backend) error {
    if b.Weight < 0 || b.Weight > 100 {
        return ErrInvalidWeight  // invariant
    }
    dir := s.findDirector(dirName)
    if dir.hasBackend(b.Name) {
        return ErrDuplicateBackend  // invariant
    }
    dir.Backends = append(dir.Backends, b)
    return nil
}
```

**Category 2: Cross-aggregate rules → Domain Services**

Rules that need information from **multiple aggregates** that don't belong to each other. These are **rare**.

- "No more than 50 services across ALL portfolios for a given provider" — no single portfolio knows about other portfolios
- "A backend IP cannot be used in two different portfolios" — a service aggregate doesn't know about other portfolios' services

```go
// Domain service — pure logic, no I/O, no repositories
type GlobalProviderPolicy struct{}

func (p GlobalProviderPolicy) CanAddService(allPortfolios []Portfolio, provider ProviderName) error {
    count := 0
    for _, portfolio := range allPortfolios {
        if portfolio.HasProviderService(provider) {
            count++
        }
    }
    if count >= 50 {
        return ErrProviderLimitReached
    }
    return nil
}
```

**Important:** If one aggregate can answer the question alone, it's a method on that aggregate — not a domain service. "A portfolio can only have one service per provider type" → the Portfolio aggregate knows its own provider list, so `portfolio.CanAddService(providerName)` is sufficient. No domain service needed.

**Simple rule: if one aggregate can answer the question alone, it's a method on that aggregate. If it can't, it's a domain service. Don't create domain services preemptively.**

**Category 3: Orchestration → Use Cases**

"Load this, call that, save this, notify that." No business decisions — just workflow coordination.

- "After creating a service, sync the configuration to Fastly"
- "Load portfolio, check if service can be added, create service, save, start workflow"

#### The decision test

Ask: **"If I moved this logic to a different use case, would the system break?"**

- **Yes** → It's a domain invariant. It belongs in the entity/aggregate. It must be enforced regardless of which use case triggers it.
- **No** → It's orchestration. It belongs in the use case.

#### Example: Delete service

Business rule: "A service can only be deleted if all its backends are drained (weight = 0)."

```go
// Domain — validates the rule (does NOT delete itself from DB)
func (s *Service) CanDelete() error {
    for _, d := range s.Directors {
        for _, b := range d.Backends {
            if b.Weight > 0 {
                return ErrBackendsNotDrained
            }
        }
    }
    return nil
}

// Use case — orchestrates
func (uc *DeleteService) Execute(name ServiceName) error {
    service, err := uc.serviceRepo.FindByName(name)
    if err != nil {
        return err
    }
    if err := service.CanDelete(); err != nil {
        return err  // domain says no
    }
    return uc.serviceRepo.Delete(name)
}
```

The domain doesn't delete itself from the database — it doesn't know databases exist. It only answers "am I in a state where deletion is allowed?"

---

### 7. Fitting Temporal Workflows into Hexagonal Architecture

**Why this matters:** Temporal is a workflow orchestration engine (long-running, durable processes with retries, timeouts, sagas). If your domain imports the Temporal SDK, you've coupled your business logic to a workflow engine.

#### Where Temporal sits

Temporal occupies **two adapter roles**:

1. **Temporal Worker** = **inbound adapter** (like an HTTP server). It listens for work and triggers your code. Just like an HTTP handler calls a use case, a Temporal activity calls a use case.

2. **Temporal Client** = **outbound adapter** (like a database client). When a use case needs to start a long-running process, it calls a port interface. The Temporal client implements that port.

```
┌──────────────────────────────────────────────────┐
│                  Adapters (outer)                 │
│                                                  │
│  Inbound:                    Outbound:           │
│  ┌──────────────────┐        ┌────────────────┐  │
│  │  HTTP Handler    │        │  Postgres Repo │  │
│  │  gRPC Server     │        │  Fastly Client │  │
│  │  Temporal Worker  │◄──┐   │  Temporal Client│  │
│  └──────┬───────────┘   │   └───────▲────────┘  │
│         │               │           │            │
├─────────┼───────────────┼───────────┼────────────┤
│         ▼               │           │            │
│  ┌──────────────────────┼───────────┼─────────┐  │
│  │   Use Case Layer     │           │         │  │
│  │                      │           │         │  │
│  │  CreateService ──────┼───────────┘         │  │
│  │  SyncToProvider ◄────┘  (called by         │  │
│  │  TakeSnapshot          activity)           │  │
│  └──────────────┬─────────────────────────────┘  │
│                 │                                 │
├─────────────────┼─────────────────────────────────┤
│                 ▼                                 │
│  ┌────────────────────────────────────────────┐   │
│  │         Domain Layer                       │   │
│  │  Service, Director, Backend, HealthCheck   │   │
│  │  Business rules, invariants                │   │
│  │  Knows NOTHING about Temporal              │   │
│  └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

#### The port for starting workflows

```go
// Port — defined in use case or domain layer
type WorkflowStarter interface {
    StartServiceSync(ctx context.Context, name ServiceName) (WorkflowID, error)
}

// Adapter — implements the port using Temporal
type TemporalWorkflowStarter struct {
    client client.Client
}

func (t *TemporalWorkflowStarter) StartServiceSync(ctx context.Context, name ServiceName) (WorkflowID, error) {
    options := client.StartWorkflowOptions{
        ID:        "sync-" + string(name),
        TaskQueue: "service-sync",
    }
    run, err := t.client.ExecuteWorkflow(ctx, options, SyncServiceWorkflow, name)
    if err != nil {
        return "", err
    }
    return WorkflowID(run.GetID()), nil
}
```

#### Workflows are orchestration adapters

The workflow itself lives in the adapter layer. It knows the order of steps, retry policies, and compensation logic:

```go
// Workflow — adapter layer, knows about Temporal SDK
func SyncServiceWorkflow(ctx workflow.Context, serviceName ServiceName) error {
    // Step 1: Take snapshot
    err := workflow.ExecuteActivity(ctx, TakeSnapshotActivity, serviceName).Get(ctx, nil)
    if err != nil {
        return err
    }

    // Step 2: Sync to provider
    err = workflow.ExecuteActivity(ctx, SyncToProviderActivity, serviceName).Get(ctx, nil)
    if err != nil {
        // Compensation: rollback snapshot
        _ = workflow.ExecuteActivity(ctx, RollbackSnapshotActivity, serviceName).Get(ctx, nil)
        return err
    }

    // Step 3: Verify sync
    return workflow.ExecuteActivity(ctx, VerifySyncActivity, serviceName).Get(ctx, nil)
}
```

#### Activities call use cases

Activities are thin wrappers — like HTTP handlers, they're just entry points:

```go
// Activity — adapter layer, calls use case
func TakeSnapshotActivity(ctx context.Context, serviceName ServiceName) error {
    // 'takeSnapshot' is a use case injected into the worker
    return takeSnapshot.Execute(ctx, serviceName)
}

func SyncToProviderActivity(ctx context.Context, serviceName ServiceName) error {
    return syncToProvider.Execute(ctx, serviceName)
}
```

#### The complete flow for "create service and sync to Fastly"

1. HTTP handler receives "create service" request → calls `CreateService` use case
2. `CreateService` use case loads Portfolio aggregate, calls `portfolio.CanAddService(providerName)` to validate
3. Use case creates the Service domain object, saves to DB via `ServiceRepository` port
4. Use case calls `WorkflowStarter.StartServiceSync()` port to trigger async sync
5. Temporal client adapter starts the workflow
6. Temporal worker (inbound adapter) picks up the workflow, executes activities in order
7. Each activity calls a use case (e.g., `TakeSnapshot`, `SyncToProvider`)
8. Use cases load domain aggregates, call domain methods, save via repositories
9. Domain layer knows nothing about Temporal at any point

#### Saga / compensation logic

If a multi-step process fails and you need to undo previous steps (e.g., "service synced to Fastly but Cloudflare failed, roll back Fastly"):

- **When** to compensate → the Temporal workflow decides (orchestration order)
- **How** to compensate → use case / domain decides (business logic)

The rollback itself (`UnsyncFromFastly`) is a use case, called via an activity. The workflow just decides the order.

---

## Gotchas

1. **Don't create separate structs for entity and aggregate root.** The aggregate root IS the top-level entity. It's a role, not a type.

2. **Don't validate in the use case what the domain can validate itself.** If the aggregate is loaded in memory, it has all the data to enforce its own rules. The use case just calls the domain method and handles the error.

3. **Start with the replace strategy for saving aggregates** (delete children + re-insert in a transaction). Don't build diffing or dirty-tracking until you measure a real performance problem.

4. **If your aggregate is too expensive to load, your boundary is too large.** Split it into smaller aggregates that reference each other by ID.

5. **Domain services are rare.** Only use them when a rule truly spans multiple aggregates and no single aggregate has enough information. If one aggregate can answer the question, it's a method on that aggregate.

6. **Don't mirror infrastructure models in your domain.** Fastly calls it a "service," Cloudflare calls it a "zone," your domain calls it what makes sense for your business. Adapters translate.

7. **The domain never calls repositories.** If you find yourself injecting a repository into a domain entity, that logic belongs in the use case layer.

8. **Cross-aggregate coordination uses domain events, not direct calls.** "Portfolio deleted → delete all its services" is handled by the use case or an event, not by the Portfolio aggregate reaching into the Service aggregate.

---

## Quick Reference

### Layer responsibilities

```
Domain Layer:
  - Entities, value objects, aggregate roots
  - Business invariants (validation, rules)
  - Domain services (cross-aggregate rules, rare)
  - Port interfaces (what the domain needs from outside)

Use Case Layer:
  - One struct per user intention
  - Orchestration: load → call domain → save → side effects
  - No business rules, no infrastructure knowledge

Adapter Layer (Inbound):
  - HTTP handlers, gRPC servers, Temporal workers, CLI
  - Translate external input → use case command
  - Call use case Execute method

Adapter Layer (Outbound):
  - Postgres repo, Redis cache, Fastly client, Temporal client, email sender
  - Implement port interfaces defined by domain/use case
  - Translate domain objects → infrastructure format
```

### Aggregate rules

```
Load as a unit       → repository loads entire aggregate (root + children)
Save as a unit       → repository saves entire aggregate
Reference by ID      → cross-aggregate references use IDs, not embedding
Root is gatekeeper   → all mutations go through root's methods
```

### Where does the rule live?

| Rule type | Location | Test |
|-----------|----------|------|
| Must always be true | Entity / aggregate method | Would breaking it corrupt data? |
| Spans multiple aggregates | Domain service | Does one aggregate lack the info? |
| Orchestration / workflow | Use case | Is it about order of operations? |
| Infrastructure-specific | Adapter | Is it about how, not what? |

### Entity vs Value Object decision

| Question | Entity | Value Object |
|----------|--------|-------------|
| Has unique identity? | Yes | No |
| Two identical instances are different? | Yes | No (interchangeable) |
| Has a lifecycle (create/modify/delete)? | Yes | No (immutable, replaced) |
| Tracked over time? | Yes | No |

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

```
I know hexagonal architecture well. Quick context:

Domain layer has entities, value objects, and aggregate roots that enforce all
business invariants through methods. Use cases orchestrate — load aggregate,
call domain method, save. Adapters (HTTP, Temporal, Postgres) sit on the outside
and depend inward through ports (interfaces). Temporal workers are inbound
adapters (like HTTP handlers), Temporal client is an outbound adapter behind a
port interface. Domain knows nothing about infrastructure.

My aggregates:
- Portfolio: identified by PortfolioName, owns portfolio metadata
- Service: identified by ServiceName, owns Directors → Backends → HealthCheck

I'm building: [describe your task here]
```
