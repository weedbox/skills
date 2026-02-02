# daemon

Service lifecycle management with ready/health status tracking.

## Installation

```bash
go get github.com/weedbox/common-modules/daemon
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/daemon"

// IMPORTANT: Place daemon module LAST in module list
// It marks service as ready after all other modules initialize
func afterModules() ([]fx.Option, error) {
    return []fx.Option{
        daemon.Module("daemon"),
    }, nil
}
```

## Critical: Module Order

The daemon module **must be placed at the end** of all other modules because:

1. The `OnStart` method marks the service as ready
2. This ensures all dependencies are initialized first
3. Health checks will return "ready" only after full initialization

```go
func initModules() ([]fx.Option, error) {
    return []fx.Option{
        // Phase 1: Preload
        fx.Supply(config),
        logger.Module(),

        // Phase 2: Load (your modules)
        http_server.Module("http_server"),
        mymodule.Module("mymodule"),

        // Phase 3: After (daemon LAST)
        daemon.Module("daemon"),
    }, nil
}
```

## API Methods

```go
// Check if service is ready (all modules initialized)
daemon.Ready() // returns bool

// Get health status
daemon.GetHealthStatus() // returns HealthStatus
```

## Health Status Constants

```go
const (
    HealthStatus_Healthy   HealthStatus = "healthy"
    HealthStatus_Unhealthy HealthStatus = "unhealthy"
)
```

## Integration with healthcheck_apis

When combined with `healthcheck_apis` module, provides HTTP endpoints:

| Endpoint | Description | Success | Failure |
|----------|-------------|---------|---------|
| `GET /healthz` | Health status | 200 `{"status": "ok"}` | 500 `{"status": "unhealthy"}` |
| `GET /ready` | Ready state | 200 `{"ready": true}` | 500 `{"ready": false}` |

## Complete Example

```go
func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        logger.Module(),
        http_server.Module("http_server"),
        healthcheck_apis.Module("healthcheck"),

        // Your application modules
        mymodule.Module("mymodule"),

        // Daemon MUST be last
        daemon.Module("daemon"),
    }, nil
}
```

## Configuration

This module has no configuration parameters. It relies on Fx's built-in lifecycle management for shutdown handling.
