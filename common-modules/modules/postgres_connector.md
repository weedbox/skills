# postgres_connector

PostgreSQL connector implementing `database.DatabaseConnector` with GORM.

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
| `{scope}.sslmode` | `false` | Enable SSL |
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
host = "localhost"
port = 5432
dbname = "myapp"
user = "postgres"
password = "secret"
sslmode = false
loglevel = 4
debug_mode = false
```

### Environment Variables

```bash
export DATABASE_HOST=localhost
export DATABASE_PORT=5432
export DATABASE_DBNAME=myapp
export DATABASE_USER=postgres
export DATABASE_PASSWORD=secret
export DATABASE_SSLMODE=false
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
    db := m.params.Database.GetDB()
    return db.Create(user).Error
}
```

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
        // Deduct from sender
        if err := tx.Model(&Account{}).
            Where("id = ?", fromID).
            Update("balance", gorm.Expr("balance - ?", amount)).Error; err != nil {
            return err
        }

        // Add to receiver
        if err := tx.Model(&Account{}).
            Where("id = ?", toID).
            Update("balance", gorm.Expr("balance + ?", amount)).Error; err != nil {
            return err
        }

        return nil
    })
}
```

## Connection Pooling

PostgreSQL connector uses GORM's built-in connection pooling. Configure at the database level if needed.

## SSL Mode Options

| Mode | Description |
|------|-------------|
| `disable` | No SSL |
| `require` | Always SSL, no verification |
| `verify-ca` | SSL with CA verification |
| `verify-full` | SSL with full verification |

## Related

- [database](./database.md) - Database interface
- [sqlite_connector](./sqlite_connector.md) - SQLite alternative
