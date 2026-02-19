# swagger

Swagger/OpenAPI documentation with Scalar API Reference UI.

## Installation

```bash
go get github.com/weedbox/common-modules/swagger
```

## Usage in Fx

```go
import (
    "github.com/weedbox/common-modules/swagger"
    _ "your-project/docs"  // Import generated swagger docs
)

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        http_server.Module("http_server"),
        swagger.Module("swagger"),
        daemon.Module("daemon"),
    }, nil
}
```

## Dependencies

This module requires:
- `http_server` - For endpoint registration
- Generated swagger docs (using [swag](https://github.com/swaggo/swag))

### Generating Swagger Docs

Install swag CLI:

```bash
go install github.com/swaggo/swag/cmd/swag@latest
```

Add annotations to your main.go and handlers, then generate:

**Basic** (all handlers in same project):

```bash
swag init
```

**With external modules** (using external weedbox modules like `user-modules`):

```bash
swag init --parseDependency --parseDependencyLevel 3
```

The `--parseDependency` flag tells swag to scan Go dependencies for swagger annotations. This is **required** when using external weedbox modules that contain annotated handlers.

> **Note**: External library modules (e.g. `user-modules`) do NOT run `swag init`, do NOT have a `docs/` directory, and do NOT depend on swag. They only contain pure Go comment annotations on their handler functions.

This creates a `docs/` directory with swagger specification files.

## Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `{scope}.enabled` | bool | `true` | Enable or disable Swagger |
| `{scope}.base_path` | string | `/swagger` | Base path for all endpoints |
| `{scope}.title` | string | `API Reference` | Page title for Scalar UI |
| `{scope}.docs_path` | string | `/docs` | Path for JSON spec endpoints |
| `{scope}.ui_path` | string | `/ui` | Path for Scalar UI |
| `{scope}.spec_path` | string | `/doc.json` | JSON spec filename |

### TOML Configuration

```toml
[swagger]
enabled = true
base_path = "/swagger"
title = "My API Reference"
docs_path = "/docs"
ui_path = "/ui"
spec_path = "/doc.json"
```

### Environment Variables

```bash
export MYAPP_SWAGGER_ENABLED=true
export MYAPP_SWAGGER_BASE_PATH=/api-docs
export MYAPP_SWAGGER_TITLE="My API"
```

## Endpoints

With default configuration:

| Endpoint | Description |
|----------|-------------|
| `GET /swagger/docs/*` | Swagger JSON spec (gin-swagger) |
| `GET /swagger/docs/doc.json` | OpenAPI JSON specification |
| `GET /swagger/ui/*` | Scalar API Reference UI |

## Disabling in Production

To disable Swagger in production environments:

**config.toml:**
```toml
[swagger]
enabled = false
```

**Environment Variable:**
```bash
export MYAPP_SWAGGER_ENABLED=false
```

## Complete Example

### main.go

```go
package main

import (
    "os"

    "github.com/spf13/cobra"
    "github.com/weedbox/common-modules/configs"
    "github.com/weedbox/common-modules/daemon"
    "github.com/weedbox/common-modules/http_server"
    "github.com/weedbox/common-modules/logger"
    "github.com/weedbox/common-modules/swagger"
    "go.uber.org/fx"

    _ "myapp/docs"  // Import generated swagger docs
)

// @title           My API
// @version         1.0
// @description     API documentation for My App
// @host            localhost:8080
// @BasePath        /api/v1

//	@securityDefinitions.apikey	BearerAuth
//	@in							header
//	@name						Authorization
//	@description				Enter your bearer token as: Bearer <token>

var config *configs.Config

func main() {
    rootCmd := &cobra.Command{
        Use: "myapp",
        RunE: func(cmd *cobra.Command, args []string) error {
            modules, _ := initModules()
            app := fx.New(modules...)
            app.Run()
            return nil
        },
    }

    config = configs.NewConfig("MYAPP")

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}

func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        logger.Module(),

        http_server.Module("http_server"),
        swagger.Module("swagger"),

        // Your API modules here

        daemon.Module("daemon"),
    }, nil
}
```

### Handler with Swagger Annotations

**GET handler** (no `@Accept json` since there is no request body):

```go
// get retrieves a resource
//
//	@Summary		Get a resource
//	@Description	Retrieve a resource by ID
//	@Tags			Resource
//	@Produce		json
//	@Param			id	path		string	true	"Resource ID"
//	@Success		200	{object}	GetResponse
//	@Failure		400	{object}	ErrorResponse
//	@Failure		404	{object}	ErrorResponse
//	@Failure		500	{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/resource/{id} [get]
func (m *MyResourceAPIs) get(c *gin.Context) {
    // Implementation
}
```

**POST handler** (includes `@Accept json` for request body):

```go
// create creates a resource
//
//	@Summary		Create a resource
//	@Description	Create a new resource
//	@Tags			Resource
//	@Accept			json
//	@Produce		json
//	@Param			body	body		CreateRequestBody	true	"Creation payload"
//	@Success		201		{object}	CreateResponse
//	@Failure		400		{object}	ErrorResponse
//	@Failure		409		{object}	ErrorResponse
//	@Failure		500		{object}	ErrorResponse
//	@Security		BearerAuth
//	@Router			/apis/v1/resource [post]
func (m *MyResourceAPIs) create(c *gin.Context) {
    // Implementation
}
```

## Annotation Conventions

| Item | Rule |
|------|------|
| Format | Tab-indented: `//\t@Tag\t\tValue` (use tabs, not spaces) |
| First line | `// functionName description` (swag uses this for parsing) |
| Blank line | Empty comment `//` between first line and annotations |
| `@Router` | Full path with `/apis/v1` prefix; path params use `{id}` not `:id` |
| `@Param body` | Reference `*RequestBody` struct (not the outer `*Request` wrapper) |
| `@Accept json` | Only on POST/PUT handlers that have a request body; omit for GET/DELETE |
| `@Produce json` | Always include on all handlers |
| `@Security BearerAuth` | Include on authenticated endpoints; omit on public endpoints (e.g. login) |
| `ErrorResponse` | Each API package defines its own `ErrorResponse` struct in `codec.go` |

### ErrorResponse Convention

Each API package should define an `ErrorResponse` struct for swagger to reference:

```go
// ErrorResponse error response
type ErrorResponse struct {
	Error string `json:"error" example:"error message"`
}
```

Use `ErrorResponse` (not `object{error=string}`) in all `@Failure` annotations.

## External Module Integration

This section explains how swagger works when API handlers live in an **external Go module** (library) rather than in the application project itself.

### For Module Authors (developing a reusable library like `user-modules`)

- Add swag annotations as pure Go comments on handler functions
- Define `ErrorResponse` struct in each API package's `codec.go`
- Do **NOT** run `swag init` — the library has no `main.go` to parse
- Do **NOT** create a `docs/` directory
- Do **NOT** add any swag Go dependency — annotations are just comments

### For Project Developers (building an application that uses external modules)

1. Add general API info and security definitions in `main.go`:

```go
// @title           My API
// @version         1.0
// @description     API documentation
//
//	@securityDefinitions.apikey	BearerAuth
//	@in							header
//	@name						Authorization
//	@description				Enter your bearer token as: Bearer <token>
```

2. Generate docs with dependency parsing:

```bash
swag init --parseDependency --parseDependencyLevel 3
```

3. Import generated docs and load the swagger module:

```go
import _ "your-project/docs"

// In module setup:
swagger.Module("swagger"),
```

The `--parseDependency` flag tells swag to scan all Go module dependencies for annotations, producing a unified Swagger spec that includes both local and external module endpoints.

## Scalar UI Features

The module uses [Scalar](https://github.com/scalar/scalar) for API documentation UI, which provides:

- Modern, clean interface
- Try-it-out functionality (hidden by default)
- Search and navigation
- Dark/light mode support
- Mobile responsive design

## Custom Paths Example

```toml
[swagger]
enabled = true
base_path = "/api-docs"
title = "Backend API"
docs_path = "/spec"
ui_path = "/reference"
```

This configuration creates:
- `GET /api-docs/spec/doc.json` - OpenAPI spec
- `GET /api-docs/reference/*` - Scalar UI
