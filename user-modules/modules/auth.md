# auth

JWT-based authentication module with login/logout, access token generation, refresh token rotation, and two-layer Gin middleware.

## Installation

```bash
go get github.com/weedbox/user-modules/auth
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/auth"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        auth.Module("auth"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source |
|------------|------|--------|
| `database.DatabaseConnector` | Interface | `common-modules` (no name tag) |
| `*user.UserManager` | Pointer | `user-modules/user` (`name:"user"`) |
| `*rbac.RBACManager` | Pointer | `user-modules/rbac` (`name:"rbac"`) |

## Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `{scope}.jwt_secret` | `string` | `"change-this-secret-in-production"` | HMAC secret for signing JWTs |
| `{scope}.access_token_expiry` | `duration` | `"15m"` | Access token lifetime |
| `{scope}.refresh_token_expiry` | `duration` | `"168h"` (7 days) | Refresh token lifetime |
| `{scope}.issuer` | `string` | `"weedbox"` | JWT issuer claim |
| `{scope}.mode` | `string` | `"standalone"` | Identity trust mode: `"standalone"` or `"gateway"` (see [Security: Identity Trust Modes](#security-identity-trust-modes)) |
| `{scope}.trusted_header_secret` | `string` | `""` | Gateway mode only: trust inbound `X-User-Info` only when `X-Gateway-Secret` matches |

### TOML Example

```toml
[auth]
jwt_secret = "your-production-secret-key"
access_token_expiry = "15m"
refresh_token_expiry = "168h"
issuer = "my-app"
mode = "standalone"          # or "gateway" behind a trusted ingress
# trusted_header_secret = "shared-secret-with-gateway"
```

**Important:** Always override `jwt_secret` in production. Behind a gateway that injects `X-User-Info`, set `mode = "gateway"` (see [Security: Identity Trust Modes](#security-identity-trust-modes)).

## Data Model

The `RefreshToken` model maps to the `refresh_tokens` table:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `ID` | `varchar(36)` | Primary key | UUID v7 |
| `UserID` | `varchar(36)` | Not null, indexed | Owner user ID |
| `Token` | `varchar(512)` | Unique, not null | The refresh token string |
| `ExpiresAt` | `timestamp` | Not null, indexed | Expiration time |
| `Revoked` | `bool` | Default: `false` | Whether the token has been revoked |
| `CreatedAt` | `timestamp` | Indexed | Creation time |

## API Reference

### Authentication

| Method | Signature | Description |
|--------|-----------|-------------|
| `Login` | `(ctx, identifier, password string) (*TokenPair, error)` | Authenticate and return token pair |
| `RefreshTokens` | `(ctx, refreshToken string) (*TokenPair, error)` | Refresh tokens (rotates refresh token) |
| `Logout` | `(ctx, refreshToken string) error` | Revoke a single refresh token |
| `LogoutAll` | `(ctx, userID string) error` | Revoke all refresh tokens for a user |

### Token Validation

| Method | Signature | Description |
|--------|-----------|-------------|
| `ValidateAccessToken` | `(tokenString string) (*AccessTokenClaims, error)` | Validate JWT and extract claims |

### Token Management

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetActiveRefreshTokens` | `(ctx, userID string) ([]*RefreshTokenInfo, error)` | List active refresh tokens for a user |
| `CleanupExpiredTokens` | `(ctx) (int64, error)` | Remove expired/revoked tokens from DB |

### Middleware

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetMiddleware` | `(name string) interface{}` | Get middleware by name (see below) |

## Middleware

### `"authenticate"` — Token Validation

Returns `gin.HandlerFunc`. Establishes trust in `X-User-Info`, then validates the JWT:

- **Trust step (by `mode`):** `standalone` strips any client-supplied `X-User-Info`; `gateway` keeps the upstream-injected one (only if `X-Gateway-Secret` matches when `trusted_header_secret` is set).
- No token → passes through (identity, if any, is the trusted `X-User-Info` from the trust step)
- Valid token → **overwrites** `X-User-Info` with base64-encoded session from the token
- Invalid/expired token → returns `401 Unauthorized`

```go
authMiddleware := authManager.GetMiddleware("authenticate").(gin.HandlerFunc)
router.Use(authMiddleware)
```

### `"require_permission"` — Permission Check

Returns `func(string) gin.HandlerFunc`. Reads `X-User-Info` and checks RBAC:

- Permission `"*"` → **optional auth**: populates identity into the context when a trusted `X-User-Info` is present, but never rejects (public endpoint)
- Permission `""` → requires authentication but skips RBAC (any logged-in user; use for `/me`, self-service routes)
- Non-empty permission → reads `X-User-Info`, decodes session, checks RBAC permissions
- Returns `401` if not authenticated, `403` if insufficient permissions

```go
requirePerm := authManager.GetMiddleware("require_permission").(func(string) gin.HandlerFunc)
router.GET("/users", requirePerm("user.list"), listHandler)
router.GET("/me", requirePerm(""), meHandler)        // any authenticated user
router.POST("/public", requirePerm("*"), publicHandler)
```

## Security: Identity Trust Modes

`X-User-Info` carries the caller's identity **and roles** between the two middleware layers, and `require_permission` trusts it for both authentication and RBAC. Since it is an ordinary request header, a client could forge it — so `authenticate` decides when an inbound `X-User-Info` is trustworthy via `mode`:

| Mode | Inbound `X-User-Info` | Use when |
|------|-----------------------|----------|
| `standalone` (default) | **Stripped** — identity comes only from a JWT this service validates | Service validates tokens itself (single service, direct exposure, dev) |
| `gateway` | **Trusted** (upstream-injected; optionally gated by `trusted_header_secret`) | Service sits behind a gateway that terminates auth and injects `X-User-Info` |

**`gateway` mode is only safe when:**

1. The service is reachable **only** through the gateway (network policy / mTLS).
2. The gateway **strips** any client-supplied `X-User-Info` before injecting its own.
3. Recommended: set `trusted_header_secret` and have the gateway send a matching `X-Gateway-Secret`, so the service verifies the caller is the gateway rather than trusting the network alone.

> ⚠️ Running a directly-reachable service in `gateway` mode without (1) and (2) lets any client forge identity and roles (full impersonation + privilege escalation). The default is `standalone` so this trust must be opted into explicitly.

## Context Helpers

Helper functions to extract session data from Gin context (set by `require_permission`):

| Function | Signature | Description |
|----------|-----------|-------------|
| `GetSession` | `(c *gin.Context) (*Session, bool)` | Get the full session |
| `GetUserID` | `(c *gin.Context) (string, bool)` | Get authenticated user's ID |
| `GetUsername` | `(c *gin.Context) (string, bool)` | Get authenticated user's username |
| `GetRoles` | `(c *gin.Context) ([]string, bool)` | Get authenticated user's roles |
| `HasRole` | `(c *gin.Context, role string) bool` | Check if user has a specific role |

## Types

```go
type TokenPair struct {
    AccessToken      string
    RefreshToken     string
    TokenType        string    // "Bearer"
    ExpiresIn        int64     // Access token lifetime in seconds
    ExpiresAt        time.Time
    RefreshExpiresAt time.Time
}

type Session struct {
    UserID      string
    Username    string
    Email       string
    Roles       []string
    DisplayName string
}
```

## Errors

| Error | Description |
|-------|-------------|
| `ErrInvalidCredentials` | Wrong username/email or password |
| `ErrInvalidToken` | Token is malformed or has an invalid signature |
| `ErrTokenExpired` | Token has expired |
| `ErrTokenRevoked` | Refresh token has been revoked |
| `ErrTokenNotFound` | Refresh token not found in database |
| `ErrUserInactive` | User account is not active |
| `ErrOperationFailed` | General operation failure |

## Usage in Custom Modules

```go
type Params struct {
    weedbox.Params
    Auth *auth.AuthManager `name:"auth"`  // name tag required
}

func (m *MyModule) myHandler(c *gin.Context) {
    session, ok := auth.GetSession(c)
    if !ok {
        c.JSON(401, gin.H{"error": "not authenticated"})
        return
    }

    if auth.HasRole(c, "admin") {
        // admin-specific logic
    }
}
```

## Related

- [user](./user.md) - User module (dependency)
- [rbac](./rbac.md) - RBAC module (dependency)
- [auth_apis](./auth_apis.md) - REST API endpoints for auth
- [http_token_validator](./http_token_validator.md) - Global middleware wrapper
