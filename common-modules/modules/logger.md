# logger

Zap-based structured logging with Uber Fx integration.

## Installation

```bash
go get github.com/weedbox/common-modules/logger
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/logger"

fx.New(
    logger.Module(),
    // other modules...
)
```

## Environment Variables

| Variable | Values | Default | Purpose |
|----------|--------|---------|---------|
| `DEBUG_MODE` | `debug`, `true` | disabled | Enable debug logging |
| `DEBUG_LEVEL` | `debug`/`info`/`warn`/`error`/`dpanic`/`panic`/`fatal` | `debug` | Set log level |

## Injecting Logger

```go
import "go.uber.org/zap"

type Params struct {
    fx.In
    Logger *zap.Logger
}

func NewMyModule(p Params) *MyModule {
    p.Logger.Info("Module initialized",
        zap.String("name", "mymodule"),
    )
    return &MyModule{logger: p.Logger}
}
```

## Structured Logging Examples

```go
// Info with fields
m.logger.Info("User created",
    zap.String("user_id", userID),
    zap.String("email", email),
)

// Warning
m.logger.Warn("Rate limit approaching",
    zap.Int("current", 95),
    zap.Int("limit", 100),
)

// Error with error object
m.logger.Error("Failed to save user",
    zap.Error(err),
    zap.String("user_id", userID),
)

// Debug (only visible when DEBUG_MODE=true)
m.logger.Debug("Processing request",
    zap.Any("payload", payload),
)
```

## Output Format

Logs appear timestamped with level indicators:

```
2024-01-15T10:30:45.123Z INFO  mymodule/service.go:42  User created  {"user_id": "abc123", "email": "user@example.com"}
2024-01-15T10:30:46.456Z ERROR mymodule/service.go:58  Failed to save user  {"error": "connection refused", "user_id": "abc123"}
```

## API Methods

| Method | Description |
|--------|-------------|
| `Module()` | Returns Fx module option |
| `SetupLogger()` | Manual logger initialization |
| `GetLogger()` | Get singleton logger instance |

## Best Practices

1. **Use structured fields** instead of string formatting
2. **Include context** like IDs, timestamps, counts
3. **Use appropriate levels**: Debug for development, Info for normal operations, Warn for recoverable issues, Error for failures
4. **Avoid logging sensitive data** like passwords, tokens, PII
