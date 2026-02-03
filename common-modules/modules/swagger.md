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

```bash
swag init
```

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

```go
// GetUser godoc
// @Summary      Get user by ID
// @Description  Returns a single user
// @Tags         Users
// @Accept       json
// @Produce      json
// @Param        id   path      int  true  "User ID"
// @Success      200  {object}  User
// @Failure      404  {object}  ErrorResponse
// @Router       /users/{id} [get]
func (h *Handler) GetUser(c *gin.Context) {
    // Implementation
}
```

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
