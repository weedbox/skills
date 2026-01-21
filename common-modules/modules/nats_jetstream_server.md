# nats_jetstream_server

Embedded NATS JetStream server for development and testing.

## Installation

```bash
go get github.com/weedbox/common-modules/nats_jetstream_server
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/nats_jetstream_server"

func preloadModules() ([]fx.Option, error) {
    return []fx.Option{
        nats_jetstream_server.Module("nats_server"),
    }, nil
}
```

## Configuration

### Server Basics

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0` | Server bind address |
| `{scope}.port` | `4222` | Client port |
| `{scope}.http_port` | `8222` | Monitoring HTTP port |
| `{scope}.max_connections` | `0` | Max client connections (0 = unlimited) |
| `{scope}.max_payload` | `1048576` | Max message payload (1MB) |
| `{scope}.log_level` | `info` | Log level |

### JetStream Settings

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.jetstream.enabled` | `true` | Enable JetStream |
| `{scope}.jetstream.memory_limit` | `1073741824` | Memory limit (1GB) |
| `{scope}.jetstream.storage_limit` | `10737418240` | Storage limit (10GB) |
| `{scope}.jetstream.store_dir` | `./data/jetstream` | Storage directory |

### Authentication

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.auth.username` | (empty) | Basic auth username |
| `{scope}.auth.password` | (empty) | Basic auth password |
| `{scope}.auth.token` | (empty) | Token authentication |

### TLS

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.tls.cert` | (empty) | Server certificate |
| `{scope}.tls.key` | (empty) | Server private key |
| `{scope}.tls.ca` | (empty) | CA certificate |

### Clustering

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.cluster.enabled` | `false` | Enable clustering |
| `{scope}.cluster.name` | (empty) | Cluster name |
| `{scope}.cluster.host` | (empty) | Cluster bind address |
| `{scope}.cluster.port` | `6222` | Cluster port |
| `{scope}.cluster.routes` | (empty) | Cluster routes |

### TOML Example

```toml
[nats_server]
host = "0.0.0.0"
port = 4222
http_port = 8222
log_level = "info"

[nats_server.jetstream]
enabled = true
memory_limit = 1073741824    # 1GB
storage_limit = 10737418240  # 10GB
store_dir = "./data/jetstream"

[nats_server.auth]
username = "admin"
password = "secret"
# OR
token = "my-secret-token"

[nats_server.tls]
cert = "/path/to/server.crt"
key = "/path/to/server.key"
ca = "/path/to/ca.crt"

[nats_server.cluster]
enabled = true
name = "my-cluster"
host = "0.0.0.0"
port = 6222
routes = ["nats://node2:6222", "nats://node3:6222"]
```

### Environment Variables

```bash
export NATS_SERVER_HOST=0.0.0.0
export NATS_SERVER_PORT=4222
export NATS_SERVER_HTTP_PORT=8222
export NATS_SERVER_JETSTREAM_ENABLED=true
export NATS_SERVER_JETSTREAM_STORE_DIR=./data/jetstream
```

## API Methods

```go
type Params struct {
    fx.In
    NATSServer *nats_jetstream_server.NATSJetStreamServer
}

// Get client connection URL
url := m.params.NATSServer.GetClientURL()
// Returns: "nats://0.0.0.0:4222"

// Get monitoring URL
httpURL := m.params.NATSServer.GetHTTPURL()
// Returns: "http://0.0.0.0:8222"

// Check if server is running
running := m.params.NATSServer.IsRunning()

// Get active connection count
count := m.params.NATSServer.GetConnectionCount()
```

## Monitoring Endpoints

The embedded server exposes HTTP monitoring endpoints:

| Endpoint | Description |
|----------|-------------|
| `GET /varz` | Server statistics |
| `GET /connz` | Connection information |
| `GET /routez` | Route information |
| `GET /subsz` | Subscription information |
| `GET /jsz` | JetStream statistics |

### Example Monitoring

```bash
# Server stats
curl http://localhost:8222/varz

# Connections
curl http://localhost:8222/connz

# JetStream info
curl http://localhost:8222/jsz
```

## Use Cases

### Development Environment

Run embedded server for local development:

```go
func preloadModules() ([]fx.Option, error) {
    return []fx.Option{
        nats_jetstream_server.Module("nats_server"),
        // Connector will connect to this server
        nats_connector.Module("nats"),
    }, nil
}
```

```toml
[nats_server]
port = 4222

[nats]
host = "nats://localhost:4222"
```

### Testing

Use embedded server in tests:

```go
func TestWithNATS(t *testing.T) {
    app := fx.New(
        nats_jetstream_server.Module("nats_server"),
        nats_connector.Module("nats"),
        // Your test modules...
    )

    ctx := context.Background()
    app.Start(ctx)
    defer app.Stop(ctx)

    // Run tests...
}
```

### Production Considerations

For production, prefer:
- Dedicated NATS cluster
- External NATS JetStream deployment
- Kubernetes-managed NATS

The embedded server is best for:
- Development
- Testing
- Single-node deployments
- Edge computing

## Complete Example

```go
func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        logger.Module(),

        // Start embedded NATS server
        nats_jetstream_server.Module("nats_server"),

        // Connect to it
        nats_connector.Module("nats"),

        // Your modules using NATS
        mymodule.Module("mymodule"),

        daemon.Module("daemon"),
    }, nil
}
```

## Related

- [nats_connector](./nats_connector.md) - NATS client connector
