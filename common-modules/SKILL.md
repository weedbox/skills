---
name: common-modules
description: |
  Reference guide for weedbox/common-modules - reusable modules for weedbox applications.
  Use when: integrating configs, logger, HTTP server (Gin), database (PostgreSQL, SQLite, GORM),
  NATS messaging (JetStream), Redis cache, or mailer (SMTP) into weedbox projects.
  Covers: configuration management, structured logging, health checks, database connectors,
  message queues, caching, email sending.
  Keywords: common-modules, weedbox module, configs, logger, http_server, database, postgres, sqlite,
  nats, jetstream, redis, mailer, healthcheck, daemon, Uber Fx modules.
---

# Common Modules Reference

This skill provides detailed usage instructions for all modules in `github.com/weedbox/common-modules`.

## Module Index

### Core

| Module | Description | Documentation |
|--------|-------------|---------------|
| `configs` | Configuration management with Viper (TOML, environment variables) | [configs.md](./modules/configs.md) |
| `logger` | Zap-based structured logging with debug mode | [logger.md](./modules/logger.md) |
| `daemon` | Service lifecycle management with ready/health status | [daemon.md](./modules/daemon.md) |

### HTTP

| Module | Description | Documentation |
|--------|-------------|---------------|
| `http_server` | Gin-based HTTP server with CORS support | [http_server.md](./modules/http_server.md) |
| `healthcheck_apis` | Kubernetes-compatible health check endpoints | [healthcheck_apis.md](./modules/healthcheck_apis.md) |

### Database

| Module | Description | Documentation |
|--------|-------------|---------------|
| `database` | Database interface abstraction | [database.md](./modules/database.md) |
| `postgres_connector` | PostgreSQL connector with GORM | [postgres_connector.md](./modules/postgres_connector.md) |
| `sqlite_connector` | SQLite connector with GORM | [sqlite_connector.md](./modules/sqlite_connector.md) |

### Messaging

| Module | Description | Documentation |
|--------|-------------|---------------|
| `nats_connector` | NATS client with JetStream and work queue | [nats_connector.md](./modules/nats_connector.md) |
| `nats_jetstream_server` | Embedded NATS JetStream server | [nats_jetstream_server.md](./modules/nats_jetstream_server.md) |

### Cache

| Module | Description | Documentation |
|--------|-------------|---------------|
| `redis_connector` | Redis client with connection pooling | [redis_connector.md](./modules/redis_connector.md) |

### Utilities

| Module | Description | Documentation |
|--------|-------------|---------------|
| `mailer` | SMTP email sending with TLS support | [mailer.md](./modules/mailer.md) |

---

## Quick Start

### Installation

```bash
go get github.com/weedbox/common-modules
```

### Basic Application Setup

**Important**:
- `configs.NewConfig()` automatically reads `config.toml` from current directory or `./configs/`
- Use `fx.New().Run()` to start the application (not `weedbox.Run()`)

```go
package main

import (
    "os"

    "github.com/spf13/cobra"
    "github.com/weedbox/common-modules/configs"
    "github.com/weedbox/common-modules/daemon"
    "github.com/weedbox/common-modules/http_server"
    "github.com/weedbox/common-modules/logger"
    "go.uber.org/fx"
)

var config *configs.Config

func main() {
    rootCmd := &cobra.Command{
        Use: "myapp",
        RunE: func(cmd *cobra.Command, args []string) error {
            modules, _ := initModules()
            app := fx.New(modules...)
            app.Run()
            return nil
        },
    }

    // Initialize config - automatically reads config.toml
    config = configs.NewConfig("MYAPP")

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}

func initModules() ([]fx.Option, error) {
    return []fx.Option{
        // Phase 1: Preload
        fx.Supply(config),
        logger.Module(),

        // Phase 2: Load
        http_server.Module("http_server"),

        // Phase 3: After (daemon must be last)
        daemon.Module("daemon"),
    }, nil
}
```

---

## Module Loading Order

Modules should be loaded in three phases:

| Phase | Purpose | Modules |
|-------|---------|---------|
| **1. Preload** | Configuration and logging | `configs`, `logger` |
| **2. Load** | Application modules | `http_server`, `database`, `nats_connector`, `redis_connector`, `mailer`, custom modules |
| **3. After** | Lifecycle management | `daemon` (must be last) |

**Important**: The `daemon` module must be placed **last** because it marks the service as ready after all other modules initialize.

---

## Configuration

All modules support configuration via:

1. **TOML files** - Structured configuration
2. **Environment variables** - Override settings per environment

### TOML Example

```toml
[http_server]
host = "0.0.0.0"
port = 8080
mode = "release"

[database]
host = "localhost"
port = 5432
dbname = "myapp"
user = "postgres"
password = "secret"

[nats]
host = "nats://localhost:4222"

[redis]
host = "localhost"
port = 6379

[mailer]
host = "smtp.gmail.com"
port = 587
tls = true
```

### Environment Variables

Environment variables use the prefix defined in `configs.NewConfig()`:

```bash
# With prefix "MYAPP"
export MYAPP_HTTP_SERVER_PORT=8080
export MYAPP_DATABASE_HOST=localhost
export MYAPP_DATABASE_PASSWORD=secret
```

---

## Dependency Injection

Modules are injected via Uber Fx parameter structs:

```go
type Params struct {
    fx.In
    Logger      *zap.Logger
    HTTPServer  *http_server.HTTPServer
    Database    database.DatabaseConnector
    NATS        *nats_connector.NATSConnector
    Redis       *redis_connector.RedisConnector
    Mailer      *mailer.Mailer
}
```

---

## Related Skills

- [project-dev](../project-dev/SKILL.md) - Project structure and application setup
- [module-dev](../module-dev/SKILL.md) - Custom module development
