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
| `{scope}.max_idle_conns` | `1` | Max idle connections (replica pool when split is on) |
| `{scope}.conn_max_lifetime` | `3600` | Connection max lifetime in seconds |
| `{scope}.conn_max_idle_time` | `30` | Seconds an idle connection can stay before being closed |
| `{scope}.enable_read_write_split` | `true` | Enable read/write splitting via dbresolver |
| `{scope}.write_max_open_conns` | `10` | Max open connections in the primary (write) pool |
| `{scope}.write_max_idle_conns` | `1` | Max idle connections in the primary (write) pool |
| `{scope}.synchronous` | `""` (SQLite default) | `PRAGMA synchronous`: `OFF`, `NORMAL`, `FULL`, `EXTRA` |
| `{scope}.cache_size` | `0` (SQLite default) | `PRAGMA cache_size`: negative = KB (e.g. `-65536` = 64 MiB), positive = pages |
| `{scope}.locking_mode` | `""` (SQLite default `NORMAL`) | `PRAGMA locking_mode`: `NORMAL` or `EXCLUSIVE`. `EXCLUSIVE` is single-connection only |

> `max_idle_conns` and `write_max_idle_conns` default to `1` so idle connections do not pin WAL frames and block `wal_checkpoint`. `conn_max_idle_time` works with this — see [Connection Pool Behavior](#connection-pool-behavior).

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
max_idle_conns = 1
conn_max_lifetime = 3600
conn_max_idle_time = 30
enable_read_write_split = true
write_max_open_conns = 10
write_max_idle_conns = 1
synchronous = ""
cache_size = 0
locking_mode = ""
```

### Environment Variables

```bash
export DATABASE_PATH=./data/app.db
export DATABASE_LOGLEVEL=2
export DATABASE_DEBUG_MODE=false
export DATABASE_ENABLE_WAL=true
export DATABASE_BUSY_TIMEOUT=5000
export DATABASE_MAX_OPEN_CONNS=10
export DATABASE_MAX_IDLE_CONNS=1
export DATABASE_CONN_MAX_LIFETIME=3600
export DATABASE_CONN_MAX_IDLE_TIME=30
export DATABASE_ENABLE_READ_WRITE_SPLIT=true
export DATABASE_WRITE_MAX_OPEN_CONNS=10
export DATABASE_WRITE_MAX_IDLE_CONNS=1
export DATABASE_SYNCHRONOUS=NORMAL
export DATABASE_CACHE_SIZE=-65536
export DATABASE_LOCKING_MODE=
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

- **Primary (write)** — pool size is controlled by `write_max_open_conns` / `write_max_idle_conns` (defaults `10` / `1`). The idle default is `1` because SQLite serializes writes; lower `write_max_open_conns` to `1` to fully serialize writes at the Go layer if you are seeing `SQLITE_BUSY` errors under contention.
- **Replicas (read)** — pool size follows `max_open_conns` / `max_idle_conns`, allowing many concurrent readers.

Both pools open the database with the **same DSN** (including `_journal_mode=WAL`). Routing happens at the SQL operation layer via `dbresolver`: `SELECT` / `First` / `Find` / `Count` / `Pluck` / `Take` / `Scan` go to the replica pool, while `Create` / `Save` / `Update` / `Delete` / `Transaction` / `Raw` / `Exec` go to the primary. Application code does not need to change. Set `enable_read_write_split = false` to fall back to the legacy single-pool behavior.

> **Why no `mode=ro` on the replica DSN**: a `mode=ro` connection cannot create or write the `-shm` file, register read marks in the WAL, or otherwise participate in the WAL protocol — so it would silently miss frames produced by the primary. Routing safety comes from `dbresolver` at the SQL layer, not from the file open mode.

> Read/write splitting only makes sense when `enable_wal = true` — WAL is what allows readers and writers to operate concurrently against the same SQLite file.

### Connection Pool Behavior

- When `enable_read_write_split = false`: a single pool is sized by `max_open_conns` / `max_idle_conns`. `conn_max_lifetime` and `conn_max_idle_time` apply.
- When `enable_read_write_split = true`:
  - Primary (write) pool uses `write_max_open_conns` / `write_max_idle_conns`.
  - Replica (read) pool uses `max_open_conns` / `max_idle_conns`.
  - `conn_max_lifetime` and `conn_max_idle_time` apply to both pools.
- **`conn_max_idle_time` is critical under WAL**: an idle connection holds the read snapshot of its last query and prevents `wal_checkpoint` from advancing past it. Closing idle connections promptly lets WAL frames recycle into the main file. The default `30s` together with low idle defaults (`1`) keeps the WAL bounded.

### PRAGMA Tuning Knobs

These knobs are passed via the DSN and apply to every connection in both pools.

- **`synchronous`** — sets `PRAGMA synchronous`. With WAL, `NORMAL` is safe and significantly cheaper than `FULL` (one fsync per checkpoint instead of one per commit). Use `NORMAL` on slow-fsync storage (NFS, networked PVs); a crash can lose the very last committed transaction but the database remains consistent. `OFF` drops durability further; avoid unless you understand the trade-off.
- **`cache_size`** — sets `PRAGMA cache_size`. Negative values are KB (e.g. `-65536` = 64 MiB), positive values are pages. A larger cache reduces page reads from disk and is one of the cheapest wins on slow storage.
- **`locking_mode`** — sets `PRAGMA locking_mode`. `EXCLUSIVE` keeps the database lock held for the connection's lifetime, avoiding repeated lock acquisition syscalls on filesystems with slow/unreliable advisory locks (e.g. NFS). **Only safe with a single connection**: requires `enable_read_write_split=false`, `max_open_conns=1`, `write_max_open_conns=1`. The connector logs a warning at startup if it detects an incompatible combination.

## Running on Slow / Networked Storage (e.g. NFS-backed PV)

SQLite expects a local filesystem with working POSIX advisory locks and fast `fsync`. Networked or shared storage (NFS, some CSI drivers, certain k8s `PersistentVolume` classes) provides neither reliably:

- `fsync` over the network turns every COMMIT into a round-trip-bound operation (often 10–100× slower than local disk).
- Byte-range advisory locks (used by WAL) can break under contention.
- The `-shm` shared-memory file used by WAL behaves poorly when the filesystem is not truly local.

**The first thing to check if SQL feels slow on SQLite is the StorageClass** — `kubectl get pvc -n <ns>` then `kubectl get storageclass`. If it routes through NFS / `nfs-csi-driver` / Longhorn-over-network / similar, that is almost certainly the bottleneck and no amount of connector tuning will eliminate it. The proper fix is a local-disk-backed StorageClass (`local-path`, `topolvm`, hostPath with locality), or switching to PostgreSQL.

If you cannot move off networked storage, the following PRAGMAs trade a small amount of durability/concurrency for substantially less fsync pressure:

```toml
[database]
# WAL + NORMAL: one fsync per checkpoint, not one per commit. Safe under WAL —
# you can lose the very last committed transaction on a crash, but the DB
# remains consistent.
synchronous = "NORMAL"

# 64 MiB page cache. Cuts how often SQLite has to read pages back from the
# slow filesystem.
cache_size = -65536

# Optional, single-connection only: keep the file lock held for the lifetime of
# the connection so SQLite stops doing per-statement lock dances against NFS.
# REQUIRES enable_read_write_split=false, max_open_conns=1, write_max_open_conns=1.
# locking_mode = "EXCLUSIVE"
```

> `locking_mode=EXCLUSIVE` only works with a single connection holding the database. If `enable_read_write_split=true` or any pool size is greater than `1`, the second connection will block forever or fail with `SQLITE_BUSY`. The connector emits a warning log at startup if it detects this misconfiguration.

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
