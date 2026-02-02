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
// This automatically reads config.toml from current directory or ./configs/
config := configs.NewConfig("MYAPP")

// Set defaults programmatically (won't override existing values)
config.SetConfigs(map[string]interface{}{
    "http_server.host": "0.0.0.0",
    "http_server.port": 8080,
})

// Debug: print all settings
config.PrintAllSettings()
```

## Auto-Loading Config File

`NewConfig()` automatically searches for config files:
1. `./config.toml` (current directory)
2. `./configs/config.toml` (configs subdirectory)

**Note**: The file must be named `config.toml`, not `configs.toml`.

## Configuration Priority

Settings are resolved in this order (highest priority first):

1. **Environment variables**: `MYAPP_HTTP_SERVER_PORT=8080`
2. **TOML config file** (`config.toml`)
3. **Programmatic defaults** via `SetConfigs()`

## Environment Variable Mapping

Variables use the pattern `PREFIX_KEY_NAME`, where dots and dashes convert to underscores.

| Config Key | Environment Variable |
|------------|---------------------|
| `http_server.port` | `MYAPP_HTTP_SERVER_PORT` |
| `database.host` | `MYAPP_DATABASE_HOST` |
| `nats.max_reconnects` | `MYAPP_NATS_MAX_RECONNECTS` |

## TOML Format

Create `config.toml` in your project root:

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
| `NewConfig(prefix string)` | Create new config instance with env prefix; auto-loads config.toml |
| `SetConfigs(map)` | Set default values (won't override existing) |
| `GetAllSettings()` | Return all settings as nested map |
| `PrintAllSettings()` | Print all settings to stdout |
| `PrintSettings(scope, map)` | Print settings for a specific scope |

## Integration with Fx

```go
package main

import (
    "os"

    "github.com/spf13/cobra"
    "github.com/weedbox/common-modules/configs"
    "go.uber.org/fx"
)

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

    // Initialize config - automatically reads config.toml
    config = configs.NewConfig("MYAPP")

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}

func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        // other modules...
    }, nil
}
```

## Reading Config Values

Use `github.com/spf13/viper` to read config values in your modules:

```go
import "github.com/spf13/viper"

// Read values
port := viper.GetInt("http_server.port")
host := viper.GetString("http_server.host")
timeout := viper.GetDuration("mymodule.timeout")
enabled := viper.GetBool("mymodule.enabled")

// With module scope helper
func (m *MyModule) GetConfigPath(key string) string {
    return fmt.Sprintf("%s.%s", m.scope, key)
}

timeout := viper.GetDuration(m.GetConfigPath("timeout"))
```
