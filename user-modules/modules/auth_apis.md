# auth_apis

REST API endpoints for authentication. Provides Gin HTTP handlers for login, token refresh, and logout.

## Installation

```bash
go get github.com/weedbox/user-modules/auth_apis
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/auth_apis"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        auth_apis.Module("auth_apis"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source |
|------------|------|--------|
| `*http_server.HTTPServer` | Pointer | `common-modules` (no name tag) |
| `*auth.AuthManager` | Pointer | `user-modules/auth` (`name:"auth"`) |

## Endpoints

All endpoints are prefixed with `/apis/v1/auth`. All are public (permission `"*"`).

| Method | Path | Description |
|--------|------|-------------|
| POST | `/apis/v1/auth/login` | Login with username/email and password |
| POST | `/apis/v1/auth/refresh` | Refresh tokens using a refresh token |
| POST | `/apis/v1/auth/logout` | Revoke a refresh token |

## Login

```bash
curl -X POST http://localhost:8080/apis/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"identifier": "admin", "password": "1qaz@WSX"}'
```

**Response (200):**

```json
{
  "message": "Login successful",
  "token": {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "base64-encoded-token",
    "token_type": "Bearer",
    "expires_in": 900,
    "expires_at": "2025-01-01T00:15:00Z",
    "refresh_expires_at": "2025-01-08T00:00:00Z"
  },
  "user": {
    "id": "...",
    "username": "admin",
    "email": "admin@localhost",
    "display_name": "System Administrator",
    "roles": ["admin"]
  }
}
```

**Errors:** `400` (bad request), `401` (invalid credentials), `403` (user inactive), `500` (server error)

## Refresh Tokens

```bash
curl -X POST http://localhost:8080/apis/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "base64-encoded-token"}'
```

**Response (200):**

```json
{
  "message": "Token refreshed successfully",
  "token": {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "new-base64-encoded-token",
    "token_type": "Bearer",
    "expires_in": 900,
    "expires_at": "...",
    "refresh_expires_at": "..."
  }
}
```

The old refresh token is automatically revoked (token rotation).

**Errors:** `400` (bad request), `401` (invalid/expired/revoked token), `403` (user inactive), `500` (server error)

## Logout

```bash
curl -X POST http://localhost:8080/apis/v1/auth/logout \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "base64-encoded-token"}'
```

**Response (200):**

```json
{
  "message": "Logged out successfully"
}
```

**Errors:** `400` (bad request), `404` (token not found), `500` (server error)

## Security Notes

- **Token Rotation:** Each refresh operation revokes the old refresh token and issues a new one
- **Public Endpoints:** All auth endpoints are public by design — login requires no prior authentication
- **Inactive Users:** If a user's status is not `active`, login and refresh are rejected with `403`

## Related

- [auth](./auth.md) - Auth business logic module
