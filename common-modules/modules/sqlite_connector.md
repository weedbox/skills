# sqlite_connector

SQLite connector implementing `database.DatabaseConnector` with GORM.

## Installation

```bash
go get github.com/weedbox/common-modules/sqlite_connector
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/sqlite_connector"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        sqlite_connector.Module("database"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.path` | `./data.db` | Database file path |
| `{scope}.loglevel` | `4` | GORM log level |
| `{scope}.debug_mode` | `false` | Enable detailed SQL logging |

### GORM Log Levels

| Level | Value | Description |
|-------|-------|-------------|
| Silent | `1` | No logging |
| Error | `2` | Log errors only |
| Warn | `3` | Log errors and warnings |
| Info | `4` | Log all queries |

### TOML Example

```toml
[database]
path = "./data/app.db"
loglevel = 4
debug_mode = false
```

### Environment Variables

```bash
export DATABASE_PATH=./data/app.db
export DATABASE_LOGLEVEL=4
export DATABASE_DEBUG_MODE=false
```

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

    // Auto-migrate
    db.AutoMigrate(&models.User{})

    return nil
}
```

## When to Use SQLite

**Good for:**
- Prototypes and development
- Embedded applications
- Testing environments
- Single-user applications
- Offline-first apps
- Small to medium datasets

**Not recommended for:**
- High-concurrency write operations
- Multi-process access
- Large datasets (>10GB)
- High availability requirements

## SQLite-Specific Features

### In-Memory Database

```toml
[database]
path = ":memory:"
```

Useful for testing - database is destroyed when connection closes.

### File Path Auto-Creation

The connector automatically creates parent directories for the database file.

```toml
[database]
path = "./data/nested/deep/app.db"
# ./data/nested/deep/ will be created automatically
```

### WAL Mode

SQLite uses WAL (Write-Ahead Logging) mode by default for better concurrency:

```go
// Enable WAL mode (usually automatic)
db.Exec("PRAGMA journal_mode=WAL;")
```

## Testing with SQLite

SQLite is excellent for unit tests:

```go
func TestUserRepository(t *testing.T) {
    // Use in-memory database for tests
    db, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    db.AutoMigrate(&models.User{})

    repo := NewUserRepository(db)

    // Run tests...
}
```

## Switching to PostgreSQL

When moving to production, change only the module import:

```go
// Development
func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        sqlite_connector.Module("database"),
        mymodule.Module("mymodule"),
    }, nil
}

// Production
func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        postgres_connector.Module("database"),
        mymodule.Module("mymodule"),
    }, nil
}
```

Your application code using `database.DatabaseConnector` remains unchanged.

## Limitations

| Feature | SQLite | PostgreSQL |
|---------|--------|------------|
| Concurrent writes | Limited | Excellent |
| JSON operators | Basic | Full JSONB |
| Full-text search | FTS5 | Built-in |
| Array types | No | Yes |
| Network access | No | Yes |

## Related

- [database](./database.md) - Database interface
- [postgres_connector](./postgres_connector.md) - PostgreSQL alternative
