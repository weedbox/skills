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
| `{scope}.enabled` | `true` | Whether to start the embedded server. Set `false` when connecting to an external NATS server. |
| `{scope}.host` | `0.0.0.0` | Server bind address (also used as `HTTPHost` and the cluster bind address). |
| `{scope}.port` | `4222` | Client port |
| `{scope}.http_port` | `8222` | Monitoring HTTP port |
| `{scope}.cluster_port` | `6222` | Cluster listen port (only used when `cluster.enabled=true`) |
| `{scope}.max_connections` | `65536` | Max client connections |
| `{scope}.max_payload` | `1048576` | Max message payload in bytes (1 MiB) |
| `{scope}.write_deadline` | `2s` | Write deadline (Go duration string, e.g. `2s`, `500ms`) |
| `{scope}.log_level` | `INFO` | `INFO` / `DEBUG` / `TRACE` (uppercase). `DEBUG` enables `opts.Debug`; `TRACE` enables `opts.Trace`. |

### JetStream Settings

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.jetstream.enabled` | `true` | Enable JetStream |
| `{scope}.jetstream.max_memory` | `1073741824` | Memory limit in bytes (1 GiB) |
| `{scope}.jetstream.max_storage` | `10737418240` | Storage limit in bytes (10 GiB) |
| `{scope}.jetstream.store_dir` | `./data/jetstream` | Storage directory (auto-created at startup) |

### Authentication

Auth is gated by `auth.enabled`; the connector chooses token auth first if `auth.token` is set, otherwise username+password.

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.auth.enabled` | `false` | Master switch for auth |
| `{scope}.auth.username` | (empty) | Basic auth username |
| `{scope}.auth.password` | (empty) | Basic auth password |
| `{scope}.auth.token` | (empty) | Token authentication (takes precedence over username/password) |

### TLS

TLS is gated by `tls.enabled`; `cert_file` and `key_file` are required when enabled. Providing `ca_file` additionally turns on **mutual TLS** (`tls.RequireAndVerifyClientCert`).

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.tls.enabled` | `false` | Master switch for TLS |
| `{scope}.tls.cert_file` | (empty) | Server certificate path |
| `{scope}.tls.key_file` | (empty) | Server private key path |
| `{scope}.tls.ca_file` | (empty) | CA certificate path — enables mTLS when set |

### Clustering

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.cluster.enabled` | `false` | Enable clustering |
| `{scope}.cluster.name` | (empty) | Cluster name |
| `{scope}.cluster.routes` | (empty) | Cluster route URLs, e.g. `["nats://node2:6222"]` |

> The cluster listens on `{scope}.host`:`{scope}.cluster_port` — there is no separate `cluster.host` / `cluster.port` key.

### TOML Example

```toml
[nats_server]
enabled = true
host = "0.0.0.0"
port = 4222
http_port = 8222
cluster_port = 6222
max_connections = 65536
max_payload = 1048576
write_deadline = "2s"
log_level = "INFO"

[nats_server.jetstream]
enabled = true
max_memory = 1073741824     # 1 GiB
max_storage = 10737418240   # 10 GiB
store_dir = "./data/jetstream"

[nats_server.auth]
enabled = true
username = "admin"
password = "secret"
# OR
# token = "my-secret-token"

[nats_server.tls]
enabled = true
cert_file = "/path/to/server.crt"
key_file = "/path/to/server.key"
ca_file = "/path/to/ca.crt"   # presence enables mTLS

[nats_server.cluster]
enabled = true
name = "my-cluster"
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
