---
name: user-modules
description: |
  Priority: HIGH — Self-contained reference. This file and its sub-files (modules/*.md) contain
  ALL necessary code examples, API references, and implementation patterns for user-modules.
  DO NOT fetch source code from GitHub — everything needed is documented here.
  Reference guide for weedbox/user-modules - reusable modules for user management, authentication, and RBAC.
  Use when: adding user management, JWT authentication (login/refresh/logout), role-based access control,
  permission-protected REST APIs, password management, or token validation middleware to weedbox projects.
  Covers: user CRUD, bcrypt password hashing, JWT access/refresh tokens, RBAC with privy,
  extensible permissions, REST API endpoints, two-layer auth middleware.
  Keywords: user-modules, user management, authentication, JWT, refresh token, RBAC, permissions,
  login, logout, password, access control, middleware, privy, bcrypt, UUID v7,
  user CRUD, role, session, token rotation, auth middleware, 使用者, 權限, 登入,
  user module, auth module, rbac module, permission check, protected API.
---

# User Modules Reference

This skill provides usage instructions for all modules in `github.com/weedbox/user-modules`.

## Module Index

### Core

| Module | Description | Documentation |
|--------|-------------|---------------|
| `permissions` | Builtin permission definitions and extension API (plain Go package, not a module) | [permissions.md](./modules/permissions.md) |
| `rbac` | RBAC manager with privy integration, extensible via Option pattern | [rbac.md](./modules/rbac.md) |
| `user` | User CRUD, bcrypt password hashing, UUID v7 IDs, pagination/search | [user.md](./modules/user.md) |
| `auth` | JWT access tokens, refresh token rotation, two-layer middleware | [auth.md](./modules/auth.md) |

### API

| Module | Description | Documentation |
|--------|-------------|---------------|
| `user_apis` | REST API handlers for user management | [user_apis.md](./modules/user_apis.md) |
| `auth_apis` | REST API handlers for login/refresh/logout | [auth_apis.md](./modules/auth_apis.md) |

### Optional

| Module | Description | Documentation |
|--------|-------------|---------------|
| `http_token_validator` | Global JWT validation middleware for HTTP server | [http_token_validator.md](./modules/http_token_validator.md) |

---

## ⚠️ Critical Rules

### Source of Truth — Do NOT Fetch from GitHub

This file and its sub-files (`modules/*.md`) are the **complete, authoritative reference** for `weedbox/user-modules`. They contain all necessary implementation details including code examples, API endpoints, configuration, dependency injection patterns, and middleware usage. **DO NOT browse `github.com/weedbox/user-modules`** to look up source code — the answer is already here.

### Dependency Injection: name Tags Required

All user-modules use Method 2 (weedbox generic). When injecting them in your Params, you **MUST use `name` tags**:

```go
// ✅ Correct - name tags required for user-modules
type Params struct {
    weedbox.Params
    Database database.DatabaseConnector           // common-module: NO name tag
    User     *user.UserManager    `name:"user"`   // user-module: name tag required
    RBAC     *rbac.RBACManager    `name:"rbac"`   // user-module: name tag required
    Auth     *auth.AuthManager    `name:"auth"`   // user-module: name tag required
}

// ❌ Wrong - missing name tags
type Params struct {
    weedbox.Params
    User *user.UserManager
    Auth *auth.AuthManager
}
```

### Module Loading Order

User-modules must be loaded in the correct order due to dependencies:

```go
// ✅ Correct order
user.Module("user"),       // 1. User first (no user-module deps)
rbac.Module("rbac"),       // 2. RBAC second (no user-module deps)
auth.Module("auth"),       // 3. Auth third (depends on user + rbac)

// API modules (depend on auth)
user_apis.Module("user_apis"),
auth_apis.Module("auth_apis"),
http_token_validator.Module("http_token_validator"),  // optional
```

---

## Quick Start

### Installation

```bash
go get github.com/weedbox/user-modules
```

### Loading Modules

Add user-modules in the `loadModules()` phase of your `modules.go`:

```go
import (
    "github.com/weedbox/user-modules/auth"
    "github.com/weedbox/user-modules/auth_apis"
    "github.com/weedbox/user-modules/http_token_validator"
    "github.com/weedbox/user-modules/rbac"
    "github.com/weedbox/user-modules/user"
    "github.com/weedbox/user-modules/user_apis"
)

func loadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        // Infrastructure (from common-modules)
        http_server.Module("http_server"),
        sqlite_connector.Module("database"),

        // User modules
        user.Module("user"),
        rbac.Module("rbac"),
        auth.Module("auth"),

        // API modules
        http_token_validator.Module("http_token_validator"),
        user_apis.Module("user_apis"),
        auth_apis.Module("auth_apis"),
    }
    return modules, nil
}
```

This gives you:
- A default `admin` user (username: `admin`, password: `1qaz@WSX`)
- Two builtin roles: `admin`, `user`
- JWT-based authentication with access/refresh token pairs
- REST API endpoints for user management and authentication
- Global token validation on all HTTP routes

### Configuration

```toml
[user]
bcrypt_cost = 12
min_password_length = 8
create_default_admin = true
default_admin_password = "your-secure-password"

[rbac]
init_default_roles = true

[auth]
jwt_secret = "your-production-secret-key"
access_token_expiry = "15m"
refresh_token_expiry = "168h"
issuer = "my-app"
```

---

## Module Dependency Graph

```
database.DatabaseConnector (from common-modules)
    |
    +---> user.UserManager
    |         |
    +---> rbac.RBACManager
    |         |
    +---> auth.AuthManager <--- user.UserManager + rbac.RBACManager
              |
              +---> user_apis (+ http_server + user.UserManager)
              +---> auth_apis (+ http_server)
              +---> http_token_validator (+ http_server) [optional]
```

---

## API Endpoints

### Authentication (`auth_apis`)

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/apis/v1/auth/login` | Public | Login with username/email and password |
| POST | `/apis/v1/auth/refresh` | Public | Refresh tokens using a refresh token |
| POST | `/apis/v1/auth/logout` | Public | Revoke a refresh token |

### User Management (`user_apis`)

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/apis/v1/users` | `user.list` | List users with pagination/search |
| POST | `/apis/v1/user` | `user.create` | Create a new user |
| GET | `/apis/v1/user/:id` | `user.read` | Get user details |
| PUT | `/apis/v1/user/:id` | `user.update` | Update user information |
| DELETE | `/apis/v1/user/:id` | `user.delete` | Delete a user |
| PUT | `/apis/v1/user/:id/password` | `user.password.update` | Update user password |
| POST | `/apis/v1/user/authenticate` | `user.read` | Authenticate credentials |

---

## Extending Permissions

The builtin permissions cover `user` and `auth` resources. To add your own, use the Option pattern on `rbac.Module`:

```go
import (
    "github.com/weedbox/privy"
    "github.com/weedbox/user-modules/permissions"
    "github.com/weedbox/user-modules/rbac"
)

rbac.Module("rbac",
    rbac.WithResourceConfigs([]privy.ResourceConfig{
        {
            Key:         "product",
            Name:        "Product",
            Description: "Product management",
            Actions:     permissions.CRUDActions(), // full CRUD
        },
        {
            Key:         "report",
            Name:        "Report",
            Description: "Report browsing",
            Actions: []privy.Action{  // custom subset using privy.DefineAction()
                privy.DefineAction("read", "Read", "Read report details"),
                privy.DefineAction("list", "List", "List reports"),
            },
        },
    }),
    rbac.WithDefaultRoles(map[string]privy.RoleConfig{
        "operator": {
            Name:        "Operator",
            Description: "Can manage products",
            Permissions: []string{"product.*"},
        },
    }),
)
```

Custom resources and roles are **merged** with the builtins. See [permissions.md](./modules/permissions.md) for details.

---

## Authentication Flow

```
Client                    Server
  |                         |
  |-- POST /auth/login ---->|  (username + password)
  |<-- access + refresh ----|
  |                         |
  |-- GET /users ---------->|  (Authorization: Bearer <access_token>)
  |   [authenticate MW]     |  -> validates JWT, sets X-User-Info header
  |   [require_permission]  |  -> reads X-User-Info, checks RBAC
  |<-- 200 OK --------------|
  |                         |
  |-- POST /auth/refresh -->|  (refresh_token)
  |<-- new access+refresh --|  (old refresh token is revoked)
  |                         |
  |-- POST /auth/logout --->|  (refresh_token)
  |<-- 200 OK --------------|  (refresh token is revoked)
```

The two-layer middleware design (`authenticate` + `require_permission`) supports ingress/gateway architectures where token validation happens at the edge and user info is forwarded via the `X-User-Info` header.

---

## Using Auth Middleware in Custom Modules

When building custom API modules that need permission protection:

```go
type Params struct {
    weedbox.Params
    HTTPServer *http_server.HTTPServer
    Auth       *auth.AuthManager `name:"auth"`
}

func (m *MyAPIs) OnStart(ctx context.Context) error {
    router := m.Params().HTTPServer.GetRouter().Group("/apis/v1")

    requirePerm := m.Params().Auth.GetMiddleware("require_permission").(func(string) gin.HandlerFunc)

    router.GET("/products", requirePerm("product.list"), m.listProducts)
    router.POST("/product", requirePerm("product.create"), m.createProduct)
    router.POST("/public", requirePerm("*"), m.publicHandler)  // "*" = no auth required

    return nil
}

func (m *MyAPIs) listProducts(c *gin.Context) {
    // Get authenticated user info from context
    session, ok := auth.GetSession(c)
    if ok {
        fmt.Printf("User: %s, Roles: %v\n", session.Username, session.Roles)
    }

    if auth.HasRole(c, "admin") {
        // admin-specific logic
    }
}
```

### Context Helpers

| Function | Returns | Description |
|----------|---------|-------------|
| `auth.GetSession(c)` | `(*Session, bool)` | Full session data |
| `auth.GetUserID(c)` | `(string, bool)` | Authenticated user's ID |
| `auth.GetUsername(c)` | `(string, bool)` | Authenticated user's username |
| `auth.GetRoles(c)` | `([]string, bool)` | Authenticated user's roles |
| `auth.HasRole(c, role)` | `bool` | Check if user has a specific role |

---

## Related Skills

- [common-modules](../common-modules/SKILL.md) - Infrastructure modules (HTTP server, database, etc.)
- [module-dev](../module-dev/SKILL.md) - Custom module development
- [crud-api-dev](../crud-api-dev/SKILL.md) - CRUD API patterns
