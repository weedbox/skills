# rbac

Role-based access control module powered by [privy](https://github.com/weedbox/privy). Supports extensible resource and role definitions via the Option pattern.

## Installation

```bash
go get github.com/weedbox/user-modules/rbac
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/rbac"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        rbac.Module("rbac"),
    }, nil
}
```

## Dependencies

| Dependency | Type | Source |
|------------|------|--------|
| `database.DatabaseConnector` | Interface | `common-modules` (no name tag) |

## Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `{scope}.init_default_roles` | `bool` | `true` | Whether to create default roles on startup |

### TOML Example

```toml
[rbac]
init_default_roles = true
```

## Extending with Custom Resources and Roles

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
            Actions:     permissions.CRUDActions(), // full CRUD: create, read, update, delete, list
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
            Description: "Product operator",
            Permissions: []string{"product.*"},
        },
    }),
)
```

Custom resources and roles are **merged** with the builtins (`user`, `auth`, `admin`, `user`). If a custom role key matches a builtin key, the custom definition takes precedence.

## Options

| Option | Description |
|--------|-------------|
| `WithResourceConfigs(configs []privy.ResourceConfig)` | Extra resource definitions to merge with builtins |
| `WithDefaultRoles(roles map[string]privy.RoleConfig)` | Extra role definitions to merge with builtins |

## API Reference

### Permission Checking

| Method | Signature | Description |
|--------|-----------|-------------|
| `CheckPermission` | `(roleKey, permission string) (bool, error)` | Check if a single role has a permission |
| `CheckPermissions` | `(roleKeys []string, permission string) (bool, error)` | Check if any role in a list has a permission |

### Role Management

| Method | Signature | Description |
|--------|-----------|-------------|
| `CreateRole` | `(key string, config privy.RoleConfig) (*privy.Role, error)` | Create a new role |
| `GetRole` | `(key string) (*privy.Role, error)` | Get a role by key |
| `UpdateRole` | `(key string, config privy.RoleConfig) (*privy.Role, error)` | Update an existing role |
| `ListRoles` | `() ([]privy.Role, error)` | List all roles |
| `DeleteRole` | `(key string) error` | Delete a role |
| `AssignPermissions` | `(roleKey string, permissions []string) error` | Add permissions to a role |
| `RemovePermissions` | `(roleKey string, permissions []string) error` | Remove permissions from a role |

### Resource Management

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetResource` | `(path string) (*privy.Resource, error)` | Get a resource by path |
| `ListResources` | `() ([]privy.Resource, error)` | List all top-level resources |
| `GetManager` | `() *privy.Manager` | Get the underlying privy manager |

## Usage in Custom Modules

```go
type Params struct {
    weedbox.Params
    RBAC *rbac.RBACManager `name:"rbac"`  // name tag required
}

func (m *MyModule) checkAccess(roles []string) bool {
    allowed, err := m.Params().RBAC.CheckPermissions(roles, "product.create")
    if err != nil {
        return false
    }
    return allowed
}
```

## Related

- [permissions](./permissions.md) - Permission definitions consumed by this module
- [auth](./auth.md) - Auth module that uses RBAC for permission checks
- [role_apis](./role_apis.md) - REST API handlers for role and resource management
