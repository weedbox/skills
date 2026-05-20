# Method 3: Interface Module (`fxmodule.InterfaceModule`)

This approach is for **connector-style modules** — modules that implement
a shared interface (database connectors, cache backends, message brokers,
greeters, etc.) and may be loaded side-by-side with other implementations
of the same interface in a single `fx.App`.

Backed by `github.com/weedbox/weedbox/fxmodule`. The headline helper is
`fxmodule.InterfaceModule[I](scope, ctor)`.

## When to Use

- The module is one implementation of an interface that other
  implementations may swap in for (`database.DatabaseConnector`,
  `cache.Cache`, `MessageBroker`, etc.).
- Multiple implementations of the same interface need to be loadable
  side-by-side in the same `fx.App` and addressable by scope.
- You still want existing single-load consumers (those that inject the
  interface without a `name` tag) to keep working unchanged.

**Don't use** for plain business-logic modules that expose a concrete
struct (use Method 1 or Method 2 for those). `InterfaceModule` is
specifically for the "one interface, swappable implementations" pattern.

## What It Does

`fxmodule.InterfaceModule[I](scope, ctor)`:

1. Registers `ctor` with `name:"<scope>"` returning `I`.
2. Adds an internal `fx.Invoke` that depends on the named result. This
   forces materialization, so lifecycle hooks wired inside `ctor` always
   run — even if no other consumer references the named instance.
3. On the **first** call to `InterfaceModule[I]` across the process,
   also aliases the same instance to the unnamed default of `I` (via
   `Alias`). Subsequent calls only contribute their named instance, so
   there is no duplicate-provider conflict.

The result: any consumer that injects `I` without a `name` tag receives
the first-loaded implementation; consumers that inject `I` with
`name:"<scope>"` get the specific one.

## Complete Template

```go
package my_connector

import (
    "context"

    "github.com/weedbox/common-modules/database"
    "github.com/weedbox/weedbox/fxmodule"
    "go.uber.org/fx"
    "go.uber.org/zap"
)

type Params struct {
    fx.In

    Lifecycle fx.Lifecycle
    Logger    *zap.Logger
}

type MyConnector struct {
    logger *zap.Logger
    db     *somelib.DB
}

func (c *MyConnector) GetDB() *gorm.DB { /* ... */ }

func Module(scope string) fx.Option {
    return fxmodule.InterfaceModule[database.DatabaseConnector](
        scope,
        func(p Params) database.DatabaseConnector {
            c := &MyConnector{logger: p.Logger.Named(scope)}
            p.Lifecycle.Append(fx.Hook{
                OnStart: c.onStart,
                OnStop:  c.onStop,
            })
            return c
        },
    )
}

func (c *MyConnector) onStart(ctx context.Context) error {
    c.logger.Info("connector starting")
    // open the connection, etc.
    return nil
}

func (c *MyConnector) onStop(ctx context.Context) error {
    c.logger.Info("connector stopping")
    return nil
}
```

The application composition root still writes the usual
`my_connector.Module("my_connector")` — `InterfaceModule` is
connector-author scaffolding, not something the application needs to
wire by hand.

## Consuming Implementations

### Single-load (backwards-compat path)

```go
fx.New(
    sqlite_connector.Module("database"),
    fx.Invoke(func(p struct {
        fx.In
        DB database.DatabaseConnector  // no name tag — works as before
    }) {
        // ...
    }),
)
```

### Multi-load (named injection)

```go
fx.New(
    sqlite_connector.Module("cache"),     // claims unnamed default (loaded first)
    postgres_connector.Module("main"),
    fx.Invoke(func(p struct {
        fx.In
        Default database.DatabaseConnector                       // == cache
        Cache   database.DatabaseConnector `name:"cache"`
        Main    database.DatabaseConnector `name:"main"`
    }) {
        // ...
    }),
)
```

If load order is brittle in your app, **always inject by named tag** and
ignore the unnamed default. The unnamed default exists for the
single-load / backwards-compat case, not as a routing mechanism.

## Test Caveat: `ResetClaim` Between `fx.App`s

The "first call wins" claim on the unnamed default uses **process-level**
state. Tests that build more than one `fx.App` in the same process must
reset that claim between apps, otherwise the second app cannot register
an unnamed default and any consumer asking for `I` without a name tag
will fail to wire:

```go
import "github.com/weedbox/weedbox/fxmodule"

func TestSomething(t *testing.T) {
    fxmodule.ResetClaim[database.DatabaseConnector]()
    t.Cleanup(func() {
        fxmodule.ResetClaim[database.DatabaseConnector]()
    })

    app := fx.New(
        sqlite_connector.Module("database"),
        // ...
    )
    // ...
}
```

## Lower-Level Primitives

`InterfaceModule` is composed from smaller helpers, also exported for
custom wiring:

| Helper                | Purpose                                                                                                                  |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------|
| `Provide(name, ctor)` | Like `fx.Provide`, but tags the result `name:"<name>"` when `name` is non-empty. Empty `name` falls back to plain provide. |
| `Invoke(name, fn)`    | Like `fx.Invoke`, but tags the function's first parameter `name:"<name>"` when `name` is non-empty.                       |
| `Alias[T](name)`      | Re-exports a `name:"<name>"`-tagged `T` as the unnamed default.                                                          |
| `ClaimDefault[T]()`   | Atomically claims the unnamed default slot for `T`. Returns `true` on the first call, `false` afterwards.                |
| `ResetClaim[T]()`     | Clears any prior `ClaimDefault[T]`. Test-only.                                                                            |

Reach for these directly when you need behavior `InterfaceModule`
doesn't cover — e.g. claiming the default conditionally, or registering
a concrete type rather than an interface.

## Real-World References

`common-modules` ships two connectors built on top of `InterfaceModule`:

- `sqlite_connector.Module(scope)`
- `postgres_connector.Module(scope)`

Both can be loaded into the same `fx.App` and injected via named tags
or the unnamed default. See:

- [common-modules database.md](../../common-modules/modules/database.md#loading-multiple-connectors)
- [common-modules sqlite_connector.md](../../common-modules/modules/sqlite_connector.md#loading-multiple-connectors)
- [common-modules postgres_connector.md](../../common-modules/modules/postgres_connector.md#loading-multiple-connectors)

## Comparison with Methods 1 / 2

| Aspect                  | Method 1 (Manual FX)              | Method 2 (Weedbox Generic)         | Method 3 (InterfaceModule)                       |
|-------------------------|-----------------------------------|------------------------------------|--------------------------------------------------|
| Module shape            | Concrete struct                   | Concrete struct + base class       | Constructor returning an **interface**           |
| Side-by-side loading    | Not supported (one provider only) | Not supported (one provider only)  | **Supported** — multiple impls in one app        |
| Injection (single-load) | No `name` tag                     | Requires `name` tag                | No `name` tag (unnamed default)                  |
| Injection (multi-load)  | N/A                               | N/A                                | `name:"<scope>"` tag on each consumer            |
| Lifecycle wiring        | Manual `lc.Append(...)`           | `OnStart` / `OnStop` methods       | Manual `lc.Append(...)` inside `ctor`            |
| Boilerplate             | Highest                           | Lowest                             | Low (one-line `Module` factory)                  |

## Checklist

- [ ] Confirm the module implements an **interface** (not a concrete-only struct)
- [ ] Define `Params` with `fx.In`, `Lifecycle`, `Logger`, and dependencies
- [ ] Implement the interface on the concrete struct
- [ ] Write `Module(scope)` as a one-liner calling `fxmodule.InterfaceModule[I](scope, ctor)`
- [ ] Wire `Lifecycle.Append(fx.Hook{...})` **inside** the constructor
- [ ] In tests that build multiple `fx.App`s in one process, call
      `fxmodule.ResetClaim[I]()` between apps
- [ ] Document multi-load usage in the module's README if the interface
      is expected to host multiple implementations
