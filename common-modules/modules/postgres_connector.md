# postgres_connector

PostgreSQL connector implementing `database.DatabaseConnector` with GORM. Supports connection pool tuning via `database/sql`.

## Installation

```bash
go get github.com/weedbox/common-modules/postgres_connector
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/postgres_connector"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        postgres_connector.Module("database"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0` | PostgreSQL host |
| `{scope}.port` | `5432` | PostgreSQL port |
| `{scope}.dbname` | `default` | Database name |
| `{scope}.user` | `postgres` | Username |
| `{scope}.password` | (empty) | Password |
| `{scope}.sslmode` | `false` | Enable SSL (mapped to `sslmode=enable` / `sslmode=disable` in DSN) |
| `{scope}.loglevel` | `2` (Error) | GORM log level |
| `{scope}.debug_mode` | `false` | Enable detailed SQL logging |
| `{scope}.max_open_conns` | `0` (unlimited) | Max open connections; set finite (e.g. `50`) in prod to avoid exhausting `max_connections` |
| `{scope}.max_idle_conns` | `2` | Max idle connections kept in the pool (Go `database/sql` default) |
| `{scope}.conn_max_lifetime` | `0` (no expiration) | Max connection lifetime in seconds; set e.g. `1800` to rotate through failovers |
| `{scope}.conn_max_idle_time` | `0` (no expiration) | Max idle time in seconds; set e.g. `600` to release idle conns under low load |

> Defaults `0 / 2 / 0 / 0` mirror Go's `database/sql` native defaults. They are not production-tuned — set explicit values for production deployments.

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
host = "localhost"
port = 5432
dbname = "myapp"
user = "postgres"
password = "secret"
sslmode = false
loglevel = 2
debug_mode = false

# Connection pool tuning — typical production values
max_open_conns = 50
max_idle_conns = 25
conn_max_lifetime = 1800   # 30 minutes
conn_max_idle_time = 600   # 10 minutes
```

### Environment Variables

```bash
export DATABASE_HOST=localhost
export DATABASE_PORT=5432
export DATABASE_DBNAME=myapp
export DATABASE_USER=postgres
export DATABASE_PASSWORD=secret
export DATABASE_SSLMODE=false
export DATABASE_LOGLEVEL=2
export DATABASE_DEBUG_MODE=false
export DATABASE_MAX_OPEN_CONNS=50
export DATABASE_MAX_IDLE_CONNS=25
export DATABASE_CONN_MAX_LIFETIME=1800
export DATABASE_CONN_MAX_IDLE_TIME=600
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

func (m *MyModule) CreateUser(user *models.User) error {
    return m.params.Database.GetDB().Create(user).Error
}
```

## Connection Pool

The connector applies the four pool knobs to the underlying `*sql.DB` after `gorm.Open`:

- **`max_open_conns`** — hard cap on concurrent connections to PostgreSQL. Pick this with the server's `max_connections` and other clients in mind. `0` (unlimited) is rarely what you want in production.
- **`max_idle_conns`** — pool retains up to this many idle connections to avoid reconnect churn. The Go default of `2` is too low for bursty workloads; raising to roughly `max_open_conns / 2` is a common starting point.
- **`conn_max_lifetime`** — recycle connections after this duration. Critical for managed PostgreSQL behind a load balancer / failover (RDS, Cloud SQL): without it, connections may pin to a node that no longer exists.
- **`conn_max_idle_time`** — close idle connections sooner than `conn_max_lifetime` so the pool shrinks under low load.

## Debug Mode

When `debug_mode = true`:
- Log level is forced to Info (all SQL queries logged)
- Parameterized queries are logged with actual values
- Colorful output is enabled
- `RecordNotFound` errors are shown (not silenced)

Useful for development; disable in production.

## SSL Mode

The connector exposes a single boolean `sslmode` that maps to `sslmode=enable` or `sslmode=disable` in the DSN. If you need full PostgreSQL SSL modes (`require` / `verify-ca` / `verify-full`), the connector currently does not expose them — open an issue or extend the DSN builder.

| Mode | Description |
|------|-------------|
| `disable` | No SSL |
| `require` | Always SSL, no verification |
| `verify-ca` | SSL with CA verification |
| `verify-full` | SSL with full verification |

## PostgreSQL-Specific Features

### JSON/JSONB Columns

```go
type User struct {
    ID       string `gorm:"type:varchar(36);primary_key"`
    Name     string `gorm:"type:varchar(255)"`
    Metadata JSON   `gorm:"type:jsonb"`
}

// Query JSONB
db.Where("metadata->>'role' = ?", "admin").Find(&users)
```

### Array Columns

```go
import "github.com/lib/pq"

type User struct {
    ID   string         `gorm:"type:varchar(36);primary_key"`
    Tags pq.StringArray `gorm:"type:text[]"`
}

// Query arrays
db.Where("? = ANY(tags)", "golang").Find(&users)
```

### Indexes

```go
type User struct {
    ID        string    `gorm:"type:varchar(36);primary_key"`
    Email     string    `gorm:"type:varchar(255);uniqueIndex"`
    CreatedAt time.Time `gorm:"index"`
    Name      string    `gorm:"index:idx_name_email"`
    Status    string    `gorm:"index:idx_name_email"`
}
```

### Transactions

```go
func (m *MyModule) TransferFunds(fromID, toID string, amount int) error {
    db := m.params.Database.GetDB()

    return db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Model(&Account{}).
            Where("id = ?", fromID).
            Update("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
            return err
        }

        if err := tx.Model(&Account{}).
            Where("id = ?", toID).
            Update("balance", gorm.Expr("balance + ?", amount)).Error; err != nil {
            return err
        }

        return nil
    })
}
```

## Related

- [database](./database.md) - Database interface
- [sqlite_connector](./sqlite_connector.md) - SQLite alternative
