# workqueue

Shared contract implemented by every backend in `workqueue-modules`. Backends register themselves as implementations of `workqueue.WorkQueue` via `fxmodule.InterfaceModule`, so an application injects the interface and can swap or load multiple backends side by side.

## Installation

```bash
go get github.com/weedbox/workqueue-modules/workqueue
```

## The Interface

```go
type WorkQueue interface {
    // Enqueue adds a task carrying payload to the named queue and returns
    // the stored task (with its assigned ID).
    Enqueue(ctx context.Context, queue string, payload []byte, opts ...EnqueueOption) (*Task, error)

    // Consume starts delivering tasks from the named queue to handler and
    // returns immediately with a Subscription controlling the worker pool.
    // Multiple Consume calls (across processes or within one) compete for
    // tasks; each task is delivered to one consumer at a time.
    Consume(ctx context.Context, queue string, handler Handler, opts ...ConsumeOption) (Subscription, error)

    // ListDeadTasks returns tasks that exhausted their retries, ordered by
    // the time they died (oldest first).
    ListDeadTasks(ctx context.Context, queue string, limit, offset int) ([]*Task, error)

    // RequeueDeadTask moves a dead task back to the pending queue with its
    // attempt counter reset. Returns ErrTaskNotFound if absent.
    RequeueDeadTask(ctx context.Context, queue string, taskID string) error

    // DeleteDeadTask permanently removes a dead task. Returns
    // ErrTaskNotFound if absent.
    DeleteDeadTask(ctx context.Context, queue string, taskID string) error
}

type Handler func(ctx context.Context, task *Task) error

type Subscription interface {
    Stop(ctx context.Context) error // waits for in-flight handlers; idempotent
    Done() <-chan struct{}          // closed once fully stopped
}

var ErrTaskNotFound = errors.New("workqueue: task not found")
```

## Task Model

```go
type Task struct {
    ID         string            // unique task ID (UUID), assigned on Enqueue
    Queue      string            // queue name
    Payload    []byte            // opaque task body; encoding is up to the application
    Metadata   map[string]string // optional application-defined key/value pairs
    Attempts   int               // handler invocations so far (inside a handler: current invocation, first = 1)
    MaxRetries int               // retries allowed after the first failed attempt
    EnqueuedAt time.Time         // original enqueue time
    RunAt      time.Time         // earliest delivery time
    LastError  string            // most recent handler error (populated on dead tasks)
    DiedAt     time.Time         // when the task entered the DLQ; zero for live tasks
}
```

## Options

### Enqueue Options

| Option | Default | Purpose |
|--------|---------|---------|
| `WithDelay(d)` | none | Postpone first delivery by `d` |
| `WithRunAt(t)` | now | Deliver no earlier than `t`; takes precedence over `WithDelay` |
| `WithMaxRetries(n)` | `3` (`DefaultMaxRetries`) | Retry budget after the first failed attempt; handler runs at most `n+1` times. Negative values are treated as 0 |
| `WithMetadata(md)` | none | Attach key/value pairs; later calls merge over earlier ones |

### Consume Options

| Option | Default | Purpose |
|--------|---------|---------|
| `WithConcurrency(n)` | `10` (`DefaultConcurrency`) | Max handler invocations in flight for this subscription; values < 1 are clamped to 1 |
| `WithRetryBackoff(base, max)` | `10s` / `10m` | Bounds for exponential retry backoff |

### Retry Backoff

All backends share one formula, exported for reuse:

```go
// delay before the attempt-th retry (attempt >= 1)
workqueue.RetryBackoff(base, max, attempt) // = min(base * 2^(attempt-1), max)
```

With defaults: 10s, 20s, 40s, ... capped at 10m.

## Semantics (guaranteed by every backend)

- **Attempts** — `Task.Attempts` counts handler invocations (first delivery = 1). A handler panic counts as a failed attempt; backends recover it.
- **Dead-letter** — a task moves to the DLQ when a handler fails and `Attempts > MaxRetries`, with `LastError` and `DiedAt` recorded. `RequeueDeadTask` resets `Attempts`/`LastError`/`DiedAt` and makes the task immediately runnable.
- **Delivery** — at-least-once. Competing consumers never double-deliver an attempt under normal operation, but crash recovery may redeliver; make handlers idempotent.
- **Handler context** — handlers receive `context.Background()`, not the `Consume` call's context; cancelling the `Consume` ctx does not cancel running handlers. Stop via `Subscription.Stop`.
- **Shutdown** — `Stop(ctx)` waits for in-flight handlers to finish or `ctx` to expire, whichever comes first.

## Backend-Specific Features

The interface covers only operations every backend implements cleanly. Backend-specific features (e.g. `AutoMigrate` on the database backends, `Close` on postgres) live on the concrete type and are reached in the module's own lifecycle hooks — application code should not depend on them.
