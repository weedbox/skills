# permissions

Builtin permission definitions, role configurations, and extension API. This is a **plain Go package** (not a weedbox module) consumed by the `rbac` module.

## Installation

```bash
go get github.com/weedbox/user-modules/permissions
```

## Builtin Resources

### user

| Action | Permission String | Description |
|--------|-------------------|-------------|
| create | `user.create` | Create a new user |
| read | `user.read` | Read user details |
| update | `user.update` | Update user information |
| delete | `user.delete` | Delete a user |
| list | `user.list` | List users |

Sub-resource: **password**

| Action | Permission String | Description |
|--------|-------------------|-------------|
| update | `user.password.update` | Update user password |

### auth

| Action | Permission String | Description |
|--------|-------------------|-------------|
| login | `auth.login` | User login |
| logout | `auth.logout` | User logout |
| refresh | `auth.refresh` | Refresh token |

### role

| Action | Permission String | Description |
|--------|-------------------|-------------|
| create | `role.create` | Create a new role |
| read | `role.read` | Read role details and browse resources |
| update | `role.update` | Update role and manage permissions |
| delete | `role.delete` | Delete a role |
| list | `role.list` | List roles |

## Builtin Roles

| Role | Description | Permissions |
|------|-------------|-------------|
| `admin` | Full system access | `*` (wildcard) |
| `user` | Standard user | `auth.login`, `auth.logout`, `auth.refresh`, `user.read`, `user.password.update` |

## Permission Constants

```go
import "github.com/weedbox/user-modules/permissions"

// Resource keys
permissions.ResourceUser  // "user"
permissions.ResourceAuth  // "auth"
permissions.ResourceRole  // "role"

// User permissions
permissions.PermUserCreate         // "user.create"
permissions.PermUserRead           // "user.read"
permissions.PermUserUpdate         // "user.update"
permissions.PermUserDelete         // "user.delete"
permissions.PermUserList           // "user.list"
permissions.PermUserPasswordUpdate // "user.password.update"

// Auth permissions
permissions.PermAuthLogin   // "auth.login"
permissions.PermAuthLogout  // "auth.logout"
permissions.PermAuthRefresh // "auth.refresh"

// Role permissions
permissions.PermRoleCreate // "role.create"
permissions.PermRoleRead   // "role.read"
permissions.PermRoleUpdate // "role.update"
permissions.PermRoleDelete // "role.delete"
permissions.PermRoleList   // "role.list"
```

## Standard CRUD Actions

When defining custom resources, use `CRUDActions()` for convenience:

```go
config := privy.ResourceConfig{
    Key:     "product",
    Name:    "Product",
    Actions: permissions.CRUDActions(), // create, read, update, delete, list
}
```

## Custom Actions

When a resource only needs a subset of actions (not full CRUD), use `privy.DefineAction()` to create individual actions:

```go
config := privy.ResourceConfig{
    Key:         "article",
    Name:        "Article",
    Description: "News articles",
    Actions: []privy.Action{
        privy.DefineAction("read", "Read", "Read article details"),
        privy.DefineAction("list", "List", "List articles"),
    },
}
```

**Important**: The action type is `privy.Action`, NOT `privy.ActionConfig` (which does not exist). Always use `privy.DefineAction(key, name, description)` to construct actions.

## Extending Permissions

Use `MergeResourceConfigs` and `MergeDefaultRoles` to combine builtins with custom definitions:

```go
allResources := permissions.MergeResourceConfigs(myResourceConfigs)
allRoles := permissions.MergeDefaultRoles(myRoleConfigs)
```

These merge functions are used internally by the `rbac` module when you pass options via `rbac.WithResourceConfigs()` and `rbac.WithDefaultRoles()`.

## API Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `CRUDActions()` | `[]privy.Action` | Standard create/read/update/delete/list actions |
| `GetBuiltinResourceConfigs()` | `[]privy.ResourceConfig` | Builtin user + auth resource definitions |
| `GetBuiltinDefaultRoles()` | `map[string]privy.RoleConfig` | Builtin admin/user role definitions |
| `MergeResourceConfigs(extra)` | `[]privy.ResourceConfig` | Merge builtins with extra resources |
| `MergeDefaultRoles(extra)` | `map[string]privy.RoleConfig` | Merge builtins with extra roles |

## Related

- [rbac](./rbac.md) - RBAC module that consumes these definitions
