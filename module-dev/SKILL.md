---
name: module-dev
description: Help with weedbox framework module development including creating new modules, dependency injection with Uber FX, configuration management, and lifecycle hooks. Use when creating new modules, understanding module structure, or working with dependency injection.
---

# Weedbox Module Development

This skill helps develop modules using the weedbox framework with Uber FX dependency injection.

## Framework Overview

Plasma backend uses:
- **weedbox/weedbox** - Base module framework
- **weedbox/common-modules** - Common reusable modules
- **go.uber.org/fx** - Dependency injection

## Module Structure

```
pkg/mymodule/
├── module.go           # Module definition and FX setup
├── <feature>.go        # Business logic (named by feature, e.g., job.go, schedule.go)
├── models/             # Data models subdirectory
│   ├── model1.go
│   └── model2.go
├── errors.go           # Error definitions
├── config.go           # Configuration structs
├── codec.go            # Serialization helpers
├── utils.go            # Utility functions
└── *_test.go           # Tests
```

**Note**: Business logic files are named by their feature/responsibility (e.g., `job.go`, `blueprint.go`, `schedule.go`), not generically as `core.go`.

## Two Methods for Creating Modules

### Method 1: Manual FX Module

Full control over FX dependency injection with manual lifecycle hook registration.

```go
type Params struct {
    fx.In
    Logger   *zap.Logger
    Database database.DatabaseConnector
}

type MyModule struct {
    params Params
    logger *zap.Logger
    scope  string
}

func Module(scope string) fx.Option {
    return fx.Module(scope,
        fx.Provide(func(p Params) *MyModule { ... }),
        fx.Invoke(func(lc fx.Lifecycle, m *MyModule) {
            lc.Append(fx.Hook{OnStart: m.onStart, OnStop: m.onStop})
        }),
    )
}
```

**When to use**: Complex FX annotations, fine-grained control, multiple lifecycle hooks.

For complete documentation, see [./references/METHOD1_MANUAL_FX.md](./references/METHOD1_MANUAL_FX.md).

---

### Method 2: Weedbox Generic Module (Recommended)

Cleaner code using `weedbox.Module[P]` generics with built-in lifecycle management.

```go
type Params struct {
    weedbox.Params
    Database database.DatabaseConnector
}

type MyModule struct {
    weedbox.Module[*Params]
}

func Module(scope string) fx.Option {
    m := new(MyModule)
    return fx.Module(scope,
        fx.Supply(fx.Annotated{Name: scope, Target: m}),
        fx.Invoke(func(p Params) { weedbox.InitModule(scope, &p, m) }),
    )
}

func (m *MyModule) OnStart(ctx context.Context) error { ... }
func (m *MyModule) OnStop(ctx context.Context) error { ... }
func (m *MyModule) InitDefaultConfigs() { ... }
```

**When to use**: New modules, simple dependencies, less boilerplate.

For complete documentation, see [./references/METHOD2_WEEDBOX_GENERIC.md](./references/METHOD2_WEEDBOX_GENERIC.md).

---

## Quick Comparison

| Aspect | Method 1 (Manual) | Method 2 (Weedbox Generic) |
|--------|-------------------|---------------------------|
| Base struct | Custom struct | `weedbox.Module[*Params]` |
| Params | `fx.In` struct | Embed `weedbox.Params` |
| Logger | `m.logger` (manual) | `m.Logger()` (built-in) |
| Config path | `m.getConfigPath()` | `m.GetConfigPath()` |
| Lifecycle | `lc.Append(fx.Hook{})` | `OnStart()` / `OnStop()` |
| Injection | No `name` tag needed | **Requires `name` tag** |

## Injection Rules

```go
type Params struct {
    weedbox.Params

    // Method 2 modules - REQUIRE `name` tag
    PkgManager *pkg_manager.PkgManager `name:"pkg_manager"`

    // Method 1 modules - NO `name` tag needed
    ViewManager *view_manager.ViewManager

    // Common modules - NO `name` tag
    Database database.DatabaseConnector
}
```

## Common Modules Reference

Available from `github.com/weedbox/common-modules`:

| Module | Import | Purpose |
|--------|--------|---------|
| `logger` | `logger.Module()` | Zap logging |
| `http_server` | `http_server.Module("scope")` | Gin HTTP server |
| `sqlite_connector` | `sqlite_connector.Module("scope")` | SQLite connection |
| `nats_connector` | `nats_connector.Module("scope")` | NATS client |
| `database` | `database.DatabaseConnector` | Database interface |
| `daemon` | `daemon.Module("scope")` | Daemon lifecycle |
| `healthcheck_apis` | `healthcheck_apis.Module("scope")` | Health endpoints |
| `configs` | `configs.NewConfig("PREFIX")` | Configuration |

## Configuration Management

### Using Viper

```go
// Set defaults
viper.SetDefault(m.GetConfigPath("timeout"), "30s")

// Read values
timeout := viper.GetDuration(m.GetConfigPath("timeout"))
```

### Config File Format

```toml
[mymodule]
timeout = "30s"
enabled = true
```

### Environment Variables

```bash
export PLASMA_MYMODULE_TIMEOUT=60s
```

## Database Models

Models are placed in a `models/` subdirectory:

```go
// pkg/mymodule/models/base.go
package models

type BaseModel struct {
    ID        string    `gorm:"type:varchar(36);primary_key" json:"id"`
    CreatedAt time.Time `gorm:"index" json:"created_at"`
    UpdatedAt time.Time `gorm:"index" json:"updated_at"`
}

// pkg/mymodule/models/mymodel.go
package models

type MyModel struct {
    BaseModel
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    WorkspaceID string `gorm:"type:varchar(36);not null;index" json:"workspace_id"`
}
```

Import in module: `"github.com/BrobridgeOrg/plasma-backend/pkg/mymodule/models"`

## Error Handling

```go
const (
    ErrCodeGeneral     = 400000
    ErrCodeNotFound    = 400001
    ErrCodeInvalidData = 400002
)
```

## Event Handling

### Publishing Events

```go
func (m *MyModule) publishEvent(ctx context.Context, subject string, data interface{}) error {
    payload, _ := json.Marshal(data)
    return m.Params().EventManager.Publish(ctx, subject, payload)
}
```

### Subscribing to Events

```go
consumer := event_manager.NewWorkQueueConsumer(
    m.Params().NATSConnector,
    "mymodule.events",
    "mymodule-consumer",
    m.handleEvent,
)
```

## Registering Module

```go
// modules.go
func loadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        mymodule.Module("mymodule"),
    }
    return modules, nil
}
```

## Best Practices

1. **Scope naming**: Use consistent naming (`mymodule`, `mymodule_apis`)
2. **Logging**: Use structured logging with `zap.String()`, `zap.Error()`
3. **Configuration**: Always set defaults, use scoped config paths
4. **Lifecycle**: Initialize in `OnStart`, cleanup in `OnStop`
5. **Dependencies**: Declare all dependencies in `Params` struct
6. **Errors**: Use module-specific error code ranges

## Module Checklist

- [ ] Choose Method 1 or Method 2 (recommended)
- [ ] Create `module.go` with `Module()` function
- [ ] Define `Params` struct with dependencies
- [ ] Implement lifecycle hooks
- [ ] Set default configurations with Viper
- [ ] Define data models if needed
- [ ] Add error codes in appropriate range
- [ ] Register in `modules.go`
- [ ] Add tests

## Related Documentation

- [./references/METHOD1_MANUAL_FX.md](./references/METHOD1_MANUAL_FX.md) - Manual FX module details
- [./references/METHOD2_WEEDBOX_GENERIC.md](./references/METHOD2_WEEDBOX_GENERIC.md) - Weedbox generic module details
