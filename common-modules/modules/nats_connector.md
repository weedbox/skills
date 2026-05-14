# nats_connector

NATS messaging client with JetStream, a work queue consumer (single-message and batch APIs), batch publishing (cross-stream async fan-out and single-stream atomic), and a distributed lock / `Once` helper.

## Installation

```bash
go get github.com/weedbox/common-modules/nats_connector
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/nats_connector"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        nats_connector.Module("nats"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0:32803` | NATS server address |
| `{scope}.pingInterval` | `10` | Ping interval (seconds) |
| `{scope}.maxPingsOutstanding` | `3` | Max missed pings before disconnect |
| `{scope}.maxReconnects` | `-1` | Max reconnects (-1 = unlimited) |
| `{scope}.auth.creds` | (empty) | Credentials file path |
| `{scope}.auth.nkey` | (empty) | NKey seed file path |
| `{scope}.tls.cert` | (empty) | TLS certificate path |
| `{scope}.tls.key` | (empty) | TLS key path |
| `{scope}.tls.ca` | (empty) | TLS CA certificate path |
| `{scope}.lock.bucket` | `{scope}_locks` | KV bucket backing the distributed lock (created lazily on first use) |
| `{scope}.lock.replicas` | `3` | KV bucket replica count. Auto-falls back to `1` when the server is not clustered, or retries with `1` when the cluster is smaller than the requested count |
| `{scope}.lock.defaultTTL` | `30s` | Default lock lease when `LockConfig.TTL` is zero. Minimum `1s` (NATS per-message TTL floor) |

### TOML Example

```toml
[nats]
host = "nats://localhost:4222"
pingInterval = 10
maxPingsOutstanding = 3
maxReconnects = -1

[nats.auth]
creds = "/path/to/user.creds"
# OR
nkey = "/path/to/user.nk"

[nats.tls]
cert = "/path/to/client.crt"
key = "/path/to/client.key"
ca = "/path/to/ca.crt"

[nats.lock]
bucket = "nats_locks"
replicas = 3
defaultTTL = "30s"
```

### Environment Variables

```bash
export NATS_HOST=nats://localhost:4222
export NATS_PINGINTERVAL=10
export NATS_MAXPINGSOUTSTANDING=3
export NATS_MAXRECONNECTS=-1
export NATS_AUTH_CREDS=/path/to/user.creds
export NATS_LOCK_BUCKET=nats_locks
export NATS_LOCK_REPLICAS=3
export NATS_LOCK_DEFAULTTTL=30s
```

## Core Messaging

### Publishing

```go
type Params struct {
    fx.In
    NATS *nats_connector.NATSConnector
}

func (m *MyModule) Publish(subject string, data []byte) error {
    return m.params.NATS.GetConnection().Publish(subject, data)
}
```

### Subscribing

```go
func (m *MyModule) OnStart(ctx context.Context) error {
    conn := m.params.NATS.GetConnection()

    conn.Subscribe("events.>", func(msg *nats.Msg) {
        m.handleMessage(msg)
    })

    // Queue subscription (load balanced)
    conn.QueueSubscribe("tasks.>", "workers", func(msg *nats.Msg) {
        m.handleTask(msg)
    })

    return nil
}
```

### Request-Reply

```go
msg, err := m.params.NATS.GetConnection().Request(subject, data, 5*time.Second)
```

## JetStream

### Publishing to Stream

```go
js := m.params.NATS.GetJetStreamContext()
ack, err := js.Publish(subject, data)
```

### Creating Streams

```go
js := m.params.NATS.GetJetStreamContext()
_, err := js.AddStream(&nats.StreamConfig{
    Name:     "EVENTS",
    Subjects: []string{"events.>"},
    Storage:  nats.FileStorage,
    MaxAge:   24 * time.Hour,
})
```

## Batch Publishing

The connector exposes two distinct entry points for publishing many messages at once. Pick based on whether you need atomic semantics:

| API | Stream scope | Guarantee | Use when |
|-----|--------------|-----------|----------|
| `BatchPublish` | Items may target subjects across **multiple streams** | None — async fan-out, per-item ack | Bulk publishing for throughput; no transactional requirement |
| `AtomicPublish` | All items must target a **single stream** | All-or-nothing when stream supports atomic batch; transparent fallback to async otherwise | One-shot commit semantics where supported |

Both APIs are methods on `*nats_connector.NATSConnector`.

### BatchPublish (cross-stream, async fan-out)

```go
items := []nats_connector.BatchPublishItem{
    {Subject: "events.created", Data: payload1},
    {Subject: "audit.created",  Data: payload2}, // different stream is fine
    {Subject: "events.created", Data: payload3},
}

res, err := nc.BatchPublish(ctx, items)
if err != nil {
    return err
}
for i, ack := range res.Acks {
    log.Printf("item %d → stream=%s seq=%d", i, ack.Stream, ack.Sequence)
}
```

`BatchPublish` calls `PublishMsgAsync` for every item and waits for all acks. **There is no atomic semantic** — each item is independently confirmed. If any item's ack returns an error the call returns the first error and abandons the rest; messages already flushed to the server may still land.

**Options:**

| Option | Purpose |
|--------|---------|
| `WithBatchTimeout(d)` | Overall wait cap (default `30s`). Only consulted when the caller's `ctx` has no deadline. |

**Result:**

```go
type BatchPublishResult struct {
    Count int                  // number of items
    Acks  []*jetstream.PubAck  // per-item ack in input order; each carries Stream
}
```

### AtomicPublish (single stream, atomic-or-fallback)

```go
res, err := nc.AtomicPublish(ctx, items)
if err != nil {
    return err
}
log.Printf("published %d msgs via %s (stream=%s, seq=%d..%d, batch=%s)",
    res.Count, res.Mode, res.Stream, res.FirstSeq, res.LastSeq, res.BatchID)
```

`AtomicPublish` resolves the target stream from the first item's subject (or `WithAtomicStream`) and uses JetStream Atomic Batch Publish when the stream advertises `AllowAtomicPublish` (NATS 2.12+). When the stream does not support atomic batches, it **transparently falls back to async fan-out** and returns `Mode == AtomicPublishModeAsyncFallback`. Use `WithStrictAtomic()` to disable the fallback.

**Options:**

| Option | Purpose |
|--------|---------|
| `WithAtomicStream(name)` | Explicit stream name; skips the `StreamNameBySubject` lookup. **Required when subjects are bound to more than one stream** |
| `WithAtomicTimeout(d)` | Overrides the commit/wait timeout (default `30s`). Only consulted when `ctx` has no deadline |
| `WithAtomicID(id)` | Supplies an explicit `Nats-Batch-Id` (otherwise auto-generated) |
| `WithStrictAtomic()` | Disables the async fallback; returns an error if the target stream does not support atomic batch publish |

**Result:**

```go
type AtomicPublishMode string

const (
    AtomicPublishModeAtomic        AtomicPublishMode = "atomic"
    AtomicPublishModeAsyncFallback AtomicPublishMode = "async-fallback"
)

type AtomicPublishResult struct {
    Mode     AtomicPublishMode    // "atomic" or "async-fallback"
    Count    int
    Stream   string               // target stream
    BatchID  string               // Nats-Batch-Id (empty in fallback mode)
    FirstSeq uint64               // first assigned stream sequence
    LastSeq  uint64               // last assigned stream sequence
    Acks     []*jetstream.PubAck  // populated only in fallback mode
}
```

In atomic mode the server returns one aggregate ack, so `Acks` is empty and `FirstSeq` is derived as `LastSeq - Count + 1`. In fallback mode `Acks` is populated in input order and `FirstSeq`/`LastSeq` reflect the min/max sequence observed.

### Atomic Semantics & Failure Modes

- **Atomic path** — `len(items)-1` non-commit messages are published via `conn.PublishMsg`, then the last item is sent via `RequestMsg` with `Nats-Batch-Commit: 1`. The aggregate ack carries the batch id, the count, and the last assigned stream sequence.
- **Header preservation** — `BatchPublishItem.Header` is cloned per message; in atomic mode the connector injects the `Nats-Batch-*` headers on top of the caller's headers.
- **Partial-failure recovery** — if an intermediate publish errors or the caller cancels `ctx` before commit, the partial batch staged on the server is discarded automatically when its atomic-batch staging timeout elapses. You can safely retry with a fresh call (the connector generates a new `Nats-Batch-Id` on each retry unless you pin one with `WithAtomicID`).
- **Commit ambiguity** — if the commit `RequestMsg` times out or the connection drops after the commit reaches the server, the server may still apply the batch while the caller observes an error. The outcome is indeterminable without correlating via your own application-level dedup key (e.g. `Nats-Msg-Id`).
- **Do not reuse a `WithAtomicID` value across two calls** — the server may treat the second call as a continuation of the first.

### Enabling Atomic Batch on a Stream

```go
js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:               "EVENTS",
    Subjects:           []string{"events.>"},
    AllowAtomicPublish: true, // required for AtomicPublish to use the atomic path
})
```

Atomic batch publish is **not propagated through mirrors or source streams**: the server strips `Nats-Batch-*` headers when replicating, so downstream readers cannot reconstruct the batch boundary. If your consumers need that, carry your own correlation header (e.g. `X-Tx-Id`).

## Work Queue Consumer

The consumer wraps JetStream pull-consume with concurrency control, durable consumer creation, exponential nack backoff, in-progress heartbeats, panic recovery, and optional restart loops. Two handler styles are available:

- **Single-message** (`Start* + MessageHandler`) — one message per handler call.
- **Batch** (`StartBatch* + BatchMessageHandler`) — up to `BatchSize` messages per call, all-or-nothing acknowledgement.

### Creating a Consumer

```go
import (
    "context"
    "log"
    "time"

    "github.com/nats-io/nats.go/jetstream"
    "github.com/weedbox/common-modules/nats_connector"
)

type Params struct {
    fx.In
    NATS *nats_connector.NATSConnector
}

func (w *Worker) StartConsumer() error {
    cfg := nats_connector.NewWorkQueueConsumerConfig()
    cfg.ConsumerName = "my-worker"
    cfg.Subjects = []string{"tasks.>"}
    cfg.MaxConcurrent = 10
    cfg.AckWait = 30 * time.Second
    cfg.OnError = func(err error) { log.Printf("nats: %v", err) }

    consumer, err := w.params.NATS.NewWorkQueueConsumer("my-stream", cfg)
    if err != nil {
        return err
    }

    // Blocking; returns when Shutdown() is called.
    return consumer.Start(func(ctx context.Context, msg jetstream.Msg) error {
        // return nil  → ack
        // return err  → NakWithDelay (exponential backoff)
        return processOne(ctx, msg)
    })
}
```

The `streamName` argument to `NewWorkQueueConsumer` is used to load `*nats.StreamInfo` if `cfg.Stream` is nil — the stream must already exist.

### WorkQueueConfig

```go
type WorkQueueConfig struct {
    Conn          *nats.Conn       // optional; defaults to connector's connection
    Stream        *nats.StreamInfo // optional; fetched by streamName when nil
    ConsumerName  string           // durable consumer name (used for load balancing)
    Subjects      []string         // FilterSubjects on the consumer
    MaxConcurrent int              // max in-flight handler invocations (default 10)
    AckWait       time.Duration    // ack deadline (default 30s)
    MaxRetries    int              // -1 unlimited; MaxDeliver = MaxRetries+1
    MaxAckPending int              // ack-pending cap (default = MaxConcurrent)
    OnError       ErrorHandler     // error/log callback (no-op if nil)

    // Restart loop (used by StartWithRestart / StartBatchWithRestart)
    MaxRestarts      int           // -1 unlimited (default), 0 disabled, N capped
    RestartBaseDelay time.Duration // default 1s
    RestartMaxDelay  time.Duration // default 30s

    // Batch APIs only
    BatchSize    int           // messages per Fetch (default 100)
    BatchMaxWait time.Duration // max wait to fill a batch (default 1s)
}
```

`NewWorkQueueConsumerConfig()` returns a populated default config; modify the fields you care about.

### Lifecycle APIs

| Method | Style | Blocking? | Restart on failure? |
|--------|-------|-----------|---------------------|
| `Start(MessageHandler)` | single | yes | no |
| `StartAsync(MessageHandler)` | single | no | no |
| `StartWithRestart(MessageHandler)` | single | yes | yes (exponential backoff) |
| `StartBatch(BatchMessageHandler)` | batch | yes | no |
| `StartBatchAsync(BatchMessageHandler)` | batch | no | no |
| `StartBatchWithRestart(BatchMessageHandler)` | batch | yes | yes |
| `Shutdown()` | — | — | cancels context, waits for in-flight handlers |
| `Done() <-chan struct{}` | — | — | closed when context is cancelled |

### Handler Signatures

```go
type MessageHandler      func(ctx context.Context, msg jetstream.Msg) error
type BatchMessageHandler func(ctx context.Context, msgs []jetstream.Msg) error
```

Single-message: return `nil` → ack; return `error` → `NakWithDelay` with exponential backoff (jittered, derived from `AckWait`); panic → recovered, treated as error.

Batch: same rule applied to the whole batch (see [Batch Processing](#batch-processing)).

### Acknowledgement Behaviour

The consumer manages ack/nak/in-progress for you. Do **not** call `msg.Ack()` / `msg.Nak()` / `msg.InProgress()` from your handler.

- While the handler runs, `InProgress()` is published every `AckWait/3` (min 1s) so long-running work does not get redelivered.
- On `Shutdown()` mid-process, in-flight messages are `Nak()`'d for immediate redelivery on restart.
- Nack delay grows as `AckWait/4 * 2^(attempt-1)`, capped at `AckWait - 1s`, with up to ±20% jitter.

## Batch Processing

When the handler is more efficient processing many messages together (bulk DB inserts, batched HTTP forwards, aggregations), use the `StartBatch*` family. The consumer pulls up to `BatchSize` messages per iteration (waiting up to `BatchMaxWait` for the batch to fill) and invokes the handler once per batch.

```go
func (w *Worker) StartBatchConsumer() error {
    cfg := nats_connector.NewWorkQueueConsumerConfig()
    cfg.ConsumerName = "my-bulk-worker"
    cfg.Subjects     = []string{"events.>"}
    cfg.MaxConcurrent = 4                       // up to 4 batches in flight
    cfg.BatchSize     = 200                     // up to 200 messages per batch
    cfg.BatchMaxWait  = 500 * time.Millisecond
    cfg.AckWait       = 60 * time.Second        // give the bulk handler time
    cfg.MaxAckPending = 1000                    // must be >= MaxConcurrent * BatchSize
    cfg.OnError       = func(err error) { log.Printf("nats: %v", err) }

    consumer, err := w.params.NATS.NewWorkQueueConsumer("events-stream", cfg)
    if err != nil {
        return err
    }

    return consumer.StartBatch(func(ctx context.Context, msgs []jetstream.Msg) error {
        rows := make([]Row, 0, len(msgs))
        for _, m := range msgs {
            rows = append(rows, parse(m.Data()))
        }
        // One DB round-trip for the whole batch.
        if err := w.db.BulkInsert(ctx, rows); err != nil {
            return err  // entire batch is nacked with backoff and redelivered
        }
        return nil      // entire batch is acked
    })
}
```

### Batch Semantics

- **All-or-nothing ack** — return `nil` → every message in the batch is acked; return `error` (or panic) → every message gets `NakWithDelay` with the same backoff as the single-message path.
- **No per-message control** — handlers MUST NOT call `msg.Ack()` / `msg.Nak()` / `msg.InProgress()`. If you need per-message acknowledgement, use the single-message `Start*` API instead — `MaxConcurrent` already gives parallelism.
- **In-progress heartbeat** — every `AckWait/3` (min 1s), `InProgress()` is sent for each message in the batch. For very large `BatchSize` (e.g. 1000) this becomes `BatchSize` publishes per tick — tune `AckWait` and `BatchSize` together.
- **Backoff** — derived from the first message's `NumDelivered`; all messages in the batch receive the same delay.
- **Shutdown** — `Shutdown()` cancels the context; in-flight batches are `Nak()`'d immediately. Graceful drain may wait up to `BatchMaxWait` for an in-progress `Fetch` to return.
- **`MaxConcurrent * BatchSize <= MaxAckPending`** — total in-flight cap. If exceeded, the consumer logs a warning at start-up via `OnError` and will throttle on the JetStream ack-pending limit.

### When to Use Single vs Batch

- **Single (`Start`)** — heterogeneous messages, per-message routing decisions, side-effects that don't batch (e.g. one-to-one external API calls).
- **Batch (`StartBatch`)** — homogeneous messages where the underlying work amortises (`INSERT ... VALUES (...), (...), ...`, bulk-write to S3, batched stream forwarding, reductions/aggregations). Avoid batch if any single failure should not redeliver the whole batch.

### Restart Loop

`StartWithRestart` / `StartBatchWithRestart` wrap the loop with panic recovery and exponential-backoff restart, controlled by `MaxRestarts` (default `-1` = unlimited; `0` disables restart), `RestartBaseDelay` (1s), `RestartMaxDelay` (30s). Use this for long-running workers where you'd rather restart than crash the process. `Shutdown()` interrupts the restart wait cleanly.

## Distributed Lock & Once

The connector exposes a JetStream KV-backed distributed lock and a `Once` helper for cross-instance "init exactly once" patterns. Typical use cases: schema migrations, stream/bucket bootstrapping, seed data, registering one-time external resources.

Both helpers share a single KV bucket (default `{scope}_locks`), created lazily on first use with `History=1` and `LimitMarkerTTL=1h` (enables per-key TTL). Keys are namespaced: `lock.<your-key>` for the mutex, `done.<your-key>` for the `Once` sentinel.

### Low-Level Lock

```go
type Params struct {
    fx.In
    NATS *nats_connector.NATSConnector
}

func (m *MyModule) DoCriticalWork(ctx context.Context) error {
    lock, err := m.params.NATS.NewLock(nats_connector.LockConfig{
        Key: "user-module.migration",
        TTL: 30 * time.Second, // lease; auto-released if the holder crashes
    })
    if err != nil {
        return err
    }

    if err := lock.Lock(ctx); err != nil {
        return err
    }
    defer lock.Unlock(ctx)

    // ... critical section ...
    return nil
}
```

### LockConfig

```go
type LockConfig struct {
    Key               string        // required: lock identity (mutually exclusive across instances)
    Bucket            string        // optional: override the default lock bucket
    TTL               time.Duration // lease duration (default: lock.defaultTTL; min: MinLockTTL = 1s)
    HeartbeatInterval time.Duration // renewal interval (default: TTL/3)
    OwnerID           string        // optional: identity stamped on the lock key (default: "<hostname>-<random>")
    OnLost            func(error)   // optional: callback when heartbeat detects loss
}
```

### Lock Methods

| Method | Description |
|--------|-------------|
| `Lock(ctx) error` | Blocks until acquired, ctx cancelled, or infra error |
| `TryLock(ctx) (bool, error)` | Single attempt: `(true, nil)` acquired; `(false, nil)` held by someone else; `(false, err)` infra error |
| `Unlock(ctx) error` | Releases via CAS Delete. Idempotent. Returns `ErrLockLost` if the heartbeat already detected loss or the revision no longer matches |
| `Done() <-chan struct{}` | Closes when the lock is released (by `Unlock` or detected loss) |
| `OwnerID() string` | The identity stamped on the lock key |
| `Bucket() string` | The KV bucket the lock lives in |

### Lock Semantics

- **Atomic acquire** — `Create` on the KV bucket is atomic; only one caller wins. The rest see `ErrKeyExists` and either return `(false, nil)` from `TryLock` or block in `Lock` until the holder releases.
- **Lease + heartbeat** — while held, a background goroutine renews the lease every `HeartbeatInterval` (default `TTL/3`) via CAS `Update`. If the holder crashes, the per-key TTL on the bucket expires the key and another caller can take over. The NATS server enforces `TTL >= 1s` (`MinLockTTL`).
- **CAS release** — `Unlock` uses `Delete` with `LastRevision`, so a stale holder cannot delete a lock that has already been taken over by someone else.
- **`Done()` channel** — closes when the lock is released, either by `Unlock` or because the heartbeat detected a loss (external purge, expired lease, network split). Useful to abort the critical section.
- **`ErrLockLost`** — returned by `Unlock` when the local view of the lock no longer matches the bucket. Callers can branch via `errors.Is(err, nats_connector.ErrLockLost)`.
- **Lock blocking uses KV `Watch`** — callers wake promptly when the lock is released (no busy polling), with a `ttl + 1s` safety-net timeout if a delete event is missed.
- **Single-use instance** — a `Lock` is single-use; allocate a new `Lock` after `Unlock` if you need to re-acquire. Concurrent use of a single instance from multiple goroutines is NOT supported.

### High-Level `Once`

```go
// First caller to acquire the underlying lock runs fn; subsequent callers
// (current and future runs) skip fn and return nil once fn succeeds.
err := m.params.NATS.Once(ctx, "user-module.bootstrap", func(ctx context.Context) error {
    return migrateSchema(ctx)
})
```

### `Once` Semantics

- **Retries on failure** — if `fn` returns an error, the `done.<key>` sentinel is NOT written, so the next caller (this run or future) will run `fn` again.
- **Idempotent fast-path** — once `fn` succeeds, the sentinel is permanent, making future calls effectively a single KV `Get` (no lock acquisition).
- **Re-check under lock** — after acquiring the lock, `Once` re-checks the sentinel, so racing callers that all observed an empty sentinel still only run `fn` once.
- **Lock released on a fresh context** — even if the caller's ctx is cancelled mid-`fn`, `Once` releases the lock with a 5s timeout context so the lock doesn't get stuck.

## Subject Patterns

| Pattern | Matches |
|---------|---------|
| `events.user.created` | Exact match |
| `events.*` | Single token: `events.user`, `events.order` |
| `events.>` | Multiple tokens: `events.user.created`, `events.order.shipped` |

## Related

- [nats_jetstream_server](./nats_jetstream_server.md) - Embedded NATS server
