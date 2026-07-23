# daemon

Service lifecycle management with composite ready / health status tracking.

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

When the [lifecycle](./lifecycle.md) module is also loaded, place it **right before** daemon (`lifecycle.Module("lifecycle")`, then `daemon.Module("daemon")`) — daemon stays last, so readiness flips only after the PostStart phase completes.

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
// Composite readiness: true only when the daemon has started AND every
// registered ready checker returns true. Safe for concurrent use.
daemon.Ready() // returns bool

// Get health status
daemon.GetHealthStatus() // returns HealthStatus

// Composite-readiness plug-in API. Checkers are invoked on every Ready()
// call, so they should be cheap and non-blocking — typically a single
// atomic / mutex-guarded field read.
daemon.RegisterReadyChecker(name string, fn func() bool)
daemon.UnregisterReadyChecker(name string)
```

## Composite Readiness

`Ready()` is a logical AND of the daemon's own `isReady` flag and every registered ready checker. Modules that have their own runtime "am I ready?" notion (cluster membership, leader election, warmed cache, established upstream connection, etc.) can plug themselves in so that `/ready` reflects the true composite state of the service — not just "all `OnStart` hooks returned".

```go
package cluster

import (
    "context"

    "github.com/weedbox/common-modules/daemon"
    "go.uber.org/fx"
)

type Params struct {
    fx.In

    Lifecycle fx.Lifecycle
    Daemon    *daemon.Daemon
    Cluster   *Cluster
}

func Register(p Params) {
    p.Lifecycle.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            // /ready only flips true once the cluster reports ready as well.
            p.Daemon.RegisterReadyChecker("cluster", func() bool {
                return p.Cluster.Ready()
            })
            return nil
        },
        OnStop: func(ctx context.Context) error {
            p.Daemon.UnregisterReadyChecker("cluster")
            return nil
        },
    })
}
```

**Semantics:**

- Registering with the same `name` replaces the previous checker.
- A `nil` `fn` is ignored (logged as a warning); use `UnregisterReadyChecker` to remove.
- `UnregisterReadyChecker` on an unknown name is a no-op.
- Pair register/unregister with `OnStart` / `OnStop` lifecycle hooks so a stopped module does not leave a dangling checker that holds `/ready` down.
- Checkers run on every call to `Ready()` — including every `/ready` HTTP hit — so keep them cheap.

## Health Status Constants

```go
type HealthStatus int32

const (
    HealthStatus_Healthy HealthStatus = iota // 0
    HealthStatus_Unhealthy                   // 1
)
```

`GetHealthStatus()` returns the current value; the daemon initialises it to `HealthStatus_Healthy` on construction. There is no public setter — modules that need to flag unhealthiness should fail their `OnStart` hook or expose their own status, not mutate the daemon's field directly.

## Integration with healthcheck_apis

When combined with `healthcheck_apis` module, provides HTTP endpoints:

| Endpoint | Description | Success | Failure |
|----------|-------------|---------|---------|
| `GET /healthz` | Health status | 200 `{"status": "ok"}` | 500 `{"status": "unhealthy"}` |
| `GET /ready` | Composite ready state (daemon started **and** every registered ready checker reports true) | 200 `{"ready": true}` | 500 `{"ready": false}` |

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
