# swagger

Swagger/OpenAPI documentation with Scalar API Reference UI.

## Table of Contents

- [Installation](#installation)
- [Usage in Fx](#usage-in-fx)
- [Dependencies](#dependencies)
- [Configuration](#configuration)
- [Endpoints](#endpoints)
- [Disabling in Production](#disabling-in-production)
- [Complete Example](#complete-example)
- [Annotation Conventions](#annotation-conventions)
- [External Module Integration](#external-module-integration)
- [swag CLI Version Compatibility (Important)](#swag-cli-version-compatibility-important)
- [Scalar UI Features](#scalar-ui-features)
- [Custom Paths Example](#custom-paths-example)

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

> **Before running `swag init` for the first time, read [swag CLI Version Compatibility](#swag-cli-version-compatibility-important) below.** The CLI version you pick determines whether external module annotations are picked up *and* whether the generated `docs.go` compiles. The `swag init` invocations below are the headline cases; the compatibility section explains how to reconcile CLI version with the runtime library pinned by `common-modules/swagger`.

Add annotations to your main.go and handlers, then generate. The recommended invocation uses `go run` to pin the CLI version inside the repo (no global install needed):

**Basic** (all handlers in same project):

```bash
go run github.com/swaggo/swag/cmd/swag@v1.16.6 init
```

**With external modules** (annotated handlers live in an external weedbox module like `user-modules`):

```bash
go run github.com/swaggo/swag/cmd/swag@v1.16.6 init \
  --dir "./,$(go env GOMODCACHE)/github.com/weedbox/user-modules@v1.2.3/user_apis" \
  --parseDependency --parseDependencyLevel 1
```

`--dir` scans the external module's source like first-party code so its `@Router` endpoints enter the spec, while `--parseDependencyLevel 1` resolves the model types those handlers reference. **Do not** use `--parseDependencyLevel 3` to pick up external operations — walking *every* dependency's operation comments aborts on malformed `@`-comments in unrelated third-party deps. The `--parseDependencyLevel` flag is only recognised by CLI `v1.16.x`+ — see [swag CLI Version Compatibility](#swag-cli-version-compatibility-important) for the rationale, the `--dir` details, and the post-generation fix-up needed against the pinned runtime.

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

### For Module Authors (Developing a Reusable Library Like `user-modules`)

- Add swag annotations as pure Go comments on handler functions
- Define `ErrorResponse` struct in each API package's `codec.go`
- Do **NOT** run `swag init` — the library has no `main.go` to parse
- Do **NOT** create a `docs/` directory
- Do **NOT** add any swag Go dependency — annotations are just comments

### For Project Developers (Building an Application That Uses External Modules)

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

2. Generate docs, scanning the external module's source for its operation annotations and resolving model types via dependency parsing:

```bash
go run github.com/swaggo/swag/cmd/swag@v1.16.6 init \
  --dir "./,$(go env GOMODCACHE)/github.com/weedbox/user-modules@v1.2.3/user_apis" \
  --parseDependency --parseDependencyLevel 1
```

3. Import generated docs and load the swagger module:

```go
import _ "your-project/docs"

// In module setup:
swagger.Module("swagger"),
```

This produces a unified Swagger spec that includes both local and external module endpoints. The older `--parseDependencyLevel 3` approach (walk *all* dependency operations) is fragile — see [Dependency Operation Parsing](#dependency-operation-parsing-the---parsedependencylevel-3-trap-and-the---dir-fix) for why `--dir` is preferred.

## swag CLI Version Compatibility (Important)

The `common-modules/swagger` package pins the **runtime library** `github.com/swaggo/swag` at `v1.8.12`. This is independent from the **swag CLI** version you invoke to generate `docs/`. Mixing the two requires care — read this section before choosing a CLI version.

### The Trade-Off in One Table

| swag CLI version | Can it scan external module annotations (e.g. `user-modules`)? | Generated `docs.go` compatible with runtime `v1.8.12`? |
|------------------|-----------------------------------------------------------------|--------------------------------------------------------|
| `v1.8.12` | **No** — `--parseDependency` exists but its parser misses handlers in nested module dependencies; common result is `paths: 0`. The flag is `--parseDepth` (AST depth, default 100), and `--parseDependencyLevel` is **not recognised**. | Yes |
| `v1.16.x` (e.g. `v1.16.6`) | **Yes** — dependency parser was rewritten; correctly resolves handlers in `user-modules/*_apis`. The depth flag was renamed to `--parseDependencyLevel` (module-hop level, not AST depth). | **No** — emits two extra fields (`LeftDelim` / `RightDelim`) in the `swag.Spec` struct literal that do not exist in `v1.8.12`. Build will fail with `unknown field LeftDelim/RightDelim`. |

**Recommended choice for weedbox projects that use external API modules**: use CLI `v1.16.6` and strip the two unsupported fields from the generated `docs.go`. This is the only combination that produces a complete spec **and** compiles against the pinned runtime.

### Flag Name Change Between Versions

| Purpose | `v1.8.12` flag | `v1.16.x` flag |
|---------|----------------|----------------|
| Enable dependency scanning | `--parseDependency` | `--parseDependency` (unchanged) |
| Control dependency depth | `--parseDepth <N>` (AST depth, default 100) | `--parseDependencyLevel <N>` (module-hop level; use `1`, not `3` — see [Dependency Operation Parsing](#dependency-operation-parsing-the---parsedependencylevel-3-trap-and-the---dir-fix)) |

Passing `--parseDependencyLevel` to `v1.8.12` produces `flag provided but not defined: -parseDependencyLevel`.

### Recommended Makefile Target

Drop this into a project Makefile so every developer gets a reproducible build regardless of whether `swag` is installed on their `$PATH`:

```makefile
SWAG_VERSION := v1.16.6

# Directories swag scans for operation annotations (@Router/@Summary/@Param),
# comma separated; the general-info file (main.go) must live in the FIRST one.
# To document an external weedbox API module pulled in as a dependency, append
# its source dir under the module cache, e.g.:
#   SWAG_DIRS := ./,$(shell go env GOMODCACHE)/github.com/weedbox/user-modules@v1.2.3/user_apis
# See "Dependency Operation Parsing" below for why this beats --parseDependencyLevel 3.
SWAG_DIRS := ./

# Generate Swagger docs.
# - --parseDependencyLevel 1 parses ONLY the *models* inside dependencies, so
#   handler annotations that reference external types still resolve. We avoid
#   level 2/3 (which also walk dependency *operations*): swag would then choke
#   on malformed @-directive comments in unrelated transitive deps (e.g.
#   antlr4-go) and abort the whole run — see "Dependency Operation Parsing".
# - The runtime lib pinned by common-modules/swagger (v1.8.12) does not know
#   about LeftDelim/RightDelim, so we strip those two lines from docs.go.
docs:
	rm -rf docs
	go run github.com/swaggo/swag/cmd/swag@$(SWAG_VERSION) init \
		--dir $(SWAG_DIRS) \
		--parseDependency --parseDependencyLevel 1
	@sed -i.bak -e '/LeftDelim:/d' -e '/RightDelim:/d' docs/docs.go && rm docs/docs.go.bak
```

> ⚠️ `rm -rf docs` runs **before** generation, so a failed `swag init` leaves you with no `docs/` directory at all. If generation fails, restore the previous spec with `git restore docs/` before retrying.

`go run github.com/swaggo/swag/cmd/swag@<version>` avoids a global `go install` and pins the CLI version inside the repo — useful in CI and on machines where `$GOPATH/bin` is not on `$PATH`.

> **Why level 1, not level 3?** Earlier versions of this guide recommended `--parseDependencyLevel 3`. That works only when the entire dependency tree is free of stray swagger-style comments. In data-heavy projects (Arrow / Iceberg / CEL pull in `antlr4-go`, etc.) level 3 aborts generation. The robust default is **level 1 + targeted `--dir`** — see the next section.

### Dependency Operation Parsing: The `--parseDependencyLevel 3` Trap and the `--dir` Fix

`--parseDependencyLevel` (v1.16.x) controls how deep swag reads into dependency packages:

| Level | What swag parses inside dependencies |
|-------|--------------------------------------|
| `0` (default, no `--parseDependency`) | nothing |
| `1` | **models only** — struct types referenced by your annotations |
| `2` | **operations only** — `@Router` / `@Summary` / `@Param` comments |
| `3` | **all** — models + operations |

Levels `2` and `3` make swag read the *operation* comments of **every transitive dependency**. swag treats any `@param` / `@Success` it encounters as a swagger directive and fails the whole run if it cannot parse it or resolve a referenced type:

```
ParseComment error in .../antlr4-go/.../diagnostic_error_listener.go for comment:
  '// @param ReportedAlts The set of conflicting ...': missing required param comment parameters ...
```

This is common in data-heavy projects: `antlr4-go` (pulled in transitively via CEL / Arrow / timewave, often as a **direct** dependency) carries Javadoc-style `@param` comments on ordinary functions, and any dependency whose annotated handler references a type swag can't resolve triggers `cannot find type definition`. **One bad comment anywhere in the dependency tree aborts generation** — you get no `docs/` at all.

**These flags do NOT rescue level 3:**

| Attempt | Why it fails |
|---------|--------------|
| `--exclude <dep path>` | `--exclude` only applies to the `--dir` search roots, never to packages loaded via `--parseDependency`. Verified ineffective. |
| `--parseDepth 1` | Bounds AST recursion depth, not the dependency set. The offending dep is frequently **direct** (depth 1) via a transitive `require`, so even depth 1 still parses it. |
| `--tags '!x'` | Tag filtering runs *after* comments are parsed; the parse error fires first. |

**The fix — scan only the modules you actually want, with `--dir`:**

Keep `--parseDependencyLevel 1` (models, so external types still resolve) and add the *specific* external weedbox API module's source directory to `--dir`. swag then scans it like first-party code, without walking every other dependency's operations:

```bash
go run github.com/swaggo/swag/cmd/swag@v1.16.6 init \
  --dir "./,$(go env GOMODCACHE)/github.com/weedbox/user-modules@v1.2.3/user_apis" \
  --parseDependency --parseDependencyLevel 1
```

- The general-info file (`main.go` with `@title` etc.) **must** be in the first `--dir`.
- The external module's handlers reference request/response types. When the module is a real `require` of your project, `--parseDependencyLevel 1` resolves them automatically. If a referenced type lives in a sibling package that swag still reports as `cannot find type definition`, add that package's directory to `--dir` too (e.g. `.../user-modules@v1.2.3/user_apis,.../user-modules@v1.2.3/user`).
- This is strictly more precise than level 3: you opt in exactly the modules you want documented and never touch unrelated third-party comments.

**Verified**: a project whose own handlers live in `./`, plus an external `repository_manager_apis` module added via `--dir` (with its `repository_manager` type package), produced a single spec containing **both** endpoint sets (`/apis/v1/logs/*` and `/apis/v1/repos/*`) with zero dependency-walk failures.

### Legacy: Using the `v1.8.12` CLI

Only viable when **all** annotated handlers live in the application itself (no external weedbox API modules like `user_apis`, `auth_apis`, `role_apis`). In this self-contained case the `v1.8.12` CLI matches the pinned runtime exactly, so no post-generation patching is needed.

**Self-contained project** (no external `*_apis` modules):

```bash
go run github.com/swaggo/swag/cmd/swag@v1.8.12 init
```

**With dependency scanning attempted** — note the flag name is `--parseDepth`, not `--parseDependencyLevel`:

```bash
go run github.com/swaggo/swag/cmd/swag@v1.8.12 init \
    --parseDependency --parseInternal --parseDepth 5
```

> ⚠️ **Known limitation**: even with `--parseDependency --parseDepth 5`, `v1.8.12` typically returns `paths: 0` when handlers live in nested module dependencies (e.g. `weedbox/user-modules/user_apis`). The dependency parser was rewritten in `v1.16`. If your project uses any external API modules, the `v1.8.12` CLI is **not** a viable path — switch to the `v1.16.x` flow above.

**Legacy Makefile target** (no patching needed because CLI and runtime versions match):

```makefile
SWAG_VERSION := v1.8.12

# Only use this flow if ALL annotated handlers are in this project.
# For projects that depend on weedbox/user-modules or any other external
# annotated API module, use the v1.16.6 flow instead.
docs:
	rm -rf docs
	go run github.com/swaggo/swag/cmd/swag@$(SWAG_VERSION) init
```

### Choosing Between the Two Flows

| Your project... | Recommended flow |
|-----------------|------------------|
| Uses `user_apis` / `auth_apis` / `role_apis` / any external annotated module | **v1.16.6** + strip `LeftDelim`/`RightDelim` (default) |
| Has all handlers inside its own `main` module, no external annotations | **v1.8.12** plain `swag init` (no patching) |
| Unsure | Default to **v1.16.6** — it works for both cases once the two-line strip is in place |

### Symptoms Cheat Sheet

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `flag provided but not defined: -parseDependencyLevel` | Running `v1.8.12` CLI with the new flag name. | Either upgrade the CLI to `v1.16.x` or use `--parseDepth` instead. |
| `unknown field LeftDelim in struct literal of type "github.com/swaggo/swag".Spec` | `v1.16.x` CLI generated `docs.go`, but project compiles against runtime `v1.8.12` (pinned by `common-modules`). | Strip the `LeftDelim:` / `RightDelim:` lines from `docs/docs.go` (the Makefile target above does this automatically). |
| `paths: 0` in generated `swagger.json` despite annotations existing in external modules. | Using `v1.8.12` CLI — its dependency parser does not see handlers in nested module dependencies. | Switch to `v1.16.x` CLI. |
| `ParseComment error ... @param ...` in a **third-party** dependency (e.g. `antlr4-go`). | `--parseDependencyLevel 2/3` parses the operation comments of every transitive dep; one malformed `@`-comment aborts the run. `--exclude`/`--parseDepth`/`--tags` do not help. | Drop to `--parseDependencyLevel 1` and pull in the specific external API module via `--dir` instead. See [Dependency Operation Parsing](#dependency-operation-parsing-the---parsedependencylevel-3-trap-and-the---dir-fix). |
| `cannot find type definition: X` while swag parses a dependency. | Level 2/3 reached a dependency handler whose referenced type isn't resolvable. | Same fix: level 1 + `--dir` the module (and its type-source package if the type is reported missing). |
| Empty or missing `docs/` after a failed `swag init`. | The `rm -rf docs` in the Makefile target ran *before* generation failed. | `git restore docs/`, fix the swag invocation, then regenerate. |
| `swag: command not found` in CI | `swag` CLI not installed on `$PATH`. | Use `go run github.com/swaggo/swag/cmd/swag@v1.16.6 init ...` — no global install needed. |

### Why Not Just Upgrade the Runtime Lib?

`common-modules/swagger` controls the runtime version, not the application. Upgrading would require a coordinated change in `common-modules` itself. The strip-two-fields workaround is the pragmatic path until `common-modules` bumps the pin.

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
