---
name: project-dev
description: Help with creating and structuring weedbox framework projects. Use when starting a new weedbox project, setting up project structure, or configuring the application entry point.
---

# Weedbox Project Development

This skill helps create and structure projects using the weedbox framework.

## Project Structure

```
myproject/
├── main.go                 # Application entry point with CLI setup
├── modules.go              # Module loading configuration
└── pkg/                    # Modules directory
    └── mymodule/
        └── module.go       # Module implementation
```

## Entry Point (main.go)

The main entry point uses Cobra CLI and Uber FX for dependency injection.

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
    Use:   "myapp",
    Short: "My Weedbox Application",
    RunE: func(cmd *cobra.Command, args []string) error {
        config.SetConfigFile(cmd)

        modules, err := initModules()
        if err != nil {
            return err
        }

        if printConfigs {
            weedbox.PrintConfigs()
        }

        return weedbox.Run(modules...)
    },
}

var printConfigs bool
var verbose bool

func init() {
    rootCmd.PersistentFlags().BoolVar(&printConfigs, "print_configs", false, "Print configurations")
    rootCmd.PersistentFlags().BoolVar(&verbose, "verbose", false, "Verbose output")
    config.BindFlag(rootCmd, "configs", "c", "configs.toml", "Configuration file path")
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        panic(err)
    }
}

func initModules() ([]fx.Option, error) {
    modules := []fx.Option{}

    // Phase 1: Preload
    preload, err := preloadModules()
    if err != nil {
        return nil, err
    }
    modules = append(modules, preload...)

    // Phase 2: Load
    load, err := loadModules()
    if err != nil {
        return nil, err
    }
    modules = append(modules, load...)

    // Phase 3: After
    after, err := afterModules()
    if err != nil {
        return nil, err
    }
    modules = append(modules, after...)

    return modules, nil
}
```

## Module Loading (modules.go)

Modules are loaded in three phases for proper initialization order.

```go
package main

import (
    "github.com/weedbox/common-modules/daemon"
    "github.com/weedbox/common-modules/logger"
    "github.com/weedbox/weedbox"
    "go.uber.org/fx"

    "myproject/pkg/mymodule"
)

// preloadModules - Phase 1: Configuration and logging
func preloadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        fx.Supply(config),
        logger.Module(),
    }
    return modules, nil
}

// loadModules - Phase 2: Application modules
func loadModules() ([]fx.Option, error) {
    modules := []fx.Option{
        mymodule.Module("mymodule"),
    }
    return modules, nil
}

// afterModules - Phase 3: Daemon and lifecycle
func afterModules() ([]fx.Option, error) {
    modules := []fx.Option{
        daemon.Module("daemon"),
    }
    return modules, nil
}
```

## Three-Phase Module Loading

| Phase | Function | Purpose | Typical Modules |
|-------|----------|---------|-----------------|
| 1 | `preloadModules()` | Configuration and logging setup | `configs`, `logger` |
| 2 | `loadModules()` | Application business modules | Custom modules |
| 3 | `afterModules()` | Lifecycle and daemon management | `daemon` |

## Configuration

### Config File (configs.toml)

```toml
[mymodule]
timeout = "30s"
enabled = true

[daemon]
graceful_shutdown_timeout = "10s"
```

### Environment Variables

```bash
# Prefix is set in configs.NewConfig("MYAPP")
export MYAPP_MYMODULE_TIMEOUT=60s
export MYAPP_MYMODULE_ENABLED=true
```

## Creating a New Project

Use [wbox](https://github.com/weedbox/wbox) CLI tool to create a new project.

### Install wbox

```bash
go install github.com/weedbox/wbox@latest
```

### Initialize Project

```bash
mkdir myproject
cd myproject
wbox init myproject github.com/myuser/myproject
```

This generates the complete project structure with `main.go`, `modules.go`, and `pkg/` directory.

### Add Modules Manually

Create modules in `pkg/` directory by hand. See [module-dev](../module-dev/SKILL.md) for detailed guidance.

## Common Modules from weedbox/common-modules

| Module | Import | Phase | Purpose |
|--------|--------|-------|---------|
| `logger` | `logger.Module()` | preload | Zap structured logging |
| `daemon` | `daemon.Module("scope")` | after | Application lifecycle |
| `http_server` | `http_server.Module("scope")` | load | Gin HTTP server |
| `sqlite_connector` | `sqlite_connector.Module("scope")` | load | SQLite database |
| `nats_connector` | `nats_connector.Module("scope")` | load | NATS messaging |

## Running the Application

```bash
# Run with default config
go run .

# Run with custom config file
go run . -c myconfig.toml

# Print all configurations
go run . --print_configs

# Verbose mode
go run . --verbose
```

## Project Checklist

- [ ] Install wbox: `go install github.com/weedbox/wbox@latest`
- [ ] Initialize project: `wbox init <name> <module-path>`
- [ ] Create configuration file (configs.toml)
- [ ] Implement application modules in `pkg/`

## Related

- [module-dev](../module-dev/SKILL.md) - Detailed module development guide
