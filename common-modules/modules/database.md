# database

Common interface for database operations, enabling connector abstraction.

## Installation

```bash
go get github.com/weedbox/common-modules/database
```

## Interface Definition

```go
type DatabaseConnector interface {
    GetDB() *gorm.DB
}
```

## Purpose

This module provides a common interface that allows:
- Swapping database backends without changing application code
- Dependency injection of database connectors
- Consistent database access patterns across modules

## Implementations

| Connector | Module | Use Case |
|-----------|--------|----------|
| PostgreSQL | `postgres_connector` | Production systems |
| SQLite | `sqlite_connector` | Development, testing, embedded |

## Usage in Modules

```go
import "github.com/weedbox/common-modules/database"

type Params struct {
    fx.In
    Database database.DatabaseConnector
}

type MyModule struct {
    params Params
}

func (m *MyModule) OnStart(ctx context.Context) error {
    db := m.params.Database.GetDB()

    // Auto-migrate models
    if err := db.AutoMigrate(&models.User{}, &models.Post{}); err != nil {
        return err
    }

    return nil
}
```

## GORM Operations

### Create

```go
user := models.User{Name: "John", Email: "john@example.com"}
result := db.Create(&user)
if result.Error != nil {
    return result.Error
}
// user.ID is now set
```

### Read

```go
// Find by primary key
var user models.User
db.First(&user, "id = ?", userID)

// Find all
var users []models.User
db.Find(&users)

// With conditions
db.Where("age > ?", 18).Find(&users)

// With limit and offset
db.Limit(10).Offset(20).Find(&users)

// Order
db.Order("created_at desc").Find(&users)
```

### Update

```go
// Update single field
db.Model(&user).Update("name", "Jane")

// Update multiple fields
db.Model(&user).Updates(models.User{Name: "Jane", Age: 30})

// Update with map
db.Model(&user).Updates(map[string]interface{}{"name": "Jane", "age": 30})
```

### Delete

```go
// Delete record
db.Delete(&user)

// Soft delete (if model has DeletedAt field)
db.Delete(&user)

// Hard delete
db.Unscoped().Delete(&user)
```

## Switching Connectors

The power of this interface is easy backend switching:

```go
// Development: Use SQLite
func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        sqlite_connector.Module("database"),
        mymodule.Module("mymodule"),
    }, nil
}

// Production: Use PostgreSQL
func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        postgres_connector.Module("database"),
        mymodule.Module("mymodule"),
    }, nil
}
```

Your application code (`mymodule`) doesn't change - it just uses `database.DatabaseConnector`.

## Loading Multiple Connectors

`sqlite_connector` and `postgres_connector` can be loaded into the same `fx.App` simultaneously — for example to keep a local SQLite cache alongside a PostgreSQL primary. Each `Module(scope)` call always registers a **named** `database.DatabaseConnector` keyed by its `scope`, so consumers disambiguate using fx's `name` tag.

The **first** connector module loaded into the process (across all `database.DatabaseConnector` implementations) also exposes itself as the **unnamed default**, so existing single-load code that injects `database.DatabaseConnector` without a tag keeps working with zero changes. Load order controls which connector becomes the default — if load order is brittle, always inject by named tag in multi-load setups.

See [sqlite_connector.md](./sqlite_connector.md) and [postgres_connector.md](./postgres_connector.md) for the full code example and the `fxmodule.ResetClaim` test caveat.

## Implementing Your Own Connector

A custom `DatabaseConnector` implementation only needs a `Module(scope string) fx.Option` factory that wires its constructor through `database.Module`:

```go
package my_connector

import (
    "context"

    "github.com/weedbox/common-modules/database"
    "go.uber.org/fx"
    "go.uber.org/zap"
)

type Params struct {
    fx.In

    Lifecycle fx.Lifecycle
    Logger    *zap.Logger
}

func Module(scope string) fx.Option {
    return database.Module(scope, func(p Params) database.DatabaseConnector {
        c := &MyConnector{ /* ... */ }
        p.Lifecycle.Append(fx.Hook{OnStart: c.onStart, OnStop: c.onStop})
        return c
    })
}
```

`database.Module` is a thin wrapper around `weedbox/fxmodule.InterfaceModule[DatabaseConnector]`. It handles the shared multi-load wiring described above: the connector is always registered as `name:"<scope>"`, and the first connector loaded into the process also exposes itself as the unnamed default.

## Related

- [postgres_connector](./postgres_connector.md) - PostgreSQL implementation
- [sqlite_connector](./sqlite_connector.md) - SQLite implementation
