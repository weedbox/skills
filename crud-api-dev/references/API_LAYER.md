# API Layer

This document covers the implementation of the HTTP API layer for CRUD operations using Gin framework.

## Directory Structure

```
pkg/myresource_apis/
├── module.go     # Module definition and route registration
├── apis.go       # HTTP Handler implementation
└── codec.go      # Request/Response structures
```

## 1. Request/Response Structures (`codec.go`)

**Important**: Separate request structs into URI, Body, and Query components to avoid Gin binding conflicts.

```go
package myresource_apis

// ========== Request Structures ==========
// Pattern: Separate URI, Body, Query structs wrapped in a parent Request struct
// This prevents Gin's ShouldBind methods from overwriting each other

// --- Create ---

type CreateRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
}

type CreateRequestBody struct {
    Name        string `json:"name" binding:"required,min=1,max=255"`
    Description string `json:"description"`
    Status      string `json:"status" binding:"omitempty,oneof=active inactive"`
}

type CreateRequest struct {
    URI  CreateRequestURI
    Body CreateRequestBody
}

// --- Get ---

type GetRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
    ID          string `uri:"id" binding:"required"`
}

type GetRequest struct {
    URI GetRequestURI
}

// --- Update ---

type UpdateRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
    ID          string `uri:"id" binding:"required"`
}

type UpdateRequestBody struct {
    Description string `json:"description"`
    Status      string `json:"status" binding:"omitempty,oneof=active inactive"`
}

type UpdateRequest struct {
    URI  UpdateRequestURI
    Body UpdateRequestBody
}

// --- Delete ---

type DeleteRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
    ID          string `uri:"id" binding:"required"`
}

type DeleteRequest struct {
    URI DeleteRequestURI
}

// --- List ---

type ListRequestURI struct {
    WorkspaceID string `uri:"workspace_id" binding:"required"`
}

type ListRequestQuery struct {
    Page         int    `form:"page"`
    PageSize     int    `form:"page_size"`
    Keywords     string `form:"keywords"`
    SearchFields string `form:"search_fields"`
    OrderBy      string `form:"orderby"`
    Order        int    `form:"order"`
    Status       string `form:"status"`
}

type ListRequest struct {
    URI   ListRequestURI
    Query ListRequestQuery
}

// ========== Response Structures ==========

// ResourceEntry resource item in API response
type ResourceEntry struct {
    ID          string `json:"id"`
    Name        string `json:"name"`
    Description string `json:"description"`
    Status      string `json:"status"`
    WorkspaceID string `json:"workspace_id"`
    CreatedAt   string `json:"created_at"`
    UpdatedAt   string `json:"updated_at"`
}

// CreateResponse create response
type CreateResponse struct {
    Message  string         `json:"message"`
    Resource *ResourceEntry `json:"resource"`
}

// GetResponse get response
type GetResponse struct {
    Resource *ResourceEntry `json:"resource"`
}

// UpdateResponse update response
type UpdateResponse struct {
    Message  string         `json:"message"`
    Resource *ResourceEntry `json:"resource"`
}

// DeleteResponse delete response
type DeleteResponse struct {
    Message string `json:"message"`
}

// ListResponse list response
type ListResponse struct {
    Total      int64            `json:"total"`
    Page       int              `json:"page"`
    PageSize   int              `json:"page_size"`
    TotalPages int              `json:"total_pages"`
    OrderBy    []string         `json:"order_by"`
    Order      int              `json:"order"`
    Keywords   string           `json:"keywords,omitempty"`
    Resources  []*ResourceEntry `json:"resources"`
}

// ErrorResponse error response
type ErrorResponse struct {
    Error string `json:"error" example:"error message"`
}
```

## 2. Module Definition (`module.go`)

Using Weedbox Module Method 2 (recommended):

```go
package myresource_apis

import (
    "context"

    "github.com/weedbox/common-modules/http_server"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"

    "your-project/pkg/myresource"
)

const ModuleName = "myresource_apis"

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

    return fx.Module(
        scope,
        fx.Supply(fx.Annotated{Name: scope, Target: m}),
        fx.Invoke(func(p Params) {
            weedbox.InitModule(scope, &p, m)
        }),
    )
}

func (m *MyResourceAPIs) InitDefaultConfigs() {
    // API-specific configs if needed
}

func (m *MyResourceAPIs) OnStart(ctx context.Context) error {
    m.Logger().Info("Starting " + ModuleName)

    // Register routes
    router := m.Params().HTTPServer.GetRouter().Group("/apis/v1/w/:workspace_id")

    // List (plural form)
    router.GET("/resources", m.list)

    // CRUD (singular form)
    router.POST("/resource", m.create)
    router.GET("/resource/:id", m.get)
    router.PUT("/resource/:id", m.update)
    router.DELETE("/resource/:id", m.delete)

    m.Logger().Info("Started " + ModuleName)
    return nil
}

func (m *MyResourceAPIs) OnStop(ctx context.Context) error {
    m.Logger().Info("Stopped " + ModuleName)
    return nil
}
```

## 3. HTTP Handlers (`apis.go`)

```go
package myresource_apis

import (
    "net/http"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/weedbox/queryhelper"
    "go.uber.org/zap"

    "your-project/pkg/myresource"
)

// create creates a resource
//
//	@Summary		Create resource
//	@Description	Create a new resource in the specified workspace
//	@Tags			Resources
//	@Accept			json
//	@Produce		json
//	@Param			workspace_id	path		string				true	"Workspace ID"
//	@Param			body			body		CreateRequestBody	true	"Create request"
//	@Success		201				{object}	CreateResponse
//	@Failure		400				{object}	ErrorResponse
//	@Failure		409				{object}	ErrorResponse
//	@Failure		500				{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/w/{workspace_id}/resource [post]
func (m *MyResourceAPIs) create(c *gin.Context) {
    var req CreateRequest

    // Bind URI parameters
    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Bind JSON body
    if err := c.ShouldBindJSON(&req.Body); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Build config
    cfg := &myresource.ResourceConfig{
        Name:        req.Body.Name,
        Description: req.Body.Description,
        Status:      req.Body.Status,
    }

    // Call business layer
    ctx := c.Request.Context()
    resource, err := m.Params().MyResource.Create(ctx, req.URI.WorkspaceID, cfg)
    if err != nil {
        if err == myresource.ErrNameExists {
            c.JSON(http.StatusConflict, gin.H{"error": "Resource name already exists"})
            return
        }

        m.Logger().Error("Failed to create resource", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // Return response
    c.JSON(http.StatusCreated, CreateResponse{
        Message:  "resource created successfully",
        Resource: m.toEntry(resource),
    })
}

// get retrieves a resource
//
//	@Summary		Get resource
//	@Description	Get a resource by ID
//	@Tags			Resources
//	@Produce		json
//	@Param			workspace_id	path		string	true	"Workspace ID"
//	@Param			id				path		string	true	"Resource ID"
//	@Success		200				{object}	GetResponse
//	@Failure		400				{object}	ErrorResponse
//	@Failure		404				{object}	ErrorResponse
//	@Failure		500				{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/w/{workspace_id}/resource/{id} [get]
func (m *MyResourceAPIs) get(c *gin.Context) {
    var req GetRequest

    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    ctx := c.Request.Context()
    resource, err := m.Params().MyResource.Get(ctx, req.URI.ID)
    if err != nil {
        if err == myresource.ErrNotFound {
            c.JSON(http.StatusNotFound, gin.H{"error": "Resource not found"})
            return
        }

        m.Logger().Error("Failed to get resource", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, GetResponse{
        Resource: m.toEntry(resource),
    })
}

// update updates a resource
//
//	@Summary		Update resource
//	@Description	Update a resource by ID
//	@Tags			Resources
//	@Accept			json
//	@Produce		json
//	@Param			workspace_id	path		string				true	"Workspace ID"
//	@Param			id				path		string				true	"Resource ID"
//	@Param			body			body		UpdateRequestBody	true	"Update request"
//	@Success		200				{object}	UpdateResponse
//	@Failure		400				{object}	ErrorResponse
//	@Failure		404				{object}	ErrorResponse
//	@Failure		500				{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/w/{workspace_id}/resource/{id} [put]
func (m *MyResourceAPIs) update(c *gin.Context) {
    var req UpdateRequest

    // Bind URI parameters
    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Bind JSON body
    if err := c.ShouldBindJSON(&req.Body); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    cfg := &myresource.ResourceConfig{
        Description: req.Body.Description,
        Status:      req.Body.Status,
    }

    ctx := c.Request.Context()
    resource, err := m.Params().MyResource.Update(ctx, req.URI.ID, cfg)
    if err != nil {
        if err == myresource.ErrNotFound {
            c.JSON(http.StatusNotFound, gin.H{"error": "Resource not found"})
            return
        }

        m.Logger().Error("Failed to update resource", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, UpdateResponse{
        Message:  "resource updated successfully",
        Resource: m.toEntry(resource),
    })
}

// delete deletes a resource
//
//	@Summary		Delete resource
//	@Description	Delete a resource by ID
//	@Tags			Resources
//	@Produce		json
//	@Param			workspace_id	path		string	true	"Workspace ID"
//	@Param			id				path		string	true	"Resource ID"
//	@Success		200				{object}	DeleteResponse
//	@Failure		400				{object}	ErrorResponse
//	@Failure		404				{object}	ErrorResponse
//	@Failure		500				{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/w/{workspace_id}/resource/{id} [delete]
func (m *MyResourceAPIs) delete(c *gin.Context) {
    var req DeleteRequest

    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    ctx := c.Request.Context()
    err := m.Params().MyResource.Delete(ctx, req.URI.ID)
    if err != nil {
        if err == myresource.ErrNotFound {
            c.JSON(http.StatusNotFound, gin.H{"error": "Resource not found"})
            return
        }

        m.Logger().Error("Failed to delete resource", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, DeleteResponse{
        Message: "resource deleted successfully",
    })
}

// list lists resources with pagination
//
//	@Summary		List resources
//	@Description	List all resources in workspace with pagination, search, and sorting
//	@Tags			Resources
//	@Produce		json
//	@Param			workspace_id	path		string	true	"Workspace ID"
//	@Param			page			query		int		false	"Page number"						default(1)
//	@Param			page_size		query		int		false	"Page size"							default(10)
//	@Param			keywords		query		string	false	"Search keywords"
//	@Param			search_fields	query		string	false	"Search fields (comma-separated)"
//	@Param			orderby			query		string	false	"Order by fields (comma-separated)"
//	@Param			order			query		int		false	"Sort direction (1=asc, -1=desc)"	default(1)
//	@Param			status			query		string	false	"Status filter"
//	@Success		200				{object}	ListResponse
//	@Failure		400				{object}	ErrorResponse
//	@Failure		500				{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/w/{workspace_id}/resources [get]
func (m *MyResourceAPIs) list(c *gin.Context) {
    var req ListRequest

    // Bind URI parameters
    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Bind query parameters
    if err := c.ShouldBindQuery(&req.Query); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Set defaults
    if req.Query.Page == 0 {
        req.Query.Page = 1
    }
    if req.Query.PageSize == 0 {
        req.Query.PageSize = 10
    }
    if req.Query.Order == 0 {
        req.Query.Order = 1
    }

    // Parse comma-separated fields
    searchFields := parseCommaSeparated(req.Query.SearchFields)
    orderBy := parseCommaSeparated(req.Query.OrderBy)

    // Build filter conditions
    filters := make([]queryhelper.FilterCondition, 0)
    if req.Query.Status != "" {
        filters = append(filters, queryhelper.FilterCondition{
            Field:    "status",
            Operator: "=",
            Value:    req.Query.Status,
        })
    }

    // Build QueryHelper
    qh := queryhelper.NewQueryHelper(
        queryhelper.WithPage(req.Query.Page),
        queryhelper.WithPageSize(req.Query.PageSize),
        queryhelper.WithSearchText(req.Query.Keywords),
        queryhelper.WithSearchFields(searchFields),
        queryhelper.WithOrderBy(orderBy),
        queryhelper.WithSortFactor(req.Query.Order),
        queryhelper.WithFilters(filters),
    )

    // Build query conditions
    listReq := &myresource.ListResourcesRequest{
        WorkspaceID: &req.URI.WorkspaceID,
    }

    // Call business layer
    ctx := c.Request.Context()
    result, err := m.Params().MyResource.List(ctx, listReq, qh)
    if err != nil {
        m.Logger().Error("Failed to list resources", zap.Error(err))
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // Transform to response format
    entries := make([]*ResourceEntry, len(result.Data))
    for i, r := range result.Data {
        entries[i] = m.toEntry(r)
    }

    // Get query info
    qi := result.QueryHelper.Info()

    c.JSON(http.StatusOK, ListResponse{
        Total:      qi.Pagination.Total,
        Page:       qi.Pagination.Page,
        PageSize:   qi.Pagination.PageSize,
        TotalPages: qi.Pagination.TotalPages,
        OrderBy:    qi.Conditions.OrderBy,
        Order:      qi.Conditions.SortFactor,
        Keywords:   qi.Conditions.SearchText,
        Resources:  entries,
    })
}

// toEntry converts business layer structure to API response structure
func (m *MyResourceAPIs) toEntry(r *myresource.Resource) *ResourceEntry {
    return &ResourceEntry{
        ID:          r.ID,
        Name:        r.Name,
        Description: r.Description,
        Status:      r.Status,
        WorkspaceID: r.WorkspaceID,
        CreatedAt:   r.CreatedAt.UTC().Format(time.RFC3339),
        UpdatedAt:   r.UpdatedAt.UTC().Format(time.RFC3339),
    }
}

// parseCommaSeparated parses comma-separated string
func parseCommaSeparated(s string) []string {
    if s == "" {
        return nil
    }
    parts := strings.Split(s, ",")
    result := make([]string, 0, len(parts))
    for _, p := range parts {
        p = strings.TrimSpace(p)
        if p != "" {
            result = append(result, p)
        }
    }
    return result
}
```

## Request Binding Pattern

### Why Separate URI/Body/Query?

Gin's binding methods can overwrite struct fields when called multiple times on the same struct. Separating into sub-structs prevents this:

```go
// ❌ BAD - Fields may be overwritten
type BadRequest struct {
    WorkspaceID string `uri:"workspace_id" json:"workspace_id"`
    Name        string `json:"name"`
}

// ✅ GOOD - Clean separation
type GoodRequestURI struct {
    WorkspaceID string `uri:"workspace_id"`
}
type GoodRequestBody struct {
    Name string `json:"name"`
}
type GoodRequest struct {
    URI  GoodRequestURI
    Body GoodRequestBody
}
```

### Binding Order

Always bind in this order: **URI → Query → Body**

```go
func (m *MyAPIs) handler(c *gin.Context) {
    var req MyRequest

    // 1. Bind URI first
    if err := c.ShouldBindUri(&req.URI); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 2. Bind Query (if applicable)
    if err := c.ShouldBindQuery(&req.Query); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 3. Bind Body (if applicable)
    if err := c.ShouldBindJSON(&req.Body); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Use req.URI.WorkspaceID, req.Query.Page, req.Body.Name, etc.
}
```

## Swagger Annotation Format

All handler annotations must use the tab-indented format. Key rules:

- **Format**: `//\t@Tag\t\tValue` (tabs between `//` and `@`, and between tag and value)
- **First line**: `// functionName description` followed by an empty `//` line
- **`@Param body`**: Reference `*RequestBody` struct, not the outer `*Request`
- **`@Accept json`**: Only on POST/PUT (handlers with a body); omit for GET/DELETE
- **`@Router`**: Full path with `/apis/v1` prefix; path params use `{id}` not `:id`
- **`@Security BearerAuth`**: Include on authenticated endpoints
- **`ErrorResponse`**: Define in `codec.go`; use instead of `object{error=string}`

For complete annotation conventions, see [../../common-modules/modules/swagger.md](../../common-modules/modules/swagger.md).

## Development Checklist

- [ ] Create `pkg/myresource_apis/` directory
- [ ] Define request structures with URI/Body/Query separation `codec.go`
- [ ] Define response structures and `ErrorResponse` `codec.go`
- [ ] Implement module definition (Method 2) `module.go`
- [ ] Implement HTTP handlers with proper binding order `apis.go`
- [ ] Add Swagger annotations (tab-indented format)
- [ ] Register module in `modules.go`
- [ ] Run `swag init --parseDependency --parseDependencyLevel 3` to update docs
