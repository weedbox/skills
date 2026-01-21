# configs

Configuration management using Viper with TOML file and environment variable support.

## Installation

```bash
go get github.com/weedbox/common-modules/configs
```

## Usage

```go
import "github.com/weedbox/common-modules/configs"

// Create config with environment variable prefix
config := configs.NewConfig("MYAPP")

// Set defaults programmatically
config.SetConfigs(map[string]interface{}{
    "http_server.host": "0.0.0.0",
    "http_server.port": 8080,
})

// Bind CLI flags
config.BindFlag(rootCmd, "configs", "c", "configs.toml", "Configuration file path")
config.SetConfigFile(cmd)

// Debug: print all settings
config.PrintAllSettings()
```

## Configuration Priority

Settings are resolved in this order (highest priority first):

1. **Environment variables**: `MYAPP_HTTP_SERVER_PORT=8080`
2. **TOML config file**
3. **Programmatic defaults**

## Environment Variable Mapping

Variables use the pattern `PREFIX_KEY_NAME`, where dots convert to underscores.

| Config Key | Environment Variable |
|------------|---------------------|
| `http_server.port` | `MYAPP_HTTP_SERVER_PORT` |
| `database.host` | `MYAPP_DATABASE_HOST` |
| `nats.max_reconnects` | `MYAPP_NATS_MAX_RECONNECTS` |

## TOML Format

```toml
[http_server]
host = "0.0.0.0"
port = 8080

[database]
host = "localhost"
port = 5432
dbname = "myapp"
```

## API Methods

| Method | Description |
|--------|-------------|
| `NewConfig(prefix string)` | Create new config instance with env prefix |
| `SetConfigs(map)` | Set default values (won't override existing) |
| `GetAllSettings()` | Return all settings as nested map |
| `PrintAllSettings()` | Print all settings to stdout |
| `BindFlag(cmd, name, short, default, usage)` | Bind CLI flag |
| `SetConfigFile(cmd)` | Load config file from CLI flag |

## Integration with Fx

```go
package main

import (
    "github.com/spf13/cobra"
    "github.com/weedbox/common-modules/configs"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"
)

var config = configs.NewConfig("MYAPP")

var rootCmd = &cobra.Command{
    Use: "myapp",
    RunE: func(cmd *cobra.Command, args []string) error {
        config.SetConfigFile(cmd)
        modules, _ := initModules()
        return weedbox.Run(modules...)
    },
}

func init() {
    config.BindFlag(rootCmd, "configs", "c", "configs.toml", "Config file")
}

func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        // other modules...
    }, nil
}
```
