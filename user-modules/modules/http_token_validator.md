# http_token_validator

Optional module that globally mounts the `authenticate` middleware on the HTTP server. When included, every incoming HTTP request is checked for a JWT token.

## Installation

```bash
go get github.com/weedbox/user-modules/http_token_validator
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/http_token_validator"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        http_token_validator.Module("http_token_validator"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source |
|------------|------|--------|
| `*http_server.HTTPServer` | Pointer | `common-modules` (no name tag) |
| `*auth.AuthManager` | Pointer | `user-modules/auth` (`name:"auth"`) |

## Behavior

On startup, the module:

1. Gets the `authenticate` middleware from `auth.AuthManager`
2. Registers it as global middleware on the HTTP server's router via `router.Use()`

Once mounted, **every** incoming request goes through the authenticate middleware:

- Request with `Authorization: Bearer <token>` → token is validated
  - Valid token: `X-User-Info` header is set with session data
  - Invalid/expired token: Request is rejected with `401 Unauthorized`
- Request with **no** `Authorization` header → passes through without modification

Unauthenticated requests are still allowed — it's the `require_permission` middleware on individual routes that enforces authentication.

## When to Use

**Include this module when:**
- You want all routes to automatically validate JWT tokens
- You want the `X-User-Info` header available for all downstream handlers

**Skip this module when:**
- You want to selectively apply token validation to specific route groups
- You are behind an ingress/gateway that already handles token validation
- You want to manually control middleware ordering

## Manual Alternative

If you don't use this module, manually apply the middleware:

```go
router := httpServer.GetRouter()
authMiddleware := authManager.GetMiddleware("authenticate").(gin.HandlerFunc)

apiGroup := router.Group("/apis/v1")
apiGroup.Use(authMiddleware)
```

## Related

- [auth](./auth.md) - Auth module providing the authenticate middleware
