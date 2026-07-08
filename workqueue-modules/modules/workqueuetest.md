# workqueuetest

Reusable conformance suite that exercises the `workqueue.WorkQueue` contract. Every backend in `workqueue-modules` runs the same suite, so retry, delay, and dead-letter semantics stay uniform across implementations. Run it against any new backend you write.

## Installation

```bash
go get github.com/weedbox/workqueue-modules/workqueuetest
```

## Usage

Provide a `Factory` that returns a fresh, started `WorkQueue` backed by isolated storage. Register cleanup via `t.Cleanup`:

```go
import (
    "testing"

    "github.com/weedbox/workqueue-modules/workqueue"
    "github.com/weedbox/workqueue-modules/workqueuetest"
)

func TestConformance(t *testing.T) {
    workqueuetest.Run(t, func(t *testing.T) workqueue.WorkQueue {
        return newMyBackend(t) // fresh instance, isolated storage
    })
}
```

## What It Covers

`Run` executes 11 subtests:

| Subtest | Verifies |
|---------|----------|
| `EnqueueAndConsume` | Basic delivery; payload, metadata, and `Attempts == 1` on first delivery |
| `DelayedDelivery` | `WithDelay` tasks are not delivered before `RunAt` |
| `RetryThenSucceed` | Failed attempts retry with backoff and eventually succeed |
| `RetriesExhaustedGoToDeadLetter` | Task dies after `MaxRetries + 1` failures with `LastError`/`DiedAt` set |
| `ZeroMaxRetriesDiesAfterFirstFailure` | `WithMaxRetries(0)` means exactly one attempt |
| `RequeueDeadTask` | Requeued dead task runs again with the attempt counter reset |
| `DeleteDeadTask` | Deleted dead task is gone; unknown IDs return `ErrTaskNotFound` |
| `ListDeadTasksPagination` | Oldest-first ordering and limit/offset paging |
| `CompetingConsumersDeliverOnce` | Concurrent consumers never double-deliver an attempt |
| `QueuesAreIsolated` | Tasks on one queue are invisible to consumers of another |
| `StopWaitsForInFlightHandler` | `Subscription.Stop` blocks until running handlers finish |

## Conventions Inside the Suite

- Queue names come from an atomic counter (`q1`, `q2`, ...) — unique per subtest and valid as NATS subject tokens, so one shared backend instance per test is fine as long as queues don't collide across test runs against persistent storage.
- Retry tests use `WithRetryBackoff(20ms, 100ms)` so the suite runs fast; the wait timeout is 15s.
- The suite only uses the public `workqueue` API — it never reaches into backend internals.

## Tips for Backend Authors

- Persistent backends should point the factory at throwaway storage (e.g. a uniquely named in-memory SQLite database, a dropped-and-recreated table, or `t.TempDir()` for an embedded NATS store).
- Run with `-race`; the suite exercises concurrent consumers deliberately.
- A deliberately **long** poll interval in the factory is a useful trick: if conformance still passes, delivery is proven to ride on the backend's event-driven path (local wakeups, NOTIFY, push) rather than the poll safety net.
