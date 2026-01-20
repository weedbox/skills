# Method 2: Weedbox Generic Module (Recommended)

This approach uses `weedbox.Module[P]` generics for cleaner code with built-in lifecycle management.

**Reference Implementation**: `pkg/pkg_manager/module.go`

## When to Use

- Creating new modules from scratch
- Module has simple dependencies
- You want cleaner, less boilerplate code
- You prefer interface-based lifecycle hooks

## Complete Template

```go
package mymodule

import (
    "context"

    "github.com/spf13/viper"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"
)

const (
    ModuleName = "MyModule"
)

// Params embeds weedbox.Params for base dependencies (Logger, etc.)
type Params struct {
    weedbox.Params
    // Add custom dependencies here
    // Database database.DatabaseConnector
}

// MyModule embeds weedbox.Module with generic Params type
type MyModule struct {
    weedbox.Module[*Params]
}

// Module creates the FX module using weedbox.InitModule
func Module(scope string) fx.Option {
    m := new(MyModule)

    return fx.Module(
        scope,
        fx.Supply(
            fx.Annotated{
                Name:   scope,
                Target: m,
            },
        ),
        fx.Invoke(func(p Params) {
            weedbox.InitModule(scope, &p, m)
        }),
    )
}

// OnStart is called when the module starts (implements weedbox.ModuleInterface)
func (m *MyModule) OnStart(ctx context.Context) error {
    m.Logger().Info(ModuleName + " started")

    // Initialize your module here
    // Access params: m.Params()
    // Access logger: m.Logger()
    // Access config: m.GetConfigPath("key")

    return nil
}

// OnStop is called when the module stops (implements weedbox.ModuleInterface)
func (m *MyModule) OnStop(ctx context.Context) error {
    m.Logger().Info(ModuleName + " stopped")
    return nil
}

// InitDefaultConfigs sets default configuration values (implements weedbox.ModuleInterface)
func (m *MyModule) InitDefaultConfigs() {
    viper.SetDefault(m.GetConfigPath("timeout"), "30s")
    viper.SetDefault(m.GetConfigPath("enabled"), true)
}
```

## Key Components

### Params with weedbox.Params

```go
type Params struct {
    weedbox.Params  // Provides Logger and other base dependencies

    // Add custom dependencies
    Database     database.DatabaseConnector
    EventManager *event_manager.EventManager
}
```

### Module Struct with Generic

```go
type MyModule struct {
    weedbox.Module[*Params]  // Generic type parameter is pointer to Params
}
```

### Module Function with fx.Annotated

```go
func Module(scope string) fx.Option {
    m := new(MyModule)

    return fx.Module(
        scope,
        fx.Supply(
            fx.Annotated{
                Name:   scope,  // This name is used for injection
                Target: m,
            },
        ),
        fx.Invoke(func(p Params) {
            weedbox.InitModule(scope, &p, m)
        }),
    )
}
```

## Built-in Methods from weedbox.Module

```go
// Access the logger (already named with scope)
m.Logger() *zap.Logger

// Get scoped config path: "scope.key"
m.GetConfigPath(key string) string

// Access injected params
m.Params() *Params

// Get module scope name
m.Scope() string
```

## Lifecycle Interface Methods

Implement these methods to hook into module lifecycle:

```go
// Called when module starts
func (m *MyModule) OnStart(ctx context.Context) error {
    return nil
}

// Called when module stops
func (m *MyModule) OnStop(ctx context.Context) error {
    return nil
}

// Called during initialization to set Viper defaults
func (m *MyModule) InitDefaultConfigs() {
    viper.SetDefault(m.GetConfigPath("key"), "value")
}
```

## Injecting Method 2 Modules

**Important**: When injecting a Method 2 module into another module's Params, you **must** use the `name` tag matching the scope name used in `modules.go`.

This is required because Method 2 uses `fx.Annotated{Name: scope}` to provide a named dependency.

### Example: Injecting into Another Module

```go
package another_module

import (
    "github.com/BrobridgeOrg/plasma-backend/pkg/pkg_manager"
    "github.com/weedbox/weedbox"
)

type Params struct {
    weedbox.Params

    // IMPORTANT: Must use `name` tag matching the scope in modules.go
    PkgManager *pkg_manager.PkgManager `name:"pkg_manager"`
}

type AnotherModule struct {
    weedbox.Module[*Params]
}

func (m *AnotherModule) OnStart(ctx context.Context) error {
    // Access the injected PkgManager
    pm := m.Params().PkgManager
    err := pm.InstallPackage(pkgPath, targetPath)
    return err
}
```

### Injection Patterns Summary

```go
type Params struct {
    weedbox.Params

    // Method 2 modules - REQUIRE `name` tag
    PkgManager      *pkg_manager.PkgManager           `name:"pkg_manager"`
    AtomicManager   *atomic_manager.AtomicManager     `name:"atomic_manager"`

    // Method 1 modules - NO `name` tag needed
    ViewManager     *view_manager.ViewManager
    QueryManager    *query_manager.QueryManager

    // Common modules from weedbox/common-modules - NO `name` tag
    Database        database.DatabaseConnector
    HTTPServer      *http_server.HTTPServer
}
```

## Scope Names Reference

The scope name is defined when registering the module in `modules.go`:

```go
// modules.go
func loadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        pkg_manager.Module("pkg_manager"),           // scope = "pkg_manager"
        atomic_manager.Module("atomic_manager"),     // scope = "atomic_manager"
        // ...
    }
    return modules, nil
}
```

Use this scope name in the `name` tag when injecting.

## Example: With Database and Event Manager

```go
package mymodule

import (
    "context"

    "github.com/BrobridgeOrg/plasma-backend/pkg/event_manager"
    "github.com/spf13/viper"
    "github.com/weedbox/common-modules/database"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"
)

const ModuleName = "MyModule"

type Params struct {
    weedbox.Params
    Database     database.DatabaseConnector
    EventManager *event_manager.EventManager
}

type MyModule struct {
    weedbox.Module[*Params]
}

func Module(scope string) fx.Option {
    m := new(MyModule)
    return fx.Module(
        scope,
        fx.Supply(fx.Annotated{Name: scope, Target: m}),
        fx.Invoke(func(p Params) {
            weedbox.InitModule(scope, &p, m)
        }),
    )
}

func (m *MyModule) OnStart(ctx context.Context) error {
    m.Logger().Info(ModuleName + " started")

    // Access database through Params()
    // Models are in models/ subdirectory
    db := m.Params().Database.GetDB()
    if err := db.AutoMigrate(&models.MyModel{}); err != nil {
        return err
    }

    return nil
}

func (m *MyModule) OnStop(ctx context.Context) error {
    m.Logger().Info(ModuleName + " stopped")
    return nil
}

func (m *MyModule) InitDefaultConfigs() {
    viper.SetDefault(m.GetConfigPath("timeout"), "30s")
}
```

## Comparison with Method 1

| Aspect | Method 1 (Manual) | Method 2 (Weedbox Generic) |
|--------|-------------------|---------------------------|
| Base struct | Custom struct | `weedbox.Module[*Params]` |
| Params | `fx.In` struct | Embed `weedbox.Params` |
| Initialization | `fx.Provide` + `fx.Invoke` | `weedbox.InitModule()` |
| Logger access | `m.logger` (manual field) | `m.Logger()` (built-in method) |
| Config path | `m.getConfigPath()` (manual) | `m.GetConfigPath()` (built-in) |
| Lifecycle | `lc.Append(fx.Hook{})` | `OnStart()` / `OnStop()` methods |
| Injection | No `name` tag needed | Requires `name` tag |

## Checklist

- [ ] Create `Params` struct embedding `weedbox.Params`
- [ ] Create module struct embedding `weedbox.Module[*Params]`
- [ ] Implement `Module()` function with `fx.Supply` (annotated) and `fx.Invoke`
- [ ] Implement `OnStart()` lifecycle method
- [ ] Implement `OnStop()` lifecycle method
- [ ] Implement `InitDefaultConfigs()` for Viper defaults
- [ ] Register in `modules.go` with appropriate scope name
- [ ] When injecting, use `name` tag matching the scope
