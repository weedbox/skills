# nats_connector

NATS messaging client with JetStream and work queue support.

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
| `{scope}.pingInterval` | `10s` | Ping interval |
| `{scope}.maxPingsOutstanding` | `3` | Max missed pings |
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
pingInterval = "10s"
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
export NATS_PING_INTERVAL=10s
export NATS_MAX_RECONNECTS=-1
```

## Core Messaging

### Publishing

```go
type Params struct {
    fx.In
    NATSConnector *nats_connector.NATSConnector
}

func (m *MyModule) Publish(subject string, data []byte) error {
    conn := m.params.NATSConnector.GetConnection()
    return conn.Publish(subject, data)
}

// Publish with JSON
func (m *MyModule) PublishJSON(subject string, v interface{}) error {
    conn := m.params.NATSConnector.GetConnection()
    data, err := json.Marshal(v)
    if err != nil {
        return err
    }
    return conn.Publish(subject, data)
}
```

### Subscribing

```go
func (m *MyModule) OnStart(ctx context.Context) error {
    conn := m.params.NATSConnector.GetConnection()

    // Simple subscription
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
func (m *MyModule) Request(subject string, data []byte) ([]byte, error) {
    conn := m.params.NATSConnector.GetConnection()
    msg, err := conn.Request(subject, data, 5*time.Second)
    if err != nil {
        return nil, err
    }
    return msg.Data, nil
}
```

## JetStream

### Publishing to Stream

```go
func (m *MyModule) PublishToStream(subject string, data []byte) error {
    js := m.params.NATSConnector.GetJetStreamContext()
    _, err := js.Publish(subject, data)
    return err
}

// With acknowledgement
func (m *MyModule) PublishWithAck(subject string, data []byte) error {
    js := m.params.NATSConnector.GetJetStreamContext()
    ack, err := js.Publish(subject, data)
    if err != nil {
        return err
    }
    m.logger.Info("Published",
        zap.String("stream", ack.Stream),
        zap.Uint64("seq", ack.Sequence),
    )
    return nil
}
```

### Creating Streams

```go
func (m *MyModule) OnStart(ctx context.Context) error {
    js := m.params.NATSConnector.GetJetStreamContext()

    // Create or update stream
    _, err := js.AddStream(&nats.StreamConfig{
        Name:     "EVENTS",
        Subjects: []string{"events.>"},
        Storage:  nats.FileStorage,
        MaxAge:   24 * time.Hour,
    })

    return err
}
```

## Work Queue Consumer

The connector provides a convenient work queue consumer for processing messages.

```go
import "github.com/weedbox/common-modules/nats_connector"

func (m *MyModule) OnStart(ctx context.Context) error {
    consumer := nats_connector.NewWorkQueueConsumer(
        m.params.NATSConnector,
        "events.>",           // subject pattern
        "my-consumer-group",  // consumer name (for load balancing)
        m.handleMessage,      // handler function
    )

    // Configure concurrency
    consumer.SetConcurrency(10)

    // Start consuming (blocking)
    go consumer.Start()

    // Or async (non-blocking)
    // consumer.StartAsync()

    return nil
}

func (m *MyModule) handleMessage(msg *nats.Msg) error {
    var event Event
    if err := json.Unmarshal(msg.Data, &event); err != nil {
        return err
    }

    // Process event...

    return nil // Message will be acknowledged
}
```

### Work Queue Configuration

```go
consumer := nats_connector.NewWorkQueueConsumer(
    connector,
    "tasks.>",
    "task-workers",
    handler,
)

// Set number of concurrent handlers
consumer.SetConcurrency(20)

// Set acknowledgement timeout
consumer.SetAckWait(30 * time.Second)

// Set max delivery attempts
consumer.SetMaxDeliver(5)
```

## Subject Patterns

| Pattern | Matches |
|---------|---------|
| `events.user.created` | Exact match |
| `events.*` | Single token: `events.user`, `events.order` |
| `events.>` | Multiple tokens: `events.user.created`, `events.order.shipped` |

## Error Handling

```go
func (m *MyModule) handleMessage(msg *nats.Msg) error {
    var event Event
    if err := json.Unmarshal(msg.Data, &event); err != nil {
        // Return error to trigger redelivery
        return err
    }

    if err := m.processEvent(event); err != nil {
        // Log and return error for retry
        m.logger.Error("Failed to process event", zap.Error(err))
        return err
    }

    return nil // Success - message acknowledged
}
```

## Related

- [nats_jetstream_server](./nats_jetstream_server.md) - Embedded NATS server
