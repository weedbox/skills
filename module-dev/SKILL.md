---
name: module-dev
description: |
  Priority: HIGH — Self-contained reference. This file and its reference docs contain ALL necessary
  patterns for module development. DO NOT fetch source code from GitHub.
  Weedbox module/package development guide with Uber Fx dependency injection.
  Use when: creating weedbox module, developing weedbox package, building custom module,
  implementing dependency injection, managing module lifecycle (OnStart/OnStop),
  configuring module with Viper, injecting database/logger/NATS/Redis dependencies.
  Covers: Module() function, Params struct, fx.In/fx.Out, weedbox.Module generic,
  configuration management, error handling, event publishing.
  Keywords: weedbox module, weedbox package, uber fx, dependency injection, module development,
  OnStart, OnStop, Params, fx.Invoke, fx.Provide, lifecycle hooks,
  create module, new module, custom module, Go module, weedbox generic, module skills.
---

# Weedbox Module Development

This skill helps develop modules using the weedbox framework with Uber FX dependency injection.

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

    // User modules - REQUIRE `name` tag (all use Method 2)
    User *user.UserManager `name:"user"`
    RBAC *rbac.RBACManager `name:"rbac"`
    Auth *auth.AuthManager `name:"auth"`

    // Method 1 modules - NO `name` tag needed
    ViewManager *view_manager.ViewManager

    // Common modules - NO `name` tag
    Database database.DatabaseConnector
}
```

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

### GORM Index Configuration

GORM supports various index types via struct tags:

```go
type Example struct {
    // Basic index
    Email string `gorm:"index"`

    // Named index
    Name string `gorm:"index:idx_name"`

    // Unique index
    Code string `gorm:"uniqueIndex"`

    // Named unique index
    SKU string `gorm:"uniqueIndex:idx_sku"`

    // Composite index (same index name = composite)
    TenantID string `gorm:"index:idx_tenant_user"`
    UserID   string `gorm:"index:idx_tenant_user"`

    // Index with sort order
    Score int `gorm:"index:idx_score,sort:desc"`

    // Index priority (controls column order in composite index)
    FirstName string `gorm:"index:idx_full_name,priority:1"`
    LastName  string `gorm:"index:idx_full_name,priority:2"`
}
```

### Index Naming Convention

**Important**: In SQLite (and some other databases), index names are **global** across all tables, not scoped to individual tables. Using generic names like `idx_status` on multiple tables will cause conflicts during migration.

**Naming Rule**: Always prefix index names with the table name.

```
Format: idx_<table>_<column(s)>
```

```go
// ❌ BAD - Will conflict if another table also has "status" index
type Task struct {
    Status string `gorm:"index:idx_status"`
}
type Job struct {
    Status string `gorm:"index:idx_status"`  // CONFLICT!
}

// ✅ GOOD - Table-prefixed index names
type Task struct {
    Status string `gorm:"index:idx_task_status"`
}
type Job struct {
    Status string `gorm:"index:idx_job_status"`
}

// ✅ Composite index with table prefix
type Task struct {
    WorkspaceID string `gorm:"index:idx_task_ws_status"`
    Status      string `gorm:"index:idx_task_ws_status"`
}

// ✅ Unique index with table prefix
type Permission struct {
    RoleID     string `gorm:"uniqueIndex:idx_permission_role_resource"`
    ResourceID string `gorm:"uniqueIndex:idx_permission_role_resource"`
}
```

**Note**: Unnamed indexes (e.g., `gorm:"index"`) are auto-generated by GORM with table-scoped names, so they are safe. Only explicitly named indexes need the table prefix.

### Common Index Patterns

```go
// Foreign key with index (auto-named, safe)
WorkspaceID string `gorm:"type:varchar(36);not null;index" json:"workspace_id"`

// Status field with table-prefixed name
Status string `gorm:"type:varchar(20);default:'pending';index:idx_task_status" json:"status"`

// Composite index for common query patterns
// Query: WHERE workspace_id = ? AND status = ?
type Task struct {
    WorkspaceID string `gorm:"type:varchar(36);index:idx_task_ws_status"`
    Status      string `gorm:"type:varchar(20);index:idx_task_ws_status"`
}

// Unique constraint across multiple columns
type Permission struct {
    RoleID     string `gorm:"uniqueIndex:idx_permission_role_resource"`
    ResourceID string `gorm:"uniqueIndex:idx_permission_role_resource"`
}
```

### Auto Migration

```go
func (m *MyModule) OnStart(ctx context.Context) error {
    db := m.Params().Database.GetDB()
    return db.AutoMigrate(&models.MyModel{})
}
```

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

## Module Skills

Module Skills are documentation files placed in `.skills/` subdirectories within each module. They help Claude Code understand and assist with specific module development.

### Directory Structure

```
pkg/
├── mymodule/
│   ├── .skills/
│   │   └── mymodule-development.md    # Business logic module skill
│   ├── module.go
│   └── ...
└── mymodule_apis/
    ├── .skills/
    │   └── mymodule-apis-development.md  # API module skill
    └── ...
```

### When to Create Module Skills

- After completing new module development
- For complex modules with multiple files or special logic
- For modules requiring cross-module integration documentation
- For API modules to provide endpoint reference

### Skill Content Overview

**Business Logic Module** (`*-development.md`):
- Module overview and purpose
- Data model (database table structure)
- Configuration structures
- Manager methods (Create, Get, Update, Delete, List)
- Error types
- Usage examples

**API Module** (`*-apis-development.md`):
- API endpoints table
- Request/Response structures
- HTTP status code mappings
- Validation rules
- cURL examples

For complete documentation, see [./references/MODULE_SKILLS.md](./references/MODULE_SKILLS.md).

## Related Documentation

- [./references/METHOD1_MANUAL_FX.md](./references/METHOD1_MANUAL_FX.md) - Manual FX module details
- [./references/METHOD2_WEEDBOX_GENERIC.md](./references/METHOD2_WEEDBOX_GENERIC.md) - Weedbox generic module details
- [./references/MODULE_SKILLS.md](./references/MODULE_SKILLS.md) - Module skills guide
- [../user-modules/SKILL.md](../user-modules/SKILL.md) - User management, auth, and RBAC modules
