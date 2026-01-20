# Method 1: Manual FX Module

This approach gives you full control over FX dependency injection with manual lifecycle hook registration.

## When to Use

- Module needs complex FX annotations
- You need fine-grained control over initialization order
- Migrating existing code that doesn't fit the weedbox pattern
- Module requires multiple lifecycle hooks with different priorities

## Complete Template

```go
package mymodule

import (
    "context"
    "fmt"

    "github.com/spf13/viper"
    "github.com/weedbox/common-modules/database"
    "go.uber.org/fx"
    "go.uber.org/zap"
)

// Module configuration defaults
const (
    DefaultTimeout = "30s"
    DefaultRetries = 3
)

// MyModule is the main module struct
type MyModule struct {
    params  Params
    logger  *zap.Logger
    scope   string
    configs *ModuleConfig
}

// Params defines module dependencies (fx.In)
type Params struct {
    fx.In

    Logger   *zap.Logger
    Database database.DatabaseConnector
    // Add other dependencies...
}

// Module creates the FX module
func Module(scope string) fx.Option {
    return fx.Module(
        scope,
        fx.Provide(func(p Params) *MyModule {
            m := &MyModule{
                params: p,
                logger: p.Logger.Named(scope),
                scope:  scope,
            }
            m.initDefaultConfigs()
            return m
        }),
        fx.Invoke(func(lc fx.Lifecycle, m *MyModule) {
            lc.Append(fx.Hook{
                OnStart: m.onStart,
                OnStop:  m.onStop,
            })
        }),
    )
}

// getConfigPath returns scoped config key
func (m *MyModule) getConfigPath(key string) string {
    return fmt.Sprintf("%s.%s", m.scope, key)
}

// initDefaultConfigs sets default configuration values
func (m *MyModule) initDefaultConfigs() {
    viper.SetDefault(m.getConfigPath("timeout"), DefaultTimeout)
    viper.SetDefault(m.getConfigPath("retries"), DefaultRetries)
}

// onStart initializes the module
func (m *MyModule) onStart(ctx context.Context) error {
    m.logger.Info("Starting MyModule")

    // Initialize configuration
    m.configs = m.loadConfigs()

    // Auto-migrate database models (from models/ subdirectory)
    db := m.params.Database.GetDB()
    if err := db.AutoMigrate(&models.MyModel{}); err != nil {
        return fmt.Errorf("failed to migrate: %w", err)
    }

    return nil
}

// onStop cleans up the module
func (m *MyModule) onStop(ctx context.Context) error {
    m.logger.Info("Stopping MyModule")
    return nil
}
```

## Key Components

### Params Struct with fx.In

```go
type Params struct {
    fx.In

    // Required dependencies
    Logger   *zap.Logger
    Database database.DatabaseConnector

    // Optional dependencies
    Optional *OtherModule `optional:"true"`

    // Named dependencies
    Config map[string]interface{} `name:"config"`
}
```

### Manual Lifecycle Hooks

```go
fx.Invoke(func(lc fx.Lifecycle, m *MyModule) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            // Initialization code
            return nil
        },
        OnStop: func(ctx context.Context) error {
            // Cleanup code
            return nil
        },
    })
})
```

### Multiple Lifecycle Hooks

You can add multiple hooks with different priorities:

```go
fx.Invoke(func(lc fx.Lifecycle, m *MyModule) {
    // First hook - runs first on start, last on stop
    lc.Append(fx.Hook{
        OnStart: m.initDatabase,
        OnStop:  m.closeDatabase,
    })

    // Second hook - runs after first on start, before first on stop
    lc.Append(fx.Hook{
        OnStart: m.startConsumers,
        OnStop:  m.stopConsumers,
    })
})
```

## Providing Dependencies

```go
// In Module() function
fx.Provide(func(p Params) *MyModule {
    return &MyModule{params: p}
})
```

## Injecting This Module

When other modules need to inject a Method 1 module, no `name` tag is required:

```go
type OtherParams struct {
    fx.In

    // Method 1 modules - no name tag needed
    MyModule *mymodule.MyModule
}
```

## Configuration Management

### Setting Defaults

```go
func (m *MyModule) initDefaultConfigs() {
    viper.SetDefault(m.getConfigPath("timeout"), "30s")
    viper.SetDefault(m.getConfigPath("enabled"), true)
    viper.SetDefault(m.getConfigPath("port"), 8080)
}
```

### Reading Configuration

```go
func (m *MyModule) loadConfigs() *ModuleConfig {
    return &ModuleConfig{
        Timeout: viper.GetDuration(m.getConfigPath("timeout")),
        Enabled: viper.GetBool(m.getConfigPath("enabled")),
        Port:    viper.GetInt(m.getConfigPath("port")),
    }
}
```

## Example: Real-World Module

See `pkg/view_manager/module.go` for a complete example of Method 1 implementation.

```go
// pkg/view_manager/module.go
func Module(scope string) fx.Option {
    return fx.Module(
        scope,
        fx.Provide(func(p Params) *ViewManager {
            vm := &ViewManager{
                params: p,
                logger: p.Logger.Named("view_manager"),
                scope:  scope,
            }
            vm.initDefaultConfigs()
            return vm
        }),
        fx.Invoke(func(lc fx.Lifecycle, vm *ViewManager) {
            lc.Append(
                fx.Hook{
                    OnStart: vm.onStart,
                    OnStop:  vm.onStop,
                },
            )
        }),
    )
}
```

## Checklist

- [ ] Create struct with `params`, `logger`, `scope` fields
- [ ] Define `Params` struct with `fx.In`
- [ ] Implement `Module()` function with `fx.Provide` and `fx.Invoke`
- [ ] Implement `getConfigPath()` helper
- [ ] Implement `initDefaultConfigs()` for Viper defaults
- [ ] Implement `onStart()` and `onStop()` lifecycle hooks
- [ ] Register in `modules.go`
