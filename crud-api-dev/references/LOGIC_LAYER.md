# Business Logic Layer (Manager)

This document covers the implementation of the business logic layer for CRUD operations.

## Directory Structure

```
pkg/myresource/
├── module.go           # Module definition and FX setup
├── manager.go          # Core CRUD business logic
├── codec.go            # Internal data structures and config
├── errors.go           # Error definitions
└── models/
    └── myresource.go   # GORM database model
```

## 1. Database Model (`models/myresource.go`)

```go
package models

import (
    "time"
)

// BaseModel shared base model
type BaseModel struct {
    CreatedAt time.Time `gorm:"index" json:"created_at"`
    UpdatedAt time.Time `gorm:"index" json:"updated_at"`
}

// MyResource resource model
type MyResource struct {
    BaseModel
    ID          string `gorm:"type:varchar(36);primary_key" json:"id"`
    Name        string `gorm:"type:varchar(255);not null;uniqueIndex:idx_myresource_name" json:"name"`
    Description string `gorm:"type:text" json:"description"`
    Status      string `gorm:"type:varchar(50);default:'active';index:idx_myresource_status" json:"status"`
    WorkspaceID string `gorm:"type:varchar(36);not null;index:idx_myresource_workspace" json:"workspace_id"`
}

// TableName specifies the table name
func (MyResource) TableName() string {
    return "my_resources"
}
```

### Model Design Guidelines

- Use UUID v7 as primary key (time-sortable)
- Create indexes for frequently queried columns
- Use `uniqueIndex` for uniqueness constraints
- **Always prefix index names with table name** to avoid conflicts (e.g., `idx_myresource_name` instead of `idx_name`)
- Composite unique indexes use the same name, e.g., `uniqueIndex:idx_myresource_tenant_name`

### GORM Index Examples

```go
type Example struct {
    // Basic index (auto-named, safe)
    Email string `gorm:"index"`

    // Named index with table prefix
    Status string `gorm:"index:idx_example_status"`

    // Unique index with table prefix
    Code string `gorm:"uniqueIndex:idx_example_code"`

    // Composite index
    TenantID string `gorm:"index:idx_example_tenant_user"`
    UserID   string `gorm:"index:idx_example_tenant_user"`

    // Index with sort order
    Score int `gorm:"index:idx_example_score,sort:desc"`
}
```

## 2. Error Definitions (`errors.go`)

```go
package myresource

import "errors"

var (
    ErrNotFound        = errors.New("resource not found")
    ErrNameExists      = errors.New("resource name already exists")
    ErrInvalidInput    = errors.New("invalid input")
    ErrOperationFailed = errors.New("operation failed")
)
```

## 3. Internal Structures (`codec.go`)

```go
package myresource

import (
    "time"

    "github.com/weedbox/queryhelper"
)

// ResourceConfig configuration for create/update
type ResourceConfig struct {
    Name        string
    Description string
    Status      string
}

// ListResourcesRequest filter conditions for list query
type ListResourcesRequest struct {
    Name        *string
    Status      *string
    WorkspaceID *string
}

// ListResourcesResp list query response
type ListResourcesResp struct {
    Data        []*Resource
    QueryHelper *queryhelper.QueryHelper
}

// Resource public resource structure
type Resource struct {
    ID          string
    Name        string
    Description string
    Status      string
    WorkspaceID string
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// QueryHelper settings
var DefaultQuerySettings_ListResources = &queryhelper.QuerySettings{
    AllowedOrderBy: []string{"created_at", "updated_at", "name"},
    AllowedSearch:  []string{"name", "description"},
    AllowedFilters: map[string][]string{
        "status": {"=", "!=", "IN"},
        "name":   {"=", "LIKE"},
    },
}
```

## 4. Module Definition (`module.go`)

Using Weedbox Module Method 2 (recommended):

```go
package myresource

import (
    "context"

    "github.com/spf13/viper"
    "github.com/weedbox/common-modules/database"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"
    "go.uber.org/zap"

    "your-project/pkg/myresource/models"
    "your-project/pkg/system"
)

const ModuleName = "myresource"

var ServiceEnabled = true

type Params struct {
    weedbox.Params
    Database database.DatabaseConnector
    System   *system.System
}

type MyResourceManager struct {
    weedbox.Module[*Params]
}

func Module(scope string) fx.Option {
    m := new(MyResourceManager)

    return fx.Module(
        scope,
        fx.Supply(fx.Annotated{Name: scope, Target: m}),
        fx.Invoke(func(p Params) {
            weedbox.InitModule(scope, &p, m)
        }),
    )
}

func (m *MyResourceManager) InitDefaultConfigs() {
    viper.SetDefault(m.GetConfigPath("max_page_size"), 100)
}

func (m *MyResourceManager) OnStart(ctx context.Context) error {
    m.Logger().Info("Starting " + ModuleName)

    // Check if service is disabled
    if m.Params().System.IsServiceDisabled(m.Scope()) {
        m.Logger().Info("Service:" + m.Scope() + " is disabled.")
        ServiceEnabled = false
        return nil
    }

    // Database migration
    db := m.Params().Database.GetDB()
    if err := db.AutoMigrate(&models.MyResource{}); err != nil {
        m.Logger().Error("Failed to migrate tables", zap.Error(err))
        return err
    }

    m.Logger().Info("Started " + ModuleName)
    return nil
}

func (m *MyResourceManager) OnStop(ctx context.Context) error {
    m.Logger().Info("Stopped " + ModuleName)
    return nil
}
```

## 5. CRUD Business Logic (`manager.go`)

```go
package myresource

import (
    "context"
    "errors"
    "fmt"

    "github.com/gofrs/uuid/v5"
    "github.com/weedbox/queryhelper"
    "go.uber.org/zap"
    "gorm.io/gorm"

    "your-project/pkg/myresource/models"
)

// Create creates a resource
func (m *MyResourceManager) Create(ctx context.Context, workspaceID string, cfg *ResourceConfig) (*Resource, error) {
    // Generate UUID v7
    id, err := uuid.NewV7()
    if err != nil {
        return nil, fmt.Errorf("failed to generate ID: %w", err)
    }

    resource := models.MyResource{
        ID:          id.String(),
        Name:        cfg.Name,
        Description: cfg.Description,
        Status:      cfg.Status,
        WorkspaceID: workspaceID,
    }

    // Set defaults
    if resource.Status == "" {
        resource.Status = "active"
    }

    db := m.Params().Database.GetDB().WithContext(ctx)
    if err := db.Create(&resource).Error; err != nil {
        // Handle uniqueness constraint error
        if errors.Is(err, gorm.ErrDuplicatedKey) {
            return nil, ErrNameExists
        }
        return nil, fmt.Errorf("failed to create resource: %w", err)
    }

    m.Logger().Info("Created resource",
        zap.String("id", resource.ID),
        zap.String("name", resource.Name),
    )

    return m.toResource(&resource), nil
}

// Get retrieves a single resource
func (m *MyResourceManager) Get(ctx context.Context, resourceID string) (*Resource, error) {
    var resource models.MyResource

    db := m.Params().Database.GetDB().WithContext(ctx)
    if err := db.Where("id = ?", resourceID).First(&resource).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrNotFound
        }
        return nil, err
    }

    return m.toResource(&resource), nil
}

// Update updates a resource
func (m *MyResourceManager) Update(ctx context.Context, resourceID string, cfg *ResourceConfig) (*Resource, error) {
    var resource models.MyResource

    db := m.Params().Database.GetDB().WithContext(ctx)
    if err := db.Where("id = ?", resourceID).First(&resource).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrNotFound
        }
        return nil, err
    }

    // Update fields (only non-empty values)
    if cfg.Description != "" {
        resource.Description = cfg.Description
    }
    if cfg.Status != "" {
        resource.Status = cfg.Status
    }

    if err := db.Save(&resource).Error; err != nil {
        return nil, fmt.Errorf("failed to update resource: %w", err)
    }

    m.Logger().Info("Updated resource",
        zap.String("id", resource.ID),
    )

    return m.toResource(&resource), nil
}

// Delete deletes a resource
func (m *MyResourceManager) Delete(ctx context.Context, resourceID string) error {
    db := m.Params().Database.GetDB().WithContext(ctx)

    result := db.Where("id = ?", resourceID).Delete(&models.MyResource{})
    if result.Error != nil {
        return fmt.Errorf("failed to delete resource: %w", result.Error)
    }

    if result.RowsAffected == 0 {
        return ErrNotFound
    }

    m.Logger().Info("Deleted resource",
        zap.String("id", resourceID),
    )

    return nil
}

// List queries with pagination, search, sorting, and filtering
func (m *MyResourceManager) List(ctx context.Context, req *ListResourcesRequest, qh *queryhelper.QueryHelper) (*ListResourcesResp, error) {
    db := m.Params().Database.GetDB().WithContext(ctx)
    query := db.Model(&models.MyResource{})

    // Apply filter conditions
    if req != nil {
        if req.Name != nil {
            query = query.Where("name = ?", *req.Name)
        }
        if req.Status != nil {
            query = query.Where("status = ?", *req.Status)
        }
        if req.WorkspaceID != nil {
            query = query.Where("workspace_id = ?", *req.WorkspaceID)
        }
    }

    // Apply QueryHelper (pagination, search, sorting)
    query, err := qh.Apply(DefaultQuerySettings_ListResources, query)
    if err != nil {
        m.Logger().Error("Failed to apply queryhelper", zap.Error(err))
        return nil, err
    }

    // Execute query
    var resources []models.MyResource
    if err := query.Find(&resources).Error; err != nil {
        m.Logger().Error("Failed to list resources", zap.Error(err))
        return nil, err
    }

    // Transform results
    results := make([]*Resource, len(resources))
    for i, r := range resources {
        results[i] = m.toResource(&r)
    }

    return &ListResourcesResp{
        Data:        results,
        QueryHelper: qh,
    }, nil
}

// toResource converts model to public structure
func (m *MyResourceManager) toResource(model *models.MyResource) *Resource {
    return &Resource{
        ID:          model.ID,
        Name:        model.Name,
        Description: model.Description,
        Status:      model.Status,
        WorkspaceID: model.WorkspaceID,
        CreatedAt:   model.CreatedAt,
        UpdatedAt:   model.UpdatedAt,
    }
}
```

## Development Checklist

- [ ] Create `pkg/myresource/` directory
- [ ] Define database model `models/myresource.go`
- [ ] Define errors `errors.go`
- [ ] Define internal structures `codec.go`
- [ ] Implement module definition `module.go`
- [ ] Implement CRUD methods `manager.go`
- [ ] Configure QuerySettings
- [ ] Register module in `modules.go`
