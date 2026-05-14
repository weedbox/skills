# scheduler

Job scheduler module with two interchangeable backends — `gorm` (single-node,
in-memory timer backed by `*gorm.DB`) and `nats` (NATS 2.12+ JetStream
Scheduled Message Delivery, KV-persisted). The public API is identical in
both modes; only `scheduler.mode` in config changes.

Built on top of `github.com/Weedbox/scheduler` (imported as `libsched`) and
wired into Uber Fx.

## Installation

```bash
go get github.com/weedbox/common-modules/scheduler
```

## Modes Overview

| Mode   | Backend                                         | Required dependency             | Deployment                                |
|--------|-------------------------------------------------|---------------------------------|-------------------------------------------|
| `gorm` | `*gorm.DB` + in-memory timer                    | `database.DatabaseConnector`    | **Single node only**                      |
| `nats` | NATS 2.12+ JetStream Scheduled Message Delivery | `*nats_connector.NATSConnector` | **Multi-instance** with KV-backed persistence |

> ⚠️ **`gorm` mode is single-node.** The in-memory timer lives inside each
> process. Running multiple replicas against the same database will cause
> **every job to fire on every replica**. Do not deploy `gorm` mode with
> more than one instance.

> ✅ **`nats` mode supports multi-instance deployments.** Replicas share
> the same durable consumer; each scheduled message is delivered to
> exactly one replica via JetStream work-queue semantics. First-deploy
> provisioning of the stream, KV buckets, and consumer is serialised
> through `nats_connector.Once`, so concurrent boots are safe. Recurring
> chains survive replica crashes, leader re-elections, and rolling
> restarts; a background reconciler republishes any job whose next-run
> has slipped past its grace window.

## Usage in Fx

### GORM mode

Requires a `database.DatabaseConnector` (e.g. `sqlite_connector` or
`postgres_connector`) in the same fx graph.

```go
import (
    "github.com/weedbox/common-modules/scheduler"
    "github.com/weedbox/common-modules/sqlite_connector"
)

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        sqlite_connector.Module("sqlite"),  // or postgres_connector
        scheduler.Module("scheduler"),
    }, nil
}
```

### NATS mode

Requires a `nats_connector.NATSConnector` in the same fx graph.

```go
import (
    "github.com/weedbox/common-modules/nats_connector"
    "github.com/weedbox/common-modules/scheduler"
)

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        nats_connector.Module("nats"),
        scheduler.Module("scheduler"),
    }, nil
}
```

Both `NATS` and `DB` are declared `optional:"true"` in the module's
internal `Params`. If the configured mode's required dependency is missing
at `OnStart`, the hook returns `scheduler.ErrModeNotConfigured`.

> **Loading order:** `scheduler` belongs in **Phase 2 (`loadModules`)**,
> and must be declared **after** its backing connector (`sqlite_connector`
> / `postgres_connector` for `gorm` mode, `nats_connector` for `nats`
> mode). Register handlers and jobs via `fx.Invoke` — they are safe even
> before `OnStart` (see the pending-queue behaviour below).

### Injection into your own modules

Inject `*scheduler.SchedulerModule` directly — no `name` tag required.

```go
import "github.com/weedbox/common-modules/scheduler"

type Params struct {
    fx.In
    Scheduler *scheduler.SchedulerModule
}
```

## Configuration

### Core

| Parameter                      | Default                | Purpose                                        |
|--------------------------------|------------------------|------------------------------------------------|
| `{scope}.mode`                 | `gorm`                 | Backend: `gorm` or `nats`                      |
| `{scope}.nats.streamName`      | `SCHEDULER`            | JetStream stream name (NATS mode)              |
| `{scope}.nats.subjectPrefix`   | `scheduler`            | Scheduled message subject prefix               |
| `{scope}.nats.consumerName`    | `scheduler-worker`     | Durable consumer name                          |
| `{scope}.nats.jobBucket`       | `SCHEDULER_JOBS`       | KV bucket for job definitions                  |
| `{scope}.nats.execBucket`      | `SCHEDULER_EXECUTIONS` | KV bucket for in-flight execution state        |

`{scope}` is the string passed to `scheduler.Module("scheduler")` — by
convention `scheduler`.

### NATS-mode tuning knobs (optional)

All keys live under `{scope}.nats.*`. Each one is **only forwarded when
explicitly set**; omit a key to keep the underlying `Weedbox/scheduler`
default. Durations accept Viper-friendly forms (`"30s"`, `"5m"`).

| Parameter                              | Upstream default | Purpose                                                                 |
|----------------------------------------|------------------|-------------------------------------------------------------------------|
| `duplicatesWindow`                     | (lib default)    | Stream-level dedup window                                               |
| `reconcilerInterval`                   | `30s`            | Background republish reconciler tick                                    |
| `reconcilerGracePeriod`                | `30s`            | Lag tolerated before a recurring job is treated as stuck and republished |
| `addJobRetryBudget`                    | `5s`             | `AddJob` / `UpdateJobSchedule` retry budget across raft hiccups          |
| `startupStreamReadyTimeout`            | `30s`            | Wait for the stream's raft leader at start-up                           |
| `jetStreamReadyTimeout`                | `30s`            | Wait for the JetStream metaleader                                       |
| `startPhaseTimeout`                    | `30s`            | Per-phase cap inside `Start()`                                          |
| `loadJobsConcurrency`                  | `32`             | Worker pool size for parallel KV reads during start-up                  |
| `loadJobsAsyncPublishTimeout`          | `30s`            | Cap on draining the start-up async-publish queue                        |
| `onceKey`                              | `scheduler.init` | Key used on the shared `nats_connector` lock bucket for first-deploy provisioning |
| `publishRetry.attempts`                | (lib default)    | Retry attempts for scheduled-message publishes                          |
| `publishRetry.initialBackoff`          | (lib default)    | Initial backoff between publish retries                                 |

> 💡 `loadJobsConcurrency` and `loadJobsAsyncPublishTimeout` (added in
> `Weedbox/scheduler` v0.3.0) tune the parallel KV reload at start-up.
> Increase concurrency on deployments with many persisted jobs to cut
> `Start()` time from `O(N × RTT)` to roughly `O((N / concurrency) × RTT)`.

> ℹ️ The lock bucket used for `onceKey` is owned by `nats_connector` and
> configured via its own `lock.bucket` key — **not** through the
> scheduler module.

### TOML example

```toml
[scheduler]
mode = "gorm"      # or "nats"

# Only relevant in nats mode
[scheduler.nats]
streamName    = "SCHEDULER"
subjectPrefix = "scheduler"
consumerName  = "scheduler-worker"
jobBucket     = "SCHEDULER_JOBS"
execBucket    = "SCHEDULER_EXECUTIONS"

# Optional tuning knobs (omit to use upstream defaults)
# duplicatesWindow             = "24h"
# reconcilerInterval           = "30s"
# reconcilerGracePeriod        = "30s"
# addJobRetryBudget            = "5s"
# startupStreamReadyTimeout    = "30s"
# jetStreamReadyTimeout        = "30s"
# startPhaseTimeout            = "30s"
# loadJobsConcurrency          = 32
# loadJobsAsyncPublishTimeout  = "30s"
# onceKey                      = "scheduler.init"

# [scheduler.nats.publishRetry]
# attempts        = 3
# initialBackoff  = "1s"
```

### Environment variables

Environment variables follow Viper's default mapping used by the `configs`
module: dots in the key become underscores, the whole key is uppercased,
and camelCase segments are **not** split. So `scheduler.nats.streamName`
maps to `<PREFIX>_SCHEDULER_NATS_STREAMNAME` — one token, no extra
underscore inside `STREAMNAME`.

```bash
# With configs.NewConfig("MYAPP")
export MYAPP_SCHEDULER_MODE=nats
export MYAPP_SCHEDULER_NATS_STREAMNAME=SCHEDULER
export MYAPP_SCHEDULER_NATS_SUBJECTPREFIX=scheduler
export MYAPP_SCHEDULER_NATS_CONSUMERNAME=scheduler-worker
export MYAPP_SCHEDULER_NATS_JOBBUCKET=SCHEDULER_JOBS
export MYAPP_SCHEDULER_NATS_EXECBUCKET=SCHEDULER_EXECUTIONS
export MYAPP_SCHEDULER_NATS_LOADJOBSCONCURRENCY=64
export MYAPP_SCHEDULER_NATS_RECONCILERGRACEPERIOD=1m
```

## API Reference

All methods live on `*scheduler.SchedulerModule`.

### SetHandler

```go
func (m *SchedulerModule) SetHandler(h libsched.JobHandler)
```

Sets the single **global** job handler. Typically called once at startup
via `fx.Invoke`. May be called before or after the scheduler has started;
subsequent events are routed to the most recently set handler.

```go
import (
    libsched "github.com/Weedbox/scheduler"
    "github.com/weedbox/common-modules/scheduler"
    "go.uber.org/fx"
)

fx.Invoke(func(s *scheduler.SchedulerModule) {
    s.SetHandler(func(ctx context.Context, e libsched.JobEvent) error {
        // Dispatch on e.ID() or e.Metadata()
        return nil
    })
})
```

### EnsureJob

```go
func (m *SchedulerModule) EnsureJob(id string, schedule libsched.Schedule, metadata map[string]string) error
```

Idempotent registration:

- If the ID does not exist, the job is added.
- If it exists, its schedule is updated (no-op if unchanged).

Safe to call from `fx.Invoke`. If the scheduler has not started yet, the
operation is queued and applied during `OnStart`.

```go
fx.Invoke(func(s *scheduler.SchedulerModule) error {
    sch, _ := libsched.NewCronSchedule("0 3 * * *")
    return s.EnsureJob("daily_cleanup", sch, nil)
})
```

### SubmitJob

```go
func (m *SchedulerModule) SubmitJob(id string, schedule libsched.Schedule, metadata map[string]string) error
```

Adds a one-shot or dynamically created job. Returns
`libsched.ErrJobAlreadyExists` if the ID collides. Also supports the
pending-queue pattern used by `EnsureJob`.

```go
func scheduleReminder(s *scheduler.SchedulerModule, userID string, at time.Time) error {
    sch, _ := libsched.NewOnceSchedule(at)
    return s.SubmitJob(
        fmt.Sprintf("reminder-%s-%d", userID, at.Unix()),
        sch,
        map[string]string{"kind": "reminder", "user_id": userID},
    )
}
```

### RemoveJob / GetJob / ListJobs

```go
func (m *SchedulerModule) RemoveJob(id string) error
func (m *SchedulerModule) GetJob(id string) (libsched.Job, error)
func (m *SchedulerModule) ListJobs() []libsched.Job
```

Valid **only after the scheduler has started**. Before startup, `RemoveJob`
and `GetJob` return `libsched.ErrSchedulerNotStarted`; `ListJobs` returns
`nil`.

```go
func (m *MyModule) cancelReminder(id string) error {
    return m.params.Scheduler.RemoveJob(id)
}

func (m *MyModule) listAll() {
    for _, job := range m.params.Scheduler.ListJobs() {
        m.logger.Info("job", zap.String("id", job.ID()))
    }
}
```

### GetScheduler

```go
func (m *SchedulerModule) GetScheduler() libsched.Scheduler
```

Escape hatch — direct access to the underlying `libsched.Scheduler`. Use
only when the wrapper API is insufficient. Returns `nil` before `OnStart`.

## Schedule Types

Schedule constructors live in the underlying library. Import it as
`libsched "github.com/Weedbox/scheduler"`:

| Constructor                                        | Meaning                                                 |
|----------------------------------------------------|---------------------------------------------------------|
| `libsched.NewIntervalSchedule(d)`                  | Fire every `d` (`time.Duration`)                        |
| `libsched.NewCronSchedule(expr)`                   | Fire on a cron expression (e.g. `"0 3 * * *"`)          |
| `libsched.NewOnceSchedule(t)`                      | Fire once at `time.Time` `t`                            |
| `libsched.NewStartAtIntervalSchedule(start, d)`    | Begin at `start`, then every `d` thereafter             |

```go
import (
    "time"
    libsched "github.com/Weedbox/scheduler"
)

// Every 5 minutes
sch1, _ := libsched.NewIntervalSchedule(5 * time.Minute)

// Daily at 03:00
sch2, _ := libsched.NewCronSchedule("0 3 * * *")

// One-shot reminder
sch3, _ := libsched.NewOnceSchedule(time.Now().Add(10 * time.Minute))

// Begin tomorrow at 09:00, then hourly
start := time.Now().Add(24 * time.Hour).Truncate(time.Hour)
sch4, _ := libsched.NewStartAtIntervalSchedule(start, time.Hour)
```

## Static vs Dynamic Jobs

Two patterns cover most use cases.

### Static jobs

Compile-time constant IDs, registered at startup with `EnsureJob`. Because
`EnsureJob` is idempotent, it is safe to call on every boot.

```go
const (
    JobDailyCleanup  = "daily_cleanup"
    JobHourlyMetrics = "hourly_metrics"
)

fx.Invoke(func(s *scheduler.SchedulerModule) error {
    sch, _ := libsched.NewCronSchedule("0 3 * * *")
    return s.EnsureJob(JobDailyCleanup, sch, nil)
}),
fx.Invoke(func(s *scheduler.SchedulerModule) error {
    sch, _ := libsched.NewIntervalSchedule(time.Hour)
    return s.EnsureJob(JobHourlyMetrics, sch, nil)
}),
```

### Dynamic jobs

Runtime-generated unique IDs (reminders, delayed tasks, user-scheduled
events). Use `SubmitJob` and carry classifying info in `metadata`.

```go
func (m *ReminderModule) Schedule(userID string, at time.Time, payload string) error {
    sch, _ := libsched.NewOnceSchedule(at)
    return m.params.Scheduler.SubmitJob(
        fmt.Sprintf("reminder-%s-%d", userID, at.Unix()),
        sch,
        map[string]string{
            "kind":    "reminder",
            "user_id": userID,
            "payload": payload,
        },
    )
}
```

## Handler Dispatching Pattern

There is exactly one global handler. Dispatch inside it by `e.ID()` (for
static jobs with known constants) or by `e.Metadata()` (for dynamic jobs
with a conventional key such as `"kind"`).

```go
fx.Invoke(func(s *scheduler.SchedulerModule, r *ReminderModule) {
    s.SetHandler(func(ctx context.Context, e libsched.JobEvent) error {
        // Dynamic jobs: dispatch by metadata
        if kind := e.Metadata()["kind"]; kind != "" {
            switch kind {
            case "reminder":
                return r.Send(ctx, e.Metadata()["user_id"], e.Metadata()["payload"])
            }
        }

        // Static jobs: dispatch by ID
        switch e.ID() {
        case JobDailyCleanup:
            return doCleanup(ctx)
        case JobHourlyMetrics:
            return flushMetrics(ctx)
        }
        return nil
    })
})
```

The module itself is intentionally unopinionated about how you classify
jobs — ID prefix, metadata key, or any other convention all work.

## Error Handling

| Error                                 | When                                                                 |
|---------------------------------------|----------------------------------------------------------------------|
| `scheduler.ErrModeNotConfigured`      | Required dependency for the selected mode is missing at `OnStart`.   |
| `libsched.ErrJobAlreadyExists`        | `SubmitJob` called with an ID that already exists.                   |
| `libsched.ErrJobNotFound`             | `GetJob` / `RemoveJob` called with an unknown ID.                    |
| `libsched.ErrSchedulerNotStarted`     | `RemoveJob` / `GetJob` called before `OnStart`.                      |

`EnsureJob` handles `ErrJobNotFound` internally (adds when missing,
updates when present) — callers do not need to branch on it.

In the global handler, return a non-nil error to signal failure. Behaviour
on failure depends on the underlying library scheduler (retry / redelivery
semantics are owned by `libsched`).

```go
s.SetHandler(func(ctx context.Context, e libsched.JobEvent) error {
    if err := processEvent(ctx, e); err != nil {
        logger.Error("job failed", zap.String("id", e.ID()), zap.Error(err))
        return err
    }
    return nil
})
```

## Deployment Notes

- **`gorm` mode is for single-node deployments** (a worker, a single-process
  service, a CLI). Do not scale horizontally — timers fire independently
  in every process.
- **`nats` mode requires NATS Server 2.12 or later** with JetStream
  enabled. Startup fails with `ErrNATSServerTooOld` on older servers.
- **`nats` mode supports multi-instance deployments.** Multiple scheduler
  replicas can run against the same stream concurrently:
  - First-deploy provisioning of the stream, KV buckets, and durable
    consumer is serialised through `nats_connector.Once`, so concurrent
    boots are safe and share the same lock substrate as every other
    common-module that uses `Once`.
  - The durable consumer's work-queue semantics ensure each scheduled
    message is delivered to exactly one replica.
  - On start-up the library reloads jobs from the KV bucket and filters
    stale in-flight scheduled messages against the persisted state
    (rather than purging the stream), so a crashed run does not
    duplicate triggers and a healthy peer is not disrupted.
  - A background reconciler republishes any recurring job whose next-run
    has slipped past `nats.reconcilerGracePeriod`, recovering from
    publish failures or unclean shutdowns.

## Related

- [database](./database.md) — `DatabaseConnector` interface used by `gorm` mode
- [postgres_connector](./postgres_connector.md) — PostgreSQL backend for `gorm` mode
- [sqlite_connector](./sqlite_connector.md) — SQLite backend for `gorm` mode
- [nats_connector](./nats_connector.md) — required by `nats` mode
