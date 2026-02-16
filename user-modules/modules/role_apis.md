# role_apis

REST API handlers for role and resource management. Provides CRUD operations on roles, permission assignment/removal, and resource browsing endpoints.

## Installation

```bash
go get github.com/weedbox/user-modules/role_apis
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/role_apis"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        role_apis.Module("role_apis"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source | Name Tag |
|------------|------|--------|----------|
| `*http_server.HTTPServer` | Struct pointer | `common-modules` | No |
| `*rbac.RBACManager` | Struct pointer | `user-modules` | `name:"rbac"` |
| `*auth.AuthManager` | Struct pointer | `user-modules` | `name:"auth"` |

### Params Struct

```go
type Params struct {
    weedbox.Params
    HTTPServer *http_server.HTTPServer
    RBAC       *rbac.RBACManager `name:"rbac"`
    Auth       *auth.AuthManager `name:"auth"`
}
```

## API Endpoints

### Role Management

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/apis/v1/roles` | `role.list` | List all roles |
| POST | `/apis/v1/role` | `role.create` | Create a new role |
| GET | `/apis/v1/role/:key` | `role.read` | Get role by key |
| PUT | `/apis/v1/role/:key` | `role.update` | Update a role |
| DELETE | `/apis/v1/role/:key` | `role.delete` | Delete a role |

### Permission Management

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| POST | `/apis/v1/role/:key/permissions` | `role.update` | Assign permissions to a role |
| DELETE | `/apis/v1/role/:key/permissions` | `role.update` | Remove permissions from a role |

### Resource Browsing

| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/apis/v1/resources` | `role.read` | List all top-level resources |
| GET | `/apis/v1/resource/*path` | `role.read` | Get a specific resource by path |

---

## Request / Response Structures

### Create Role

**Request** `POST /apis/v1/role`

```go
type CreateRequestBody struct {
    Key         string   `json:"key" binding:"required,min=2,max=255"`
    Name        string   `json:"name" binding:"required,min=1,max=255"`
    Description string   `json:"description"`
    Permissions []string `json:"permissions"`
}
```

**Response** `201 Created`

```json
{
  "message": "Role created successfully",
  "role": {
    "id": 3,
    "key": "editor",
    "name": "Editor",
    "description": "Content editor",
    "permissions": ["article.create", "article.update"],
    "created_at": "2024-01-15T08:30:00Z",
    "updated_at": "2024-01-15T08:30:00Z"
  }
}
```

### Get Role

**Request** `GET /apis/v1/role/:key`

**Response** `200 OK`

```json
{
  "role": {
    "id": 3,
    "key": "editor",
    "name": "Editor",
    "description": "Content editor",
    "permissions": ["article.create", "article.update"],
    "created_at": "2024-01-15T08:30:00Z",
    "updated_at": "2024-01-15T08:30:00Z"
  }
}
```

### Update Role

**Request** `PUT /apis/v1/role/:key`

```go
type UpdateRequestBody struct {
    Name        string   `json:"name" binding:"required,min=1,max=255"`
    Description string   `json:"description"`
    Permissions []string `json:"permissions"`
}
```

**Response** `200 OK`

```json
{
  "message": "Role updated successfully",
  "role": { ... }
}
```

### Delete Role

**Request** `DELETE /apis/v1/role/:key`

**Response** `200 OK`

```json
{
  "message": "Role deleted successfully"
}
```

### List Roles

**Request** `GET /apis/v1/roles`

**Response** `200 OK`

```json
{
  "roles": [
    {
      "id": 1,
      "key": "admin",
      "name": "Administrator",
      "description": "Full system access",
      "permissions": ["*"],
      "created_at": "2024-01-15T08:30:00Z",
      "updated_at": "2024-01-15T08:30:00Z"
    }
  ]
}
```

### Assign Permissions

**Request** `POST /apis/v1/role/:key/permissions`

```go
type PermissionsRequestBody struct {
    Permissions []string `json:"permissions" binding:"required,min=1"`
}
```

**Response** `200 OK`

```json
{
  "message": "Permissions assigned successfully",
  "role": { ... }
}
```

### Remove Permissions

**Request** `DELETE /apis/v1/role/:key/permissions`

```go
type PermissionsRequestBody struct {
    Permissions []string `json:"permissions" binding:"required,min=1"`
}
```

**Response** `200 OK`

```json
{
  "message": "Permissions removed successfully",
  "role": { ... }
}
```

### List Resources

**Request** `GET /apis/v1/resources`

**Response** `200 OK`

```json
{
  "resources": [
    {
      "key": "user",
      "name": "User",
      "description": "User management",
      "actions": [
        { "key": "create", "name": "Create", "description": "Create a new user" },
        { "key": "read", "name": "Read", "description": "Read user details" }
      ],
      "sub_resources": [
        {
          "key": "password",
          "name": "Password",
          "description": "User password management",
          "actions": [
            { "key": "update", "name": "Update", "description": "Update user password" }
          ]
        }
      ]
    }
  ]
}
```

### Get Resource

**Request** `GET /apis/v1/resource/*path`

**Response** `200 OK`

```json
{
  "resource": {
    "key": "user",
    "name": "User",
    "description": "User management",
    "actions": [ ... ],
    "sub_resources": [ ... ]
  }
}
```

---

## Error Handling

| Error | HTTP Status | Condition |
|-------|-------------|-----------|
| `privy.ErrRoleExists` | 409 Conflict | Role key already exists (create) |
| `privy.ErrRoleNotFound` | 404 Not Found | Role key not found (get/update/delete) |
| `privy.ErrResourceNotFound` | 404 Not Found | Resource path not found (get resource) |
| Binding error | 400 Bad Request | Invalid request body or URI parameters |
| Other errors | 500 Internal Server Error | Unexpected server errors |

## Related

- [rbac](./rbac.md) - RBAC module providing the underlying role/resource management
- [permissions](./permissions.md) - Permission definitions and constants
- [auth](./auth.md) - Auth module providing the permission middleware
