# Temporal

## The Problem It Solves

Multi-step operations (like CDN config changes) that span external API calls can fail mid-way, leaving your system in an inconsistent state. Temporal makes your workflow functions **durable** — they survive crashes and resume exactly where they left off by replaying a recorded event history.

---

## Core Concepts

### Workflow & Determinism

**Why:** Temporal replays your workflow function from line 1 on every restart, using recorded results to fast-forward through completed steps. If your code isn't deterministic, replay produces different decisions than the original run and Temporal panics.

**Mental model:** A Go function whose call stack is persisted to PostgreSQL. On crash, the stack is reloaded from the log and execution continues.

**Example:**
```go
// Use workflow.Now(ctx) instead of time.Now()
// Use workflow.SideEffect(ctx, fn) for UUIDs or random values
func MyWorkflow(ctx workflow.Context) error {
    now := workflow.Now(ctx)  // deterministic on replay
    return nil
}
```

---

### Activities

**Why:** Workflows can't do I/O (it's non-deterministic). Activities are regular Go functions with no constraints — all real work (API calls, DB writes, Vault reads) lives here.

**Mental model:** Worker goroutines dispatched by a supervisor goroutine. The workflow orchestrates; activities execute.

**Example:**
```go
// Activity — plain context.Context, real I/O
func CloneDraftVersion(ctx context.Context, serviceID string) (int, error) {
    return fastlyClient.CloneVersion(serviceID)
}

// Workflow — calls activities, never does I/O directly
err := workflow.ExecuteActivity(ctx, CloneDraftVersion, serviceID).Get(ctx, &version)
```

---

### Workers & Task Queues

**Why:** Temporal server doesn't execute your code — it stores state and dispatches tasks onto named queues. Workers are your Go processes that poll a queue and execute both workflows and activities.

**Mental model:** Kubernetes Job queue — scheduler dispatches, worker pods execute. Multiple workers on the same queue for parallelism and fault tolerance.

**Example:**
```go
// Worker — registers and polls
w := worker.New(c, "stellar-cdn-queue", worker.Options{})
w.RegisterWorkflow(UpdateTrafficWeightWorkflow)
w.RegisterActivity(CloneDraftVersion)
w.Run(worker.InterruptCh())

// API server — starts workflows on the same queue name
opts := client.StartWorkflowOptions{TaskQueue: "stellar-cdn-queue"}
run, _ := h.temporal.ExecuteWorkflow(ctx, opts, UpdateTrafficWeightWorkflow, input)
```

---

### Signals

**Why:** Some workflows need external input mid-execution (e.g., operator approval before activating a version). Signals let you send data into a running workflow without restarting it.

**Mental model:** A Go channel that crosses process boundaries. The workflow blocks on `ch.Receive()`, the API server sends via workflow ID.

**Example:**
```go
// Workflow — blocks waiting for signal
ch := workflow.GetSignalChannel(ctx, "approval")
ch.Receive(ctx, &approved)

// API server — sends signal
h.temporal.SignalWorkflow(ctx, workflowID, "", "approval", true)
```

---

### Queries

**Why:** Read workflow state without affecting execution. Your UI can poll "what step is this workflow on?" directly from the workflow's internal variables.

**Mental model:** A read-only method call on a struct living inside another process. Works on running and completed workflows.

**Example:**
```go
// Workflow — registers query handler
workflow.SetQueryHandler(ctx, "get-status", func() (string, error) {
    return status, nil  // read-only, no side effects
})

// API server — queries
val, _ := h.temporal.QueryWorkflow(ctx, workflowID, "", "get-status")
```

---

### Retry Policies & Error Handling

**Why:** External APIs fail transiently. Instead of writing retry loops in every activity, declare a retry policy and let Temporal handle backoff and attempts.

**Mental model:** Like Kubernetes pod `restartPolicy` — declare the behavior, the system executes it.

**Example:**
```go
// Workflow — sets retry policy for activities
ao := workflow.ActivityOptions{
    StartToCloseTimeout: 30 * time.Second,
    RetryPolicy: &temporal.RetryPolicy{
        MaximumAttempts:    5,
        InitialInterval:    time.Second,
        BackoffCoefficient: 2.0,
    },
}

// Activity — returns non-retryable for permanent failures
if isNotFound(err) {
    return temporal.NewNonRetryableApplicationError("not found", "NOT_FOUND", err)
}
return err  // transient — Temporal retries
```

---

## Gotchas

1. Task queue names are plain strings — typo between API and worker means workflows hang forever with no error
2. Forgetting to register an activity on the worker causes silent hangs, not errors
3. Default retry is unlimited — always set `MaximumAttempts` explicitly for provider API calls
4. `workflow.Context` vs `context.Context` — workflows get `workflow.Context`, activities get plain `context.Context`. Don't mix them
5. Map iteration in workflows — Go maps iterate non-deterministically. Use sorted keys or slices in workflow code
6. Signal name mismatch — signal is buffered silently, workflow hangs forever waiting

---

## Quick Reference

**Key types:**

| Type | Used in |
|------|---------|
| `workflow.Context` | Workflow functions |
| `context.Context` | Activity functions |
| `workflow.Future` | Return of `ExecuteActivity`, call `.Get()` to block |
| `temporal.RetryPolicy` | Retry config for activities |

**Common patterns:**
```go
workflow.ExecuteActivity(ctx, fn, args...).Get(ctx, &result)   // call activity and wait
workflow.GetSignalChannel(ctx, name).Receive(ctx, &val)        // wait for signal
workflow.SetQueryHandler(ctx, name, handlerFn)                 // expose state to queries
workflow.Now(ctx)                                              // deterministic time
workflow.SideEffect(ctx, fn)                                   // run non-deterministic code once
```

**Error handling:**
```go
temporal.NewNonRetryableApplicationError(msg, typ, err)   // permanent failure, skip retries
return err                                                // transient, Temporal retries per policy
```

---

## Session Handoff Prompt

Copy this into a new Claude Code session when you're ready to build:

> I know Temporal well. Quick context: Temporal is a durable workflow engine — workflow functions are deterministic orchestrators that survive crashes via event history replay. Activities do all real I/O. Workers poll task queues and execute both. Signals push data into running workflows, Queries read state out. Retry policies handle transient failures automatically. I'm building: [describe your task here]
