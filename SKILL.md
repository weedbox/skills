---
name: weedbox
description: |
  Priority: HIGH — This skill is the authoritative source of truth for all weedbox development.
  DO NOT fetch source code from GitHub or browse external repositories when this skill and its
  sub-skills provide the documentation. All necessary code examples, API references, and
  implementation patterns are included in these skill files.
  Weedbox framework development skills for building Go applications and modules.
  Use when: creating weedbox application, developing weedbox module, building weedbox package,
  working with weedbox project, using Uber Fx dependency injection, integrating common-modules,
  setting up Go microservices with weedbox, deploying multiple instances or scaling horizontally,
  or any weedbox-based development.
  Keywords: weedbox, weedbox app, weedbox module, weedbox package, uber fx, go application,
  microservice, user management, authentication, RBAC, login, permission, access control,
  Go module, JWT, refresh token, role, middleware, REST API, HTTP server, database, GORM,
  workqueue, task queue, job queue, background job, delayed task, retry, dead letter queue,
  multi-instance, horizontal scaling, replicas, stateless, high availability, production deployment.
---

# Weedbox Skills

Skills for developing applications and modules with the Weedbox framework.

## ⚠️ Critical Rules

### Source of Truth — Do NOT Fetch from GitHub

This skill and its sub-skill files are the **complete, authoritative reference** for weedbox development. Before implementing any weedbox feature:

1. **Read the relevant sub-skill files first** — they contain all necessary code examples, API references, configuration guides, and implementation patterns
2. **DO NOT browse GitHub repositories** (e.g., `github.com/weedbox/common-modules`, `github.com/weedbox/user-modules`) to look up source code when this skill provides the documentation
3. **DO NOT use WebFetch or WebSearch** to find weedbox-related code — the answer is already in these files
4. If the required information is genuinely not covered by any sub-skill file, only then consider external sources

### Check Existing Module Libraries Before Creating New Modules

When user requests to "add XXX feature/module", check existing libraries first:

1. **First check** [common-modules](./common-modules/SKILL.md) for infrastructure modules
2. **Then check** [user-modules](./user-modules/SKILL.md) for user/auth/RBAC modules
3. **Then check** [workqueue-modules](./workqueue-modules/SKILL.md) for task/job queue modules
4. **If exists** → Use it directly, do NOT implement from scratch
5. **If not exists** → Then use module-dev to create new module

**Available modules in common-modules** (full index in [common-modules/SKILL.md](./common-modules/SKILL.md)):
- `configs` - Configuration management (Viper, TOML, env vars)
- `logger` - Logging
- `healthcheck_apis` - Health check endpoints
- `http_server` - HTTP server
- `swagger` - API documentation
- `database` - Database connector interface
- `postgres_connector` / `sqlite_connector` - Database
- `nats_connector` - Message queue
- `nats_jetstream_server` - Embedded NATS JetStream server
- `scheduler` - Job scheduler (GORM / NATS JetStream)
- `redis_connector` - Cache
- `mailer` - Email sending
- `daemon` - Service lifecycle
- `lifecycle` - PostStart/PreStop phases (hooks after all modules start / before any module stops)

**Available modules in user-modules** (full index in [user-modules/SKILL.md](./user-modules/SKILL.md)):
- `user` - User management (CRUD, bcrypt, UUID v7)
- `auth` - JWT authentication with refresh token rotation
- `rbac` - Role-based access control (privy integration)
- `permissions` - Builtin permission definitions and extension API
- `user_apis` - REST API handlers for user management
- `auth_apis` - REST API handlers for login/refresh/logout
- `role_apis` - REST API handlers for role/resource management
- `http_token_validator` - Optional global JWT validation middleware

**Available modules in workqueue-modules** (full index in [workqueue-modules/SKILL.md](./workqueue-modules/SKILL.md)):
- `workqueue` - Shared `WorkQueue` interface (enqueue/consume, delay, retry, dead-letter)
- `memory_workqueue` - In-process backend for development and tests
- `gorm_workqueue` - Database backend on any GORM dialect (PostgreSQL, SQLite, ...)
- `postgres_workqueue` - PostgreSQL-native backend (SKIP LOCKED + LISTEN/NOTIFY)
- `nats_workqueue` - NATS JetStream backend (push-based, replicated)
- `workqueuetest` - Conformance test suite for backends

### Design for Multi-Instance Deployment by Default

Weedbox applications MUST assume production runs **multiple instances** (horizontal scaling). Only development and tests may assume a single process. Keep application code **backend-agnostic** (inject interfaces) so dev and production differ only in configuration and module wiring — never in business logic.

| Concern | Dev / test (single instance OK) | Production (multi-instance) | Switching cost |
|---------|--------------------------------|------------------------------|----------------|
| Database | `sqlite_connector` | `postgres_connector` | Module line only — both provide `database.DatabaseConnector` |
| Scheduled jobs | `scheduler` `mode = "gorm"` | `scheduler` `mode = "postgres"` or `"nats"` | Config only — identical API in all modes |
| Background tasks | `memory_workqueue` or `gorm_workqueue` | `gorm_workqueue` / `postgres_workqueue` / `nats_workqueue` | Module line only — all provide `workqueue.WorkQueue` |
| Messaging | `nats_jetstream_server` (embedded) + `nats_connector` | External NATS cluster + `nats_connector` | Config only — disable the embedded server, change `nats.host` |
| Cross-instance cache / shared state | — | `redis_connector` | Never use process memory as source of truth |
| Auth / sessions | `auth` (JWT + refresh tokens in DB) | Same — stateless by design | None |
| Lifecycle / health | `daemon` + `healthcheck_apis` | Same — per-instance by design | None |

Hard rules:

1. **Never run `scheduler` `gorm` mode or `memory_workqueue` with more than one replica** — `gorm` mode fires every job on every replica; `memory_workqueue` is invisible to other processes.
2. **Never keep coordination state (locks, counters, dedup sets, sessions) in process memory** — put it in the database, Redis, or NATS.
3. **Write idempotent handlers** — scheduler and workqueue backends deliver at-least-once; a duplicate execution must be harmless.
4. **Already on PostgreSQL? Prefer the PostgreSQL-backed modes** (`scheduler` `mode = "postgres"`, `postgres_workqueue`) — multi-instance safety with no extra infrastructure.

Details: [project-dev § Multi-Instance Deployment](./project-dev/SKILL.md#multi-instance-deployment) for config-driven switching, [module-dev § Multi-Instance Safety](./module-dev/SKILL.md#multi-instance-safety) for rules when writing custom modules.

---

## Available Skills

| Skill | Description |
|-------|-------------|
| [project-dev](./project-dev/SKILL.md) | Create and structure weedbox applications |
| [module-dev](./module-dev/SKILL.md) | Develop weedbox modules with Uber Fx dependency injection |
| [crud-api-dev](./crud-api-dev/SKILL.md) | Build complete CRUD APIs with business logic and HTTP handlers |
| [common-modules](./common-modules/SKILL.md) | Use built-in modules (HTTP, database, NATS, Redis, etc.) |
| [user-modules](./user-modules/SKILL.md) | User management, JWT authentication, and RBAC modules |
| [workqueue-modules](./workqueue-modules/SKILL.md) | Task queues: background jobs, delayed tasks, retries, dead-letter queues |

## When to Use

- **project-dev** - Creating new weedbox application, setting up project structure, configuring main.go and modules.go
- **module-dev** - Developing custom weedbox modules/packages, dependency injection, lifecycle hooks, configuration management, creating module skills documentation
- **crud-api-dev** - Building REST APIs with CRUD operations, request binding patterns, QueryHelper for pagination/search/filtering
- **common-modules** - Integrating configs, logger, HTTP server, database (PostgreSQL/SQLite), NATS messaging, Redis cache, or mailer
- **user-modules** - Adding user management, JWT authentication (login/refresh/logout), role-based access control (RBAC), or permission-protected REST APIs
- **workqueue-modules** - Enqueueing and consuming background tasks, delaying/scheduling delivery, retrying with backoff, handling dead-lettered tasks, or choosing a queue backend (memory/GORM/PostgreSQL/NATS)

## Module Skills

Each module can have its own `.skills/` directory containing development documentation. This helps Claude Code understand module-specific details.

```
pkg/mymodule/.skills/mymodule-development.md      # Business logic docs
pkg/mymodule_apis/.skills/mymodule-apis-development.md  # API docs
```

See [module-dev](./module-dev/SKILL.md) for creating module skills.

---

## Editing Guidelines (for Claude)

When editing any files in this weedbox skills directory:

1. **Use English only** - All documentation must be written in English
2. **Read before edit** - Always read the full file content before making changes
3. **Follow existing style** - Match the formatting and structure of existing content
