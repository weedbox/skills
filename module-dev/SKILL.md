---
name: module-dev
description: |
  Priority: HIGH — Self-contained reference. This file and its reference docs contain ALL necessary
  patterns for module development. DO NOT fetch source code from GitHub.
  Weedbox module/package development guide with Uber Fx dependency injection.
  Use when: creating weedbox module, developing weedbox package, building custom module,
  implementing dependency injection, managing module lifecycle (OnStart/OnStop),
  configuring module with Viper, injecting database/logger/NATS/Redis dependencies,
  building connector-style modules with swappable interface implementations.
  Covers: Module() function, Params struct, fx.In/fx.Out, weedbox.Module generic,
  fxmodule.InterfaceModule helper for swappable interfaces,
  configuration management, error handling, event publishing.
  Keywords: weedbox module, weedbox package, uber fx, dependency injection, module development,
  OnStart, OnStop, Params, fx.Invoke, fx.Provide, lifecycle hooks,
  create module, new module, custom module, Go module, weedbox generic, module skills,
  fxmodule, InterfaceModule, connector module, swappable implementation, named instance,
  multi-instance, stateless module, idempotent handler, horizontal scaling, replicas.
---

# Weedbox Module Development

This skill helps develop modules using the weedbox framework with Uber FX dependency injection.

## Contents

- [Critical Rules](#️-critical-rules) / [Common Mistakes](#️-common-mistakes-do-not) / [Pre-Development Checklist](#-pre-development-checklist)
- [Module Structure](#module-structure)
- [Three Methods for Creating Modules](#three-methods-for-creating-modules) and [Quick Comparison](#quick-comparison)
- [Injection Rules](#injection-rules)
- [User Modules Reference](#user-modules-reference) / [Common Modules Reference](#common-modules-reference)
- [Configuration Management](#configuration-management)
- [Database Models](#database-models)
- [Error Handling](#error-handling) / [Event Handling](#event-handling)
- [Multi-Instance Safety](#multi-instance-safety)
- [Registering Module](#registering-module) / [Best Practices](#best-practices) / [Module Checklist](#module-checklist)
- [Module Skills](#module-skills)
- [Related Documentation](#related-documentation)

## Framework Overview

Plasma backend uses:
- **weedbox/weedbox** - Base module framework
- **weedbox/common-modules** - Common reusable modules
- **go.uber.org/fx** - Dependency injection

## ⚠️ Critical Rules

### Source of Truth — Do NOT Fetch from GitHub

This file and its reference docs (`references/*.md`) are the **complete reference** for weedbox module development. They contain all necessary patterns, code examples, and best practices. **DO NOT browse GitHub repositories** to look up weedbox module source code.

## ⚠️ Common Mistakes (DO NOT)

| Wrong | Correct | Reason |
|-------|---------|--------|
| `m.GetParams()` | `m.Params()` | weedbox.Module uses `Params()` method |
| `*sqlite_connector.SQLiteConnector` | `database.DatabaseConnector` | Use interface types, not concrete implementations |
| `*postgres_connector.PostgresConnector` | `database.DatabaseConnector` | Same as above, always use interface for database |
| `*http_server.HTTPServer \`name:"..."\`` | `*http_server.HTTPServer` | common-modules do NOT need name tag |
| `gorm:"uniqueIndex:idx_foo_bar"` | `gorm:"uniqueIndex:idx_tablename_foo_bar"` | SQLite index names are GLOBAL - must include table name prefix |

## 📋 Pre-Development Checklist

Before creating a new module, you **MUST**:

1. **Read existing modules** - Find an existing module in the project (e.g., `customer/module.go`) and read the complete code as a template
2. **Verify Params types** - Dependencies must use interface types (e.g., `database.DatabaseConnector`), not concrete implementations
3. **Verify method names** - weedbox modules use `m.Params()` to access parameters
4. **Verify name tag rules** - Method 2 modules require `name` tag, common-modules do NOT
5. **Verify index naming** - When creating GORM models with named indexes, use `idx_<tablename>_<columns>` format (SQLite index names are global)

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

## Three Methods for Creating Modules

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

### Method 2: Weedbox Generic Module (Recommended for business modules)

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

**When to use**: New business-logic modules, simple dependencies, less boilerplate.

For complete documentation, see [./references/METHOD2_WEEDBOX_GENERIC.md](./references/METHOD2_WEEDBOX_GENERIC.md).

---

### Method 3: Interface Module (`fxmodule.InterfaceModule`) — for swappable implementations

For **connector-style** modules — those that implement a shared interface (database connectors, cache backends, message brokers, etc.) and may be loaded side-by-side with other implementations of the same interface.

```go
import (
    "github.com/weedbox/common-modules/database"
    "github.com/weedbox/weedbox/fxmodule"
)

type Params struct {
    fx.In
    Lifecycle fx.Lifecycle
    Logger    *zap.Logger
}

func Module(scope string) fx.Option {
    return fxmodule.InterfaceModule[database.DatabaseConnector](
        scope,
        func(p Params) database.DatabaseConnector {
            c := &MyConnector{ /* ... */ }
            p.Lifecycle.Append(fx.Hook{OnStart: c.onStart, OnStop: c.onStop})
            return c
        },
    )
}
```

`InterfaceModule[I]` registers the constructor as `name:"<scope>"` returning `I`, and the **first** call across the process also exposes the same instance as the unnamed default of `I`. Multiple implementations can be loaded into the same `fx.App` — consumers disambiguate via `name:"<scope>"`, and existing single-load callers that inject `I` without a tag keep working unchanged.

**When to use**: The module is one implementation of a swappable interface (`database.DatabaseConnector`, `cache.Cache`, etc.); multiple implementations may need to coexist in the same app.

**Don't use** for plain business-logic modules that expose a concrete struct — use Method 1 or Method 2 for those.

For complete documentation, see [./references/METHOD3_INTERFACE_MODULE.md](./references/METHOD3_INTERFACE_MODULE.md).

---

## Quick Comparison

| Aspect                  | Method 1 (Manual FX)          | Method 2 (Weedbox Generic)         | Method 3 (InterfaceModule)                |
|-------------------------|-------------------------------|------------------------------------|-------------------------------------------|
| Module shape            | Concrete struct               | Concrete struct + base class       | Constructor returning an **interface**    |
| Base struct             | Custom struct                 | `weedbox.Module[*Params]`          | None (constructor function)               |
| Params                  | `fx.In` struct                | Embed `weedbox.Params`             | `fx.In` struct                            |
| Logger                  | `m.logger` (manual)           | `m.Logger()` (built-in)            | Manual (in the constructor)               |
| Config path             | `m.getConfigPath()`           | `m.GetConfigPath()`                | Manual                                    |
| Lifecycle               | `lc.Append(fx.Hook{})`        | `OnStart()` / `OnStop()`           | `lc.Append(fx.Hook{})` inside `ctor`      |
| Side-by-side loading    | Not supported                 | Not supported                      | **Supported** — multiple impls in one app |
| Injection (single-load) | No `name` tag                 | Requires `name` tag                | No `name` tag (unnamed default)           |
| Injection (multi-load)  | N/A                           | N/A                                | `name:"<scope>"` on each consumer         |

## Injection Rules

```go
type Params struct {
    weedbox.Params

    // Method 2 modules - REQUIRE `name` tag
    PkgManager *pkg_manager.PkgManager `name:"pkg_manager"`

    // User modules - REQUIRE `name` tag (all use Method 2)
    User *user.UserManager `name:"user"`
    RBAC *rbac.RBACManager `name:"rbac"`
    Auth *auth.AuthManager `name:"auth"`

    // Method 1 modules - NO `name` tag needed
    ViewManager *view_manager.ViewManager

    // Common modules (single-load) - NO `name` tag
    Database database.DatabaseConnector
}
```

### Multi-load exception (Method 3 / `InterfaceModule`)

When two implementations of the same interface are loaded into the same `fx.App` (e.g. `sqlite_connector.Module("cache")` + `postgres_connector.Module("main")`), consumers MUST disambiguate via the `name` tag:

```go
type Params struct {
    fx.In

    Cache database.DatabaseConnector `name:"cache"` // sqlite
    Main  database.DatabaseConnector `name:"main"`  // postgres
}
```

The first connector loaded in the process also claims the unnamed default slot, so existing code that injects the interface without a tag continues to work — but if load order is brittle, always inject by named tag.

**Test caveat**: tests building multiple `fx.App`s in one process must call `fxmodule.ResetClaim[I]()` between apps. See [METHOD3 reference](./references/METHOD3_INTERFACE_MODULE.md#test-caveat-resetclaim-between-fxapps) for details.

## User Modules Reference

Available from `github.com/weedbox/user-modules`:

| Module | Import | Purpose |
|--------|--------|---------|
| `user` | `user.Module("user")` | User CRUD, bcrypt, UUID v7 |
| `rbac` | `rbac.Module("rbac")` | RBAC with privy |
| `auth` | `auth.Module("auth")` | JWT auth, middleware |
| `permissions` | (plain package) | Permission definitions |
| `user_apis` | `user_apis.Module("user_apis")` | User REST endpoints |
| `auth_apis` | `auth_apis.Module("auth_apis")` | Auth REST endpoints |
| `role_apis` | `role_apis.Module("role_apis")` | Role/resource REST endpoints |
| `http_token_validator` | `http_token_validator.Module("scope")` | Global JWT middleware |

See [user-modules skill](../user-modules/SKILL.md) for detailed documentation.

## Common Modules Reference

Available from `github.com/weedbox/common-modules`:

| Module | Import | Purpose |
|--------|--------|---------|
| `logger` | `logger.Module()` | Zap logging |
| `http_server` | `http_server.Module("scope")` | Gin HTTP server |
| `sqlite_connector` | `sqlite_connector.Module("scope")` | SQLite connection |
| `postgres_connector` | `postgres_connector.Module("scope")` | PostgreSQL connection |
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

Models are placed in a `models/` subdirectory of the module (e.g., `pkg/mymodule/models/`).

**Load-bearing rule**: SQLite index names are **global** across all tables — always name explicit indexes `idx_<table>_<columns>` (e.g., `idx_task_status`), never generic names like `idx_status`. Unnamed indexes (`gorm:"index"`) are auto-named by GORM and safe.

For GORM index configuration, naming examples, common index patterns, and auto migration, see [./references/DATABASE_MODELS.md](./references/DATABASE_MODELS.md).

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

## Multi-Instance Safety

Assume every module runs as **N replicas** in production (see the [root skill rule](../SKILL.md#design-for-multi-instance-deployment-by-default)). When writing a module:

1. **No in-process source of truth** — locks, counters, dedup sets, and sessions live in the database, Redis, or NATS; process memory is at most a cache.
2. **No bare `time.Ticker` / `time.AfterFunc` for business schedules** — every replica would fire independently. Use the `scheduler` module; its `postgres` and `nats` modes deliver each tick to exactly one replica.
3. **Distribute background work via `workqueue.WorkQueue`** — competing consumers across replicas — instead of a goroutine polling a table that every replica would race on.
4. **Write idempotent handlers** — scheduler and workqueue deliveries are at-least-once; a redelivered task must be harmless (guard with a unique key or a state check in the database).
5. **In-memory caches need an invalidation story** — a write on replica A must not leave stale reads on replica B indefinitely; prefer `redis_connector` or short TTLs.

Per-instance concerns (HTTP handlers, connection pools, logging, health status) need no coordination.

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
7. **Multi-instance**: Assume N replicas — see [Multi-Instance Safety](#multi-instance-safety)

## Module Checklist

- [ ] Choose a method: Method 2 (default for business modules), Method 1 (manual FX control), or Method 3 (swappable interface/connector)
- [ ] Create `module.go` with `Module()` function
- [ ] Define `Params` struct with dependencies
- [ ] Implement lifecycle hooks
- [ ] Set default configurations with Viper
- [ ] Define data models if needed
- [ ] Add error codes in appropriate range
- [ ] Verify multi-instance safety: no in-process coordination state, no bare timers for business schedules, idempotent handlers
- [ ] Register in `modules.go`
- [ ] Add tests

## Module Skills

Module Skills are `.skills/*.md` documentation files placed inside each module (e.g., `pkg/mymodule/.skills/mymodule-development.md`). Create one after completing a module, especially for complex modules or API modules that need an endpoint reference.

For directory structure, naming conventions, content templates, and checklists, see [./references/MODULE_SKILLS.md](./references/MODULE_SKILLS.md).

## Related Documentation

- [./references/METHOD1_MANUAL_FX.md](./references/METHOD1_MANUAL_FX.md) - Manual FX module details
- [./references/METHOD2_WEEDBOX_GENERIC.md](./references/METHOD2_WEEDBOX_GENERIC.md) - Weedbox generic module details
- [./references/METHOD3_INTERFACE_MODULE.md](./references/METHOD3_INTERFACE_MODULE.md) - Interface/connector module details
- [./references/DATABASE_MODELS.md](./references/DATABASE_MODELS.md) - GORM models, indexes, and migration
- [./references/MODULE_SKILLS.md](./references/MODULE_SKILLS.md) - Module skills guide
- [../user-modules/SKILL.md](../user-modules/SKILL.md) - User management, auth, and RBAC modules
