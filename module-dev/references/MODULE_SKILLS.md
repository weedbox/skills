# Module Skills Guide

This document explains how to create skills files for weedbox modules, enabling Claude Code to better understand and assist with specific module development.

## Overview

Module Skills are documentation files placed in the `.skills/` subdirectory within each module directory. They describe:
- Data models and database structures
- Configuration structures
- Manager methods
- Error types
- Usage examples
- Relationships with other modules

## Directory Structure

```
pkg/
├── mymodule/
│   ├── .skills/
│   │   └── mymodule-development.md    # Module development guide
│   ├── module.go
│   ├── manager.go
│   ├── codec.go
│   ├── errors.go
│   └── models/
│       └── mymodel.go
└── mymodule_apis/
    ├── .skills/
    │   └── mymodule-apis-development.md  # API development guide
    ├── module.go
    ├── apis.go
    └── codec.go
```

## Naming Conventions

| Module Type | Skills Filename |
|-------------|-----------------|
| Business Logic Module | `<module>-development.md` |
| API Module | `<module>-apis-development.md` |

Examples:
- `pkg/product/.skills/product-development.md`
- `pkg/product_apis/.skills/product-apis-development.md`

## Business Logic Module Skills Structure

### Standard Sections

```markdown
# <Module Name> Development Skill

## Module Overview
- Module purpose
- Module path
- Main features list

## Data Model
### <Model> Structure
- Database table name
- Field description table (field, type, description)
- GORM tag explanations

### Public Structure
- Public structure definitions from codec.go

## Configuration Structure
### <Config> Structure
- Configuration structure for create/update operations

## Manager Methods
### Create / Get / Update / Delete / List
- Method signature
- Parameter description
- Return value description
- Special behaviors (auto-generated ID, defaults)

### Other Business Methods
- Validation methods
- Related operations

## Error Types
- Error constant definitions
- Usage scenarios

## QueryHelper Settings (if applicable)
- AllowedOrderBy
- AllowedSearch
- AllowedFilters

## Usage Examples
- Create resource example
- Query example
- Special operation examples

## Relationships with Other Modules
- Dependencies
- Related entities (1:N, N:1, etc.)
```

### Example: Business Logic Module

```markdown
# Product Development Skill

## Module Overview

The Product module manages product data and is a core entity in the licensing system.

**Module Path**: `pkg/product/`

**Main Features**:
- Product CRUD operations
- Attribute schema definition and validation
- Pagination, search, sorting, filtering

## Data Model

### Product Structure

**Database Table**: `products`

| Field | Type | Description |
|-------|------|-------------|
| `ID` | `varchar(36)` | UUID v7 primary key |
| `Name` | `varchar(255)` | Product name (unique index) |
| `Status` | `varchar(50)` | Status, default `active` |
| `CreatedAt` | `datetime` | Creation time (indexed) |

## Manager Methods

### Create

\`\`\`go
func (m *ProductManager) Create(ctx context.Context, cfg *ProductConfig) (*Product, error)
\`\`\`
- Auto-generates UUID v7
- Default Status is `active`

## Error Types

\`\`\`go
var (
    ErrNotFound   = errors.New("product not found")
    ErrNameExists = errors.New("product name already exists")
)
\`\`\`

## Usage Examples

### Create a Product

\`\`\`go
product, err := productManager.Create(ctx, &product.ProductConfig{
    Name:   "Enterprise Software",
    Status: "active",
})
\`\`\`
```

## API Module Skills Structure

### Standard Sections

```markdown
# <Module> APIs Development Skill

## Module Overview
- Module purpose
- Base path
- Dependencies

## API Endpoints
- Endpoint table (method, path, description)

## Request/Response Structures
### <Operation Name> (METHOD /path)
- Request structure (Go struct)
- Response structure (JSON example)
- Error response list

## Response Structures
### <Entry> Structure
- Item structure in API responses

## Module Structure
### Params
- Dependency injection structure

### Route Registration
- Route setup in OnStart

## Error Handling
### HTTP Status Code Mapping
- Business error to HTTP status code mapping table

## Validation Rules
- Gin binding tag explanations

## Usage Examples
- cURL examples
```

### Example: API Module

```markdown
# Product APIs Development Skill

## Module Overview

The Product APIs module provides RESTful HTTP APIs for product management.

**Module Path**: `pkg/product_apis/`

**Base Path**: `/api/v1`

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/product` | Create a product |
| `GET` | `/product/:id` | Get a single product |
| `GET` | `/products` | List products (paginated) |

## Request/Response Structures

### Create Product (POST /product)

**Request**:

\`\`\`go
type CreateRequestBody struct {
    Name   string `json:"name" binding:"required,min=1,max=255"`
    Status string `json:"status" binding:"omitempty,oneof=active inactive"`
}
\`\`\`

**Response** (201 Created):

\`\`\`json
{
  "message": "product created successfully",
  "product": {...}
}
\`\`\`

## Error Handling

| Business Error | HTTP Status Code |
|----------------|------------------|
| `product.ErrNotFound` | 404 Not Found |
| `product.ErrNameExists` | 409 Conflict |

## Usage Examples

### cURL Example

\`\`\`bash
curl -X POST http://localhost:8080/api/v1/product \\
  -H "Content-Type: application/json" \\
  -d '{"name": "My Product", "status": "active"}'
\`\`\`
```

## Special Module Skills Content

### Cryptography-Related Modules

For modules involving encryption, signatures, etc., include additional documentation:

- Key structures (MasterKey, PublicKey, PrivateKey)
- Encryption/decryption flow
- Signing and verification flow
- Security considerations

### Process-Oriented Modules

For modules with multi-step processes, include additional documentation:

- Process steps (numbered list)
- Validation and operations for each step
- Failure handling

## Best Practices

### 1. Keep Documentation Synchronized

Update skills files when module code changes:
- Update method list when adding new methods
- Update data model when modifying fields
- Update error list when adding new error types

### 2. Provide Complete Examples

- Examples should be copy-paste ready
- Include necessary imports and error handling
- Explain expected results

### 3. Document Relationships

Clearly describe relationships between modules:
- Which modules it depends on
- Which modules depend on it
- Data relationships (1:1, 1:N, N:N)

### 4. Use Tables

Leverage Markdown tables to organize information:
- Field descriptions
- Method lists
- Error mappings
- API endpoints

## When to Create Skills

Create module skills in these situations:

1. **After new module development** - Serve as development documentation
2. **Complex modules** - Multiple files or special logic involved
3. **Cross-module integration** - Need to explain module collaboration
4. **API modules** - Provide endpoint reference

## Verification Checklist

After creating skills, verify:

- [ ] Module overview clearly explains purpose
- [ ] Data model fields are complete
- [ ] All public methods are documented
- [ ] Error types are fully listed
- [ ] Usage examples are executable
- [ ] Relationships with other modules are explained
- [ ] Markdown formatting is correct
