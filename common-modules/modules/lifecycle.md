# lifecycle

Application-level PostStart / PreStop phases: run hooks after **all** modules' `OnStart` have completed, and before **any** module's `OnStop` runs. Any module, anywhere in the module list, can register hooks — only `lifecycle.Module` itself must be placed near the end of the module list.

## Installation

```bash
go get github.com/weedbox/common-modules/lifecycle
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/lifecycle"

// IMPORTANT: lifecycle goes in afterModules(), AFTER all business modules
// and BEFORE daemon, so readiness flips only after PostStart completes.
func afterModules() ([]fx.Option, error) {
    return []fx.Option{
        lifecycle.Module("lifecycle"),
        daemon.Module("daemon"),
    }, nil
}
```

Resulting order:

- **Startup:** business modules `OnStart` → PostStart hooks → daemon marks ready (traffic starts)
- **Shutdown:** daemon un-marks ready (traffic drains) → PreStop hooks → business modules `OnStop`

## How It Works

fx executes `OnStart` hooks in append order and `OnStop` hooks in reverse append order, and module invokes run in module registration order. Because the lifecycle module is registered after every business module, its own fx hook is appended last: its `OnStart` (running the PostStart hooks) fires last, and its `OnStop` (running the PreStop hooks) fires first.

## Registering Hooks from a Module

Inject `*lifecycle.Manager` **without** a `name` tag (common-module rule). Name the field `AppLifecycle` — a field named `Lifecycle` would shadow the `fx.Lifecycle` promoted from the embedded `weedbox.Params`. Hooks may be registered from a constructor, an `fx.Invoke`, or another module's `OnStart`.

```go
package mymodule

import (
    "context"

    "github.com/weedbox/common-modules/lifecycle"
    "github.com/weedbox/weedbox"
)

type Params struct {
    weedbox.Params

    AppLifecycle *lifecycle.Manager
}

type MyModule struct {
    weedbox.Module[*Params]
}

func (m *MyModule) OnStart(ctx context.Context) error {

    // Runs only after ALL modules have completed OnStart
    m.Params().AppLifecycle.PostStart("mymodule.warmup", func(ctx context.Context) error {
        return m.warmupCache(ctx)
    })

    // Runs at shutdown, before ANY module's OnStop
    m.Params().AppLifecycle.PreStop("mymodule.drain", func(ctx context.Context) error {
        return m.drain(ctx)
    })

    return nil
}
```

## API Methods

```go
// Register a hook to run after all modules have started.
// Returns ErrPhaseCompleted if the post-start phase already ran.
manager.PostStart(name string, fn lifecycle.HookFunc) error

// Register a hook to run at shutdown, before all modules stop.
// Returns ErrPhaseCompleted if the pre-stop phase already ran.
manager.PreStop(name string, fn lifecycle.HookFunc) error

type HookFunc func(ctx context.Context) error
```

## Semantics

| Phase | When | Order | Failure behavior |
|-------|------|-------|------------------|
| `PostStart` | After all modules' `OnStart` completed | Registration order, sequential | First error aborts startup; fx rolls back already-started modules |
| `PreStop` | At shutdown, before all modules' `OnStop` | Reverse registration order | Logged; remaining hooks still run; errors joined into returned error |

- Hook names are for logging only; duplicates are allowed.
- A `nil` hook function is ignored with a warning.
- A panic inside a hook is recovered and converted into an error.
- If startup fails, fx skips the lifecycle module's `OnStop`, so **PreStop hooks do not run on failed startup** (standard fx rollback semantics).

## Context and Timeout

The `ctx` passed to PostStart hooks is fx's start context, forwarded unchanged:

- With `app.Run()` (the standard weedbox path), fx creates the start context with `fx.StartTimeout` (default **15 seconds**). This deadline is shared by the **entire** startup — all modules' `OnStart` plus all PostStart hooks combined, not 15s per hook.
- With a manual `app.Start(ctx)` (e.g. tests), fx does **not** apply `StartTimeout`; the deadline is whatever the caller's ctx carries.
- A hook that ignores ctx and blocks past the deadline leaves `Start` returning `DeadlineExceeded` without rollback — always propagate ctx into downstream calls.

For longer startup work:

```go
// Raise the overall start timeout (add to the modules list)
fx.StartTimeout(time.Minute)

// Or detach long work; PostStart only acts as the trigger point
m.Params().AppLifecycle.PostStart("warmup", func(ctx context.Context) error {
    go m.warmupSlowly(context.WithoutCancel(ctx)) // does not block startup
    return nil
})
```

## Caveats

- **Multi-instance deployments:** every replica runs the PostStart hooks. One-time jobs (migrations, seeding) must be idempotent or guarded by leader election / a distributed lock.
- **Root-level `fx.Invoke`:** fx runs module invokes before root-level (non-module) invokes. An `fx.Hook` appended by a bare `fx.Invoke(...)` in `modules.go` is appended after the lifecycle module's hook, so its `OnStart` runs *after* the PostStart phase. Keep glue code that appends hooks inside an `fx.Module`, or register it as a PostStart hook instead.

## Configuration

This module has no configuration parameters.
