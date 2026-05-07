# nats_connector

NATS messaging client with JetStream and a work queue consumer (single-message and batch APIs).

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
```

### Environment Variables

```bash
export NATS_HOST=nats://localhost:4222
export NATS_PINGINTERVAL=10
export NATS_MAXPINGSOUTSTANDING=3
export NATS_MAXRECONNECTS=-1
export NATS_AUTH_CREDS=/path/to/user.creds
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

## Subject Patterns

| Pattern | Matches |
|---------|---------|
| `events.user.created` | Exact match |
| `events.*` | Single token: `events.user`, `events.order` |
| `events.>` | Multiple tokens: `events.user.created`, `events.order.shipped` |

## Related

- [nats_jetstream_server](./nats_jetstream_server.md) - Embedded NATS server
