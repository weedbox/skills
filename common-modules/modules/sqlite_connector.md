# sqlite_connector

SQLite connector implementing `database.DatabaseConnector` with GORM. Supports WAL mode, connection pooling, and optional read/write splitting via `gorm.io/plugin/dbresolver`.

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
| `{scope}.loglevel` | `2` (Error) | GORM log level |
| `{scope}.debug_mode` | `false` | Enable detailed SQL logging |
| `{scope}.enable_wal` | `true` | Enable WAL (Write-Ahead Logging) journal mode |
| `{scope}.busy_timeout` | `5000` | Milliseconds to wait when DB is locked |
| `{scope}.max_open_conns` | `10` | Max open connections (replica pool when split is on) |
| `{scope}.max_idle_conns` | `5` | Max idle connections (replica pool when split is on) |
| `{scope}.conn_max_lifetime` | `3600` | Connection max lifetime in seconds |
| `{scope}.enable_read_write_split` | `true` | Enable read/write splitting via dbresolver |

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
loglevel = 2
debug_mode = false
enable_wal = true
busy_timeout = 5000
max_open_conns = 10
max_idle_conns = 5
conn_max_lifetime = 3600
enable_read_write_split = true
```

### Environment Variables

```bash
export DATABASE_PATH=./data/app.db
export DATABASE_LOGLEVEL=2
export DATABASE_DEBUG_MODE=false
export DATABASE_ENABLE_WAL=true
export DATABASE_BUSY_TIMEOUT=5000
export DATABASE_MAX_OPEN_CONNS=10
export DATABASE_MAX_IDLE_CONNS=5
export DATABASE_CONN_MAX_LIFETIME=3600
export DATABASE_ENABLE_READ_WRITE_SPLIT=true
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

WAL mode is enabled by default via the `enable_wal` config (applied through the DSN as `_journal_mode=WAL`). It allows readers and writers to operate concurrently without blocking each other. Only disable it (`enable_wal = false`) when you need rollback journal mode for specific compatibility reasons.

The connector also enables foreign keys (`_foreign_keys=on`) and applies `_busy_timeout` automatically — no manual `PRAGMA` calls are required.

### Read/Write Splitting

Enabled by default (`enable_read_write_split = true`). The connector wires a single `*gorm.DB` to two separate `*sql.DB` pools using GORM's [`dbresolver`](https://gorm.io/docs/dbresolver.html) plugin:

- **Primary (write)** — uses the original DSN. Pool is forced to `max_open_conns=1` and `max_idle_conns=1` so writes are serialized at the Go layer, eliminating one source of `SQLITE_BUSY` errors.
- **Replicas (read)** — opens the same database file with `mode=ro` appended. Pool size follows the configured `max_open_conns` / `max_idle_conns`, allowing many concurrent readers.

GORM routes `SELECT` queries to the replica pool and all mutations to the primary, so application code does not need to change. Set `enable_read_write_split = false` to fall back to the legacy single-pool behavior.

> Read/write splitting only makes sense when `enable_wal = true` — WAL is what allows readers and writers to operate concurrently against the same SQLite file.

### Connection Pool Behavior

- When `enable_read_write_split = false`: a single pool is sized by `max_open_conns` / `max_idle_conns`.
- When `enable_read_write_split = true`: the **primary (write) pool is forced to 1 connection** regardless of config, while `max_open_conns` / `max_idle_conns` apply to the **replica (read) pool only**.
- `conn_max_lifetime` applies to both pools.

## Debug Mode

When `debug_mode = true`:
- Log level is forced to Info (all SQL queries logged)
- Parameterized queries are logged with actual values
- Colorful output is enabled
- `RecordNotFound` errors are shown (not silenced)

Useful for development; disable in production.

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
| Concurrent writes | Serialized (1 writer) | Excellent |
| JSON operators | Basic | Full JSONB |
| Full-text search | FTS5 | Built-in |
| Array types | No | Yes |
| Network access | No | Yes |

## Related

- [database](./database.md) - Database interface
- [postgres_connector](./postgres_connector.md) - PostgreSQL alternative
