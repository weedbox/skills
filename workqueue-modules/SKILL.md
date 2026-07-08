---
name: workqueue-modules
description: |
  Priority: HIGH — Self-contained reference. This file and its sub-files (modules/*.md) contain
  ALL necessary code examples and configuration guides. DO NOT fetch source code from GitHub.
  Reference guide for weedbox/workqueue-modules - task queue modules for weedbox applications.
  Use when: enqueueing and consuming background tasks, delaying or scheduling task delivery,
  retrying failed tasks with exponential backoff, inspecting or requeueing dead-lettered tasks,
  or choosing between memory, GORM, PostgreSQL, and NATS JetStream queue backends.
  Covers: WorkQueue interface, task enqueue/consume, delayed tasks, retry with backoff,
  dead-letter queue (DLQ), competing consumers, multi-process workers, at-least-once delivery,
  FOR UPDATE SKIP LOCKED, LISTEN/NOTIFY, JetStream work-queue streams, conformance testing.
  Keywords: workqueue, work queue, task queue, job queue, background job, background task,
  enqueue, consume, worker, worker pool, delayed task, scheduled task, retry, backoff,
  dead letter, DLQ, requeue, memory_workqueue, gorm_workqueue, postgres_workqueue,
  nats_workqueue, workqueuetest, SKIP LOCKED, LISTEN NOTIFY, pg_notify, JetStream,
  competing consumers, at-least-once, lease, idempotent handler.
---

# Workqueue Modules Reference

This skill provides detailed usage instructions for all packages in `github.com/weedbox/workqueue-modules` — a single `workqueue.WorkQueue` interface with interchangeable backends, so application code stays backend-agnostic.

## Module Index

### Shared Interface

| Package | Description | Documentation |
|---------|-------------|---------------|
| `workqueue` | `WorkQueue` interface, `Task` model, enqueue/consume options, backoff helper | [workqueue.md](./modules/workqueue.md) |

### Backends

| Package | Description | Documentation |
|---------|-------------|---------------|
| `memory_workqueue` | In-process, in-memory backend for development and tests | [memory_workqueue.md](./modules/memory_workqueue.md) |
| `gorm_workqueue` | Database rows on any `database.DatabaseConnector` (PostgreSQL, SQLite, ...); optimistic claims, lease-based crash recovery | [gorm_workqueue.md](./modules/gorm_workqueue.md) |
| `postgres_workqueue` | PostgreSQL-native; `FOR UPDATE SKIP LOCKED` claims and LISTEN/NOTIFY push wakeups | [postgres_workqueue.md](./modules/postgres_workqueue.md) |
| `nats_workqueue` | NATS JetStream; push-based delivery with per-queue task and dead-letter streams | [nats_workqueue.md](./modules/nats_workqueue.md) |

### Testing

| Package | Description | Documentation |
|---------|-------------|---------------|
| `workqueuetest` | Conformance test suite every backend must pass | [workqueuetest.md](./modules/workqueuetest.md) |

---

## ⚠️ Critical Rules

### Source of Truth — Do NOT Fetch from GitHub

This file and its sub-files (`modules/*.md`) are the **complete reference** for `weedbox/workqueue-modules`. **DO NOT browse `github.com/weedbox/workqueue-modules`** to look up source code.

### Inject the Interface, NOT a Concrete Backend

Application code MUST depend on `workqueue.WorkQueue`, never on a backend type:

```go
// ✅ Correct - use interface
type Params struct {
    fx.In
    WorkQueue workqueue.WorkQueue
}

// ❌ Wrong - using concrete implementation
type Params struct {
    fx.In
    WorkQueue *gorm_workqueue.GormWorkQueue
}
```

### Single-Load Needs No name Tag; Multi-Load Does

Backends register via `fxmodule.InterfaceModule[workqueue.WorkQueue]`, so the single-load / multi-load rules match common-modules connectors:

```go
// Single backend loaded → no name tag
WorkQueue workqueue.WorkQueue

// Multiple backends loaded side by side → disambiguate by scope
Jobs   workqueue.WorkQueue `name:"jobs"`   // gorm_workqueue.Module("jobs")
Events workqueue.WorkQueue `name:"events"` // nats_workqueue.Module("events")
```

In tests running multiple `fx.App` instances in one process, call `fxmodule.ResetClaim[workqueue.WorkQueue]()` between apps.

### Handlers Must Be Idempotent

Every backend delivers **at-least-once**. Crash recovery (expired leases, unacked JetStream messages) can redeliver a task whose handler already ran. Design handlers so a duplicate execution is harmless.

---

## Quick Start

```bash
go get github.com/weedbox/workqueue-modules@latest
```

Load one backend module and inject the interface:

```go
import (
    "github.com/weedbox/common-modules/postgres_connector"
    "github.com/weedbox/workqueue-modules/postgres_workqueue"
    "github.com/weedbox/workqueue-modules/workqueue"
)

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        postgres_connector.Module("database"),
        postgres_workqueue.Module("workqueue"), // Phase 2, after its connector
    }, nil
}
```

Enqueue and consume:

```go
task, err := p.WorkQueue.Enqueue(ctx, "emails", payload,
    workqueue.WithDelay(30*time.Second),
    workqueue.WithMaxRetries(5),
    workqueue.WithMetadata(map[string]string{"tenant": "acme"}),
)

sub, err := p.WorkQueue.Consume(ctx, "emails",
    func(ctx context.Context, task *workqueue.Task) error {
        return send(task.Payload)
    },
    workqueue.WithConcurrency(4),
)
defer sub.Stop(context.Background())
```

> **Loading order:** backend modules belong in **Phase 2 (`loadModules`)** and must be declared **after** their backing connector (`postgres_connector` / `sqlite_connector` for the database backends, `nats_connector` for `nats_workqueue`). `memory_workqueue` has no dependency.

---

## Choosing a Backend

| | `memory_workqueue` | `gorm_workqueue` | `postgres_workqueue` | `nats_workqueue` |
|---|---|---|---|---|
| Durability | none (process-local) | database rows | database rows | replicated JetStream streams |
| Multi-process workers | no | yes (optimistic claims) | yes (`SKIP LOCKED`) | yes (shared durable consumer) |
| Cross-process delivery latency | n/a | poll interval (default 1s) | push (LISTEN/NOTIFY) | push, immediate |
| Extra infra | none | existing database (any dialect) | existing PostgreSQL | NATS with JetStream |
| Best for | tests, development | SQLite or mixed-dialect apps, modest volume | apps on PostgreSQL | high-volume or low-latency pipelines |

Decision shortcuts:

- **SQLite / single process** → `gorm_workqueue` (local enqueues wake workers immediately; scheduled tasks fire on time)
- **Already on PostgreSQL** → `postgres_workqueue` (same table layout as `gorm_workqueue`; switching later needs no data migration)
- **Already running NATS** or need high volume → `nats_workqueue`
- **Unit tests** → `memory_workqueue`, or run `workqueuetest.Run` against your target backend

## Shared Semantics (all backends)

- `Task.Attempts` = number of handler invocations (first delivery = 1); a handler panic counts as a failed attempt
- A task fails permanently after `MaxRetries + 1` failed attempts, then moves to the dead-letter queue with `LastError` and `DiedAt` set
- Retry delay: `RetryBackoff(base, max, attempt) = min(base * 2^(attempt-1), max)`; defaults 10s base / 10m cap
- `ListDeadTasks` orders oldest-first with limit/offset paging; `RequeueDeadTask` resets the attempt counter; unknown IDs return `workqueue.ErrTaskNotFound`
- `Subscription.Stop(ctx)` waits for in-flight handlers; `Done()` closes once fully stopped
- Handlers receive `context.Background()`, not the `Consume` call's context

See [workqueue.md](./modules/workqueue.md) for the full API reference.

---

## Related Skills

- [common-modules](../common-modules/SKILL.md) - Database and NATS connectors the backends build on
- [module-dev](../module-dev/SKILL.md) - `fxmodule.InterfaceModule` pattern used by every backend
- [project-dev](../project-dev/SKILL.md) - Project structure and application setup
