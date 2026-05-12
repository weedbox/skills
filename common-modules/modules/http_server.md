# http_server

Gin-based HTTP server with CORS and static file support.

## Installation

```bash
go get github.com/weedbox/common-modules/http_server
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/http_server"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        http_server.Module("http_server"),
    }, nil
}
```

## Configuration

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `{scope}.host` | `0.0.0.0` | Listen address |
| `{scope}.port` | `80` | Listen port |
| `{scope}.mode` | `test` | Controls whether CORS injects `allow_headers` (only added in `test` mode) |
| `{scope}.loglevel` | (empty) | Gin mode selector: `test` → `gin.TestMode`, `release` or `prod` → `gin.ReleaseMode`. `prod` also swaps `gin.Default()` for `gin.New()` (no default middleware) |
| `{scope}.allow_origins` | (empty) | Comma-separated CORS allowed origins. Empty enables `AllowAllOrigins=true` (wildcard) |
| `{scope}.allow_methods` | (empty) | Comma-separated CORS allowed methods |
| `{scope}.allow_headers` | `Authorization,Accept` | Comma-separated CORS allowed headers (only applied when `mode=test`) |

### Running Modes

`mode` and `loglevel` are **independent** configs despite the overlapping naming:

- `mode` only affects whether `allow_headers` is appended to the CORS config (test mode adds them; other values skip).
- `loglevel` selects the Gin runtime mode:

| `loglevel` | Gin behaviour |
|------------|---------------|
| `test` | `gin.TestMode` with `gin.Default()` middleware |
| `release` | `gin.ReleaseMode` with `gin.Default()` middleware |
| `prod` | `gin.ReleaseMode` with `gin.New()` (no default logger/recovery) |
| (empty / other) | Gin's own default; `gin.Default()` middleware |

### TOML Example

```toml
[http_server]
host = "0.0.0.0"
port = 8080
mode = "release"
allow_origins = "https://example.com,https://api.example.com"
allow_methods = "GET,POST,PUT,DELETE"
allow_headers = "Authorization,Accept,Content-Type"
```

### Environment Variables

```bash
export HTTP_SERVER_HOST=0.0.0.0
export HTTP_SERVER_PORT=8080
export HTTP_SERVER_MODE=release
```

## Registering Routes

```go
type Params struct {
    fx.In
    HTTPServer *http_server.HTTPServer
}

type MyModule struct {
    params Params
}

func (m *MyModule) OnStart(ctx context.Context) error {
    router := m.params.HTTPServer.GetRouter()

    // Register API routes
    api := router.Group("/api")
    {
        api.GET("/users", m.listUsers)
        api.GET("/users/:id", m.getUser)
        api.POST("/users", m.createUser)
        api.PUT("/users/:id", m.updateUser)
        api.DELETE("/users/:id", m.deleteUser)
    }

    return nil
}

func (m *MyModule) listUsers(c *gin.Context) {
    c.JSON(200, gin.H{"users": []string{}})
}
```

## Route Groups with Middleware

```go
func (m *MyModule) OnStart(ctx context.Context) error {
    router := m.params.HTTPServer.GetRouter()

    // Public routes
    public := router.Group("/api")
    {
        public.POST("/login", m.login)
        public.POST("/register", m.register)
    }

    // Protected routes with auth middleware
    protected := router.Group("/api")
    protected.Use(m.authMiddleware())
    {
        protected.GET("/profile", m.getProfile)
        protected.PUT("/profile", m.updateProfile)
    }

    return nil
}

func (m *MyModule) authMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }
        // Validate token...
        c.Next()
    }
}
```

## Serving Static Files (SPA)

```go
import "embed"

//go:embed dist/*
var staticFS embed.FS

func (m *MyModule) OnStart(ctx context.Context) error {
    httpServer := m.params.HTTPServer

    // Serve embedded filesystem with SPA fallback
    // All unmatched routes will serve index.html
    httpServer.ServeFS("/", staticFS)

    return nil
}
```

## API Methods

| Method | Description |
|--------|-------------|
| `Module(scope string)` | Create Fx module |
| `GetRouter()` | Get Gin engine for route registration |
| `ServeFS(path, fs)` | Serve static files with SPA support |

## Request Handling Examples

```go
// Path parameters
func (m *MyModule) getUser(c *gin.Context) {
    id := c.Param("id")
    // ...
}

// Query parameters
func (m *MyModule) listUsers(c *gin.Context) {
    page := c.DefaultQuery("page", "1")
    limit := c.DefaultQuery("limit", "10")
    // ...
}

// JSON body
func (m *MyModule) createUser(c *gin.Context) {
    var input CreateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // ...
}

// Response
func (m *MyModule) getUser(c *gin.Context) {
    user := User{ID: "123", Name: "John"}
    c.JSON(200, user)
}
```
