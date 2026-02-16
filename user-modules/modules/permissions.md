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
