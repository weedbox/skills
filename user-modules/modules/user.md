# user

User management module with CRUD operations, bcrypt password hashing, and paginated listing with search and filtering.

## Installation

```bash
go get github.com/weedbox/user-modules/user
```

## Usage in Fx

```go
import "github.com/weedbox/user-modules/user"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        user.Module("user"),
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
| `{scope}.max_page_size` | `int` | `100` | Maximum page size for list queries |
| `{scope}.bcrypt_cost` | `int` | `12` | bcrypt cost factor (valid range: 10-14) |
| `{scope}.min_password_length` | `int` | `8` | Minimum password length |
| `{scope}.create_default_admin` | `bool` | `true` | Whether to create a default admin user on startup |
| `{scope}.default_admin_password` | `string` | `"1qaz@WSX"` | Default admin password (override in production) |

### TOML Example

```toml
[user]
bcrypt_cost = 12
min_password_length = 8
create_default_admin = true
default_admin_password = "your-secure-password"
```

## Default Admin User

When `create_default_admin` is `true`, the module creates an admin user on first startup:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Email | `admin@localhost` |
| Password | Value of `default_admin_password` config |
| Display Name | `System Administrator` |
| Roles | `["admin"]` |
| Status | `active` |

## Data Model

The `User` model maps to the `users` table:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `ID` | `varchar(36)` | Primary key | UUID v7 |
| `Username` | `varchar(255)` | Unique, not null | Login username |
| `Email` | `varchar(255)` | Unique | Email address |
| `PasswordHash` | `varchar(255)` | Not null | bcrypt hash (never exposed) |
| `DisplayName` | `varchar(255)` | | Display name |
| `Roles` | `text` (JSON) | | JSON array of role keys |
| `Status` | `varchar(50)` | Default: `active` | `active`, `inactive`, `suspended` |
| `LastLoginAt` | `timestamp` | Nullable | Last successful login time |
| `CreatedAt` | `timestamp` | | Record creation time |
| `UpdatedAt` | `timestamp` | | Last update time |

## API Reference

### CRUD

| Method | Signature | Description |
|--------|-----------|-------------|
| `Create` | `(ctx, cfg *UserConfig) (*User, error)` | Create a user with hashed password |
| `Get` | `(ctx, userID string) (*User, error)` | Get user by ID |
| `GetByUsername` | `(ctx, username string) (*User, error)` | Get user by username |
| `GetByEmail` | `(ctx, email string) (*User, error)` | Get user by email |
| `Update` | `(ctx, userID string, cfg *UserConfig) (*User, error)` | Update user (partial, non-empty fields only) |
| `Delete` | `(ctx, userID string) error` | Delete a user |
| `List` | `(ctx, req *ListUsersRequest, qh *queryhelper.QueryHelper) (*ListUsersResp, error)` | List with pagination/search/filter |

### Password

| Method | Signature | Description |
|--------|-----------|-------------|
| `UpdatePassword` | `(ctx, userID string, newPassword string) error` | Update password (validates min length) |
| `VerifyPassword` | `(ctx, userID string, password string) error` | Verify a password against stored hash |

### Authentication

| Method | Signature | Description |
|--------|-----------|-------------|
| `Authenticate` | `(ctx, identifier string, password string) (*User, error)` | Authenticate by username or email + password |

## Types

```go
type UserConfig struct {
    Username    string
    Email       string
    Password    string   // Plain text (will be hashed)
    DisplayName string
    Roles       []string
    Status      string
}

type User struct {
    ID          string
    Username    string
    Email       string
    DisplayName string
    Roles       []string
    Status      string
    LastLoginAt *time.Time
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

type ListUsersRequest struct {
    Username *string
    Email    *string
    Role     *string
    Status   *string
}
```

## Errors

| Error | Description |
|-------|-------------|
| `ErrNotFound` | User not found |
| `ErrUsernameExists` | Username already taken |
| `ErrEmailExists` | Email already taken |
| `ErrInvalidInput` | Invalid input data |
| `ErrInvalidPassword` | Password verification failed |
| `ErrInvalidCredentials` | Authentication failed |
| `ErrPasswordTooShort` | Password shorter than minimum length |
| `ErrOperationFailed` | General operation failure |

## Query Settings

The `List` method supports pagination, search, and sorting via queryhelper:

- **Allowed order by:** `created_at`, `updated_at`, `username`, `email`, `last_login_at`
- **Allowed search fields:** `username`, `email`, `display_name`
- **Allowed filters:** `status` (=, !=, IN), `role` (=, !=, IN), `username` (=, LIKE), `email` (=, LIKE)

## Usage in Custom Modules

```go
type Params struct {
    weedbox.Params
    User *user.UserManager `name:"user"`  // name tag required
}

func (m *MyModule) createUser(ctx context.Context) error {
    u, err := m.Params().User.Create(ctx, &user.UserConfig{
        Username:    "john",
        Email:       "john@example.com",
        Password:    "secure-password",
        DisplayName: "John Doe",
        Roles:       []string{"user"},
    })
    // ...
}
```

## Related

- [auth](./auth.md) - Auth module that depends on UserManager
- [user_apis](./user_apis.md) - REST API endpoints for user management
