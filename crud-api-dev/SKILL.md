---
name: crud-api-dev
description: |
  Complete CRUD API and business logic development guide with API layer, business logic layer,
  data models, query mechanism (using queryhelper), error handling, and dependency injection.
  Use when: building REST APIs, implementing CRUD operations, creating manager modules,
  designing data models with GORM, handling pagination/search/filtering with queryhelper.
  Keywords: CRUD, REST API, Gin, GORM, queryhelper, pagination, filtering, HTTP handler.
---

# CRUD API Development Guide

This guide explains how to develop complete CRUD (Create, Read, Update, Delete) functionality, covering both API layer and business logic layer implementation patterns.

## Directory Structure

A complete CRUD feature requires two modules:

```
pkg/
├── myresource/                    # Business logic layer (Manager)
│   ├── module.go                  # Module definition and FX setup
│   ├── manager.go                 # Core CRUD business logic
│   ├── codec.go                   # Internal data structures and config
│   ├── errors.go                  # Error definitions
│   └── models/
│       └── myresource.go          # GORM database model
│
└── myresource_apis/               # API layer
    ├── module.go                  # Module definition and route registration
    ├── apis.go                    # HTTP Handler implementation
    └── codec.go                   # Request/Response structures
```

## Layer Overview

### Business Logic Layer (Manager)

Handles core business logic, database operations, and data validation.

**Key Components:**
- Database models with GORM tags
- Error definitions
- CRUD operations (Create, Get, Update, Delete, List)
- QueryHelper integration for pagination/search/filtering

For complete implementation details, see [./references/LOGIC_LAYER.md](./references/LOGIC_LAYER.md).

### API Layer

Handles HTTP routing, request binding, and response formatting.

**Key Components:**
- Request structs separated into URI/Body/Query
- Response structs
- HTTP handlers with Swagger annotations
- Gin binding with proper order

For complete implementation details, see [./references/API_LAYER.md](./references/API_LAYER.md).

---

## Quick Reference

### Request Binding Pattern

Separate request structs to avoid Gin binding conflicts:

```go
type CreateRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
}

type CreateRequestBody struct {
    Name   string `json:"name" binding:"required"`
    Status string `json:"status" binding:"omitempty,oneof=active inactive"`
}

type CreateRequest struct {
    URI  CreateRequestURI
    Body CreateRequestBody
}
```

**Binding order**: URI → Query → Body

```go
func (m *MyAPIs) create(c *gin.Context) {
    var req CreateRequest

    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    if err := c.ShouldBindJSON(&req.Body); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Use req.URI.WorkspaceID, req.Body.Name, etc.
}
```

### Module Pattern (Method 2)

```go
type Params struct {
    weedbox.Params
    HTTPServer *http_server.HTTPServer
    MyResource *myresource.MyResourceManager `name:"myresource"`
}

type MyResourceAPIs struct {
    weedbox.Module[*Params]
}

func Module(scope string) fx.Option {
    m := new(MyResourceAPIs)
    return fx.Module(scope,
        fx.Supply(fx.Annotated{Name: scope, Target: m}),
        fx.Invoke(func(p Params) { weedbox.InitModule(scope, &p, m) }),
    )
}
```

### Index Naming Convention

Always prefix index names with table name:

```go
type MyResource struct {
    Name   string `gorm:"uniqueIndex:idx_myresource_name"`
    Status string `gorm:"index:idx_myresource_status"`
}
```

---

## QueryHelper Usage

### Basic Setup

```go
qh := queryhelper.NewQueryHelper(
    queryhelper.WithPage(1),
    queryhelper.WithPageSize(20),
    queryhelper.WithSearchText("keyword"),
    queryhelper.WithSearchFields([]string{"name", "description"}),
    queryhelper.WithOrderBy([]string{"created_at"}),
    queryhelper.WithSortFactor(-1),  // 1=asc, -1=desc
    queryhelper.WithFilters(filters),
)
```

### Security Settings

```go
var QuerySettings = &queryhelper.QuerySettings{
    AllowedOrderBy: []string{"created_at", "updated_at", "name"},
    AllowedSearch:  []string{"name", "description"},
    AllowedFilters: map[string][]string{
        "status": {"=", "!=", "IN"},
        "name":   {"=", "LIKE"},
    },
}
```

### Supported Operators

| Operator | Purpose | Example |
|----------|---------|---------|
| `=` | Equal | `status = "active"` |
| `!=` | Not equal | `status != "deleted"` |
| `>`, `<`, `>=`, `<=` | Numeric comparison | `price >= 100` |
| `BETWEEN` | Range query | `created_at BETWEEN ...` |
| `IN` | List membership | `status IN ["active", "pending"]` |
| `LIKE` | Pattern matching | `name LIKE "%test%"` |

---

## Best Practices

### Naming Conventions

| Item | Convention | Example |
|------|------------|---------|
| Module name | lowercase_underscore | `myresource`, `myresource_apis` |
| Table name | lowercase_underscore_plural | `my_resources` |
| API path (list) | Plural form | `/resources` |
| API path (CRUD) | Singular form | `/resource`, `/resource/:id` |
| Index name | `idx_<table>_<column>` | `idx_myresource_status` |
| Request struct | `<Action>Request{URI,Body,Query}` | `CreateRequestURI` |

### HTTP Status Codes

| Operation | Success | Common Errors |
|-----------|---------|---------------|
| Create | 201 Created | 400, 409, 500 |
| Get | 200 OK | 404, 500 |
| Update | 200 OK | 400, 404, 500 |
| Delete | 200 OK | 404, 500 |
| List | 200 OK | 400, 500 |

### Time Format

API responses should use RFC3339 format (UTC):

```go
CreatedAt: r.CreatedAt.UTC().Format(time.RFC3339)
// Output: 2024-01-15T08:30:00Z
```

---

## Module Registration

```go
// modules.go
func loadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        // Business logic layer
        myresource.Module("myresource"),

        // API layer
        myresource_apis.Module("myresource_apis"),
    }
    return modules, nil
}
```

---

## Development Checklist

### Business Logic Layer
- [ ] Create `pkg/myresource/` directory
- [ ] Define database model `models/myresource.go`
- [ ] Define errors `errors.go`
- [ ] Define internal structures `codec.go`
- [ ] Implement module definition `module.go`
- [ ] Implement CRUD methods `manager.go`
- [ ] Configure QuerySettings
- [ ] Register module in `modules.go`

### API Layer
- [ ] Create `pkg/myresource_apis/` directory
- [ ] Define request structures with URI/Body/Query separation `codec.go`
- [ ] Define response structures `codec.go`
- [ ] Implement module definition (Method 2) `module.go`
- [ ] Implement HTTP handlers with proper binding order `apis.go`
- [ ] Add Swagger annotations
- [ ] Register module in `modules.go`
- [ ] Run `swag init .` to update docs

---

## Related Documentation

- [./references/LOGIC_LAYER.md](./references/LOGIC_LAYER.md) - Business logic layer details
- [./references/API_LAYER.md](./references/API_LAYER.md) - API layer details

## References

- QueryHelper package: `github.com/weedbox/queryhelper`
