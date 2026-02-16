# user_apis

REST API endpoints for user management. Provides Gin HTTP handlers for CRUD operations, password management, and user authentication.

## Installation

```bash
go get github.com/weedbox/user-modules/user_apis
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/user_apis"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        user_apis.Module("user_apis"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source |
|------------|------|--------|
| `*http_server.HTTPServer` | Pointer | `common-modules` (no name tag) |
| `*user.UserManager` | Pointer | `user-modules/user` (`name:"user"`) |
| `*auth.AuthManager` | Pointer | `user-modules/auth` (`name:"auth"`) |

## Endpoints

All endpoints are prefixed with `/apis/v1`.

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/apis/v1/users` | `user.list` | List users with pagination/search |
| POST | `/apis/v1/user` | `user.create` | Create a new user |
| GET | `/apis/v1/user/:id` | `user.read` | Get user details |
| PUT | `/apis/v1/user/:id` | `user.update` | Update user information |
| DELETE | `/apis/v1/user/:id` | `user.delete` | Delete a user |
| PUT | `/apis/v1/user/:id/password` | `user.password.update` | Update user password |
| POST | `/apis/v1/user/authenticate` | `user.read` | Authenticate credentials |

## Request/Response Examples

### Create User

```bash
curl -X POST http://localhost:8080/apis/v1/user \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john",
    "email": "john@example.com",
    "password": "secure-password",
    "display_name": "John Doe",
    "roles": ["user"],
    "status": "active"
  }'
```

| Field | Required | Validation | Description |
|-------|----------|------------|-------------|
| `username` | Yes | min=3, max=255 | Unique username |
| `email` | Yes | Valid email | Unique email |
| `password` | Yes | min=8 | Password (will be hashed) |
| `display_name` | No | | Display name |
| `roles` | No | | Role keys (default: `["user"]`) |
| `status` | No | `active`/`inactive`/`suspended` | Status (default: `active`) |

### List Users

```bash
curl "http://localhost:8080/apis/v1/users?page=1&page_size=10&keywords=john&status=active" \
  -H "Authorization: Bearer <token>"
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | `int` | `1` | Page number |
| `page_size` | `int` | `10` | Items per page |
| `keywords` | `string` | | Search text (username, email, display_name) |
| `search_fields` | `string` | | Comma-separated fields to search |
| `orderby` | `string` | | Comma-separated fields to order by |
| `order` | `int` | `-1` | Sort order: `1` = asc, `-1` = desc |
| `status` | `string` | | Filter by status |
| `role` | `string` | | Filter by role |

### Update Password

```bash
curl -X PUT http://localhost:8080/apis/v1/user/<id>/password \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "old-password",
    "new_password": "new-secure-password"
  }'
```

## Error Responses

| Status | Condition |
|--------|-----------|
| `400` | Invalid request body or parameters |
| `401` | Authentication required or invalid credentials |
| `403` | Insufficient permissions |
| `404` | User not found |
| `409` | Username or email already exists |
| `500` | Internal server error |

## Related

- [user](./user.md) - User business logic module
- [auth](./auth.md) - Auth module providing permission middleware
