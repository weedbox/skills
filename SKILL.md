---
name: weedbox
description: |
  Weedbox framework development skills for building Go applications and modules.
  Use when: creating weedbox application, developing weedbox module, building weedbox package,
  working with weedbox project, using Uber Fx dependency injection, integrating common-modules,
  setting up Go microservices with weedbox, or any weedbox-based development.
  Keywords: weedbox, weedbox app, weedbox module, weedbox package, uber fx, go application, microservice.
---

# Weedbox Skills

Skills for developing applications and modules with the Weedbox framework.

## ⚠️ Critical Rules

### Check common-modules Before Creating New Modules

When user requests to "add XXX feature/module" (e.g., healthcheck, logging, database, cache):

1. **First check** [common-modules](./common-modules/SKILL.md) for existing modules
2. **If exists** → Use it directly, do NOT implement from scratch
3. **If not exists** → Then use module-dev to create new module

**Available modules in common-modules**:
- `healthcheck_apis` - Health check endpoints
- `http_server` - HTTP server
- `swagger` - API documentation
- `postgres_connector` / `sqlite_connector` - Database
- `nats_connector` - Message queue
- `redis_connector` - Cache
- `mailer` - Email sending
- `logger` - Logging
- `daemon` - Service lifecycle

---

## Available Skills

| Skill | Description |
|-------|-------------|
| [project-dev](./project-dev/SKILL.md) | Create and structure weedbox applications |
| [module-dev](./module-dev/SKILL.md) | Develop weedbox modules with Uber Fx dependency injection |
| [crud-api-dev](./crud-api-dev/SKILL.md) | Build complete CRUD APIs with business logic and HTTP handlers |
| [common-modules](./common-modules/SKILL.md) | Use built-in modules (HTTP, database, NATS, Redis, etc.) |

## When to Use

- **project-dev** - Creating new weedbox application, setting up project structure, configuring main.go and modules.go
- **module-dev** - Developing custom weedbox modules/packages, dependency injection, lifecycle hooks, configuration management, creating module skills documentation
- **crud-api-dev** - Building REST APIs with CRUD operations, request binding patterns, QueryHelper for pagination/search/filtering
- **common-modules** - Integrating configs, logger, HTTP server, database (PostgreSQL/SQLite), NATS messaging, Redis cache, or mailer

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
