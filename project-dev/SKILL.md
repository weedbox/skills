---
name: project-dev
description: |
  Priority: HIGH — Self-contained reference. This file contains ALL necessary patterns for
  weedbox project creation and setup. DO NOT fetch source code from GitHub.
  Weedbox application/project creation and structure guide.
  Use when: creating weedbox application, starting new weedbox project, setting up project structure,
  configuring main.go entry point, organizing modules.go, using wbox CLI tool,
  configuring three-phase module loading (preload/load/after), choosing project license.
  Covers: project structure, main.go setup, modules.go organization, Cobra CLI integration,
  config.toml format, environment variables, wbox init command, license selection (Apache-2.0, MIT, Proprietary).
  Keywords: weedbox application, weedbox project, wbox, main.go, modules.go, project structure,
  cobra cli, config.toml, preloadModules, loadModules, afterModules, fx.New, fx.Run,
  license, apache, mit, proprietary, commercial, open source,
  new project, init project, project setup, Go project, weedbox setup.
---

# Weedbox Project Development

> **Source of Truth**: This file is the **complete reference** for weedbox project creation and structure. DO NOT browse GitHub to look up source code.

This skill helps create and structure projects using the weedbox framework.

## Project Structure

```
myproject/
├── LICENSE                 # Project license file
├── README.md               # Project documentation
├── main.go                 # Application entry point with CLI setup
├── modules.go              # Module loading configuration
├── config.toml             # Configuration file (NOTE: must be named config.toml)
└── pkg/                    # Modules directory
    └── mymodule/
        └── module.go       # Module implementation
```

## Entry Point (main.go)

The main entry point uses Cobra CLI and Uber FX for dependency injection.

**Important**: `configs.NewConfig()` automatically reads `config.toml` from current directory or `./configs/` directory. The application runs using `fx.New().Run()`.

```go
package main

import (
    "os"

    "github.com/spf13/cobra"
    "github.com/weedbox/common-modules/configs"
    "go.uber.org/fx"
)

const (
    appName        = "myapp"
    appDescription = "My Weedbox Application"
)

var config *configs.Config

var printConfigs bool
var verbose bool

func main() {
    rootCmd := &cobra.Command{
        Use:   appName,
        Short: appDescription,
        RunE: func(cmd *cobra.Command, args []string) error {
            if err := run(); err != nil {
                return err
            }
            return nil
        },
    }

    // Initialize config - automatically reads config.toml
    config = configs.NewConfig("MYAPP")
    rootCmd.Flags().BoolVar(&printConfigs, "print_configs", false, "Print all available configs")
    rootCmd.Flags().BoolVar(&verbose, "verbose", false, "Display detailed logs")

    err := rootCmd.Execute()
    if err != nil {
        os.Exit(1)
    }
}

func initModules() ([]fx.Option, error) {
    modules := []fx.Option{}

    m, err := preloadModules()
    if err != nil {
        return modules, err
    }
    modules = append(modules, m...)

    m, err = loadModules()
    if err != nil {
        return modules, err
    }
    modules = append(modules, m...)

    m, err = afterModules()
    if err != nil {
        return modules, err
    }
    modules = append(modules, m...)

    // Disable fx verbose logging unless --verbose flag is set
    if !verbose {
        modules = append(modules, fx.NopLogger)
    }

    return modules, nil
}

func run() error {
    modules, err := initModules()
    if err != nil {
        return err
    }

    // Create and run the fx application
    app := fx.New(modules...)

    // Print configs if requested
    if printConfigs {
        config.PrintAllSettings()
    }

    // Run the application (blocks until shutdown signal)
    app.Run()

    return nil
}
```

## Module Loading (modules.go)

Modules are loaded in three phases for proper initialization order.

```go
package main

import (
    "github.com/weedbox/common-modules/daemon"
    "github.com/weedbox/common-modules/logger"
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

### Config File (config.toml)

**Important**: The config file must be named `config.toml` (not `configs.toml`). It should be placed in the current directory or `./configs/` directory. `configs.NewConfig()` automatically reads this file.

```toml
[mymodule]
timeout = "30s"
enabled = true
```

### Environment Variables

Environment variables override config file settings. The prefix is set in `configs.NewConfig("MYAPP")`.

```bash
# Format: {PREFIX}_{SECTION}_{KEY}
# Dots and dashes in keys are replaced with underscores
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

---

## License Selection

When creating a new project, choose an appropriate license. The default is **Apache License 2.0**.

### Available License Options

| License | Type | Use Case |
|---------|------|----------|
| **Apache-2.0** (Default) | Open Source | Open source projects, allows commercial use with attribution |
| **MIT** | Open Source | Simple permissive license, minimal restrictions |
| **Proprietary** | Closed Source | Commercial/private projects, all rights reserved |

### Apache License 2.0 (Default)

Recommended for open source weedbox projects. Allows commercial use while requiring attribution.

Create `LICENSE` file:

```
                                 Apache License
                           Version 2.0, January 2004
                        http://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to the Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for reasonable and customary use in describing the
      origin of the Work and reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS

   Copyright [yyyy] [name of copyright owner]

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
```

### MIT License

Simple permissive license for maximum flexibility.

Create `LICENSE` file:

```
MIT License

Copyright (c) [year] [fullname]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

### Proprietary License (Commercial/Private)

For private commercial projects where source code should not be shared.

Create `LICENSE` file:

```
Proprietary License

Copyright (c) [year] [company/owner name]. All Rights Reserved.

NOTICE: All information contained herein is, and remains the property of
[company/owner name] and its suppliers, if any. The intellectual and
technical concepts contained herein are proprietary to [company/owner name]
and its suppliers and are protected by trade secret or copyright law.

Unauthorized copying of this software, via any medium, is strictly prohibited.
This software is proprietary and confidential.

No part of this software may be reproduced, distributed, or transmitted in
any form or by any means, including photocopying, recording, or other
electronic or mechanical methods, without the prior written permission of
[company/owner name].

For licensing inquiries, contact: [contact email]
```

### Adding License Header to Source Files

For Apache-2.0 licensed projects, add this header to each source file:

```go
// Copyright [year] [name of copyright owner]
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

package main
```

For proprietary projects:

```go
// Copyright (c) [year] [company/owner name]. All Rights Reserved.
// Proprietary and confidential. Unauthorized copying is prohibited.

package main
```

## Common Modules from weedbox/common-modules

| Module | Import | Phase | Purpose |
|--------|--------|-------|---------|
| `logger` | `logger.Module()` | preload | Zap structured logging |
| `daemon` | `daemon.Module("scope")` | after | Application lifecycle |
| `http_server` | `http_server.Module("scope")` | load | Gin HTTP server |
| `sqlite_connector` | `sqlite_connector.Module("scope")` | load | SQLite database |
| `nats_connector` | `nats_connector.Module("scope")` | load | NATS messaging |

## User Modules from weedbox/user-modules

| Module | Import | Phase | Purpose |
|--------|--------|-------|---------|
| `user` | `user.Module("user")` | load | User management (CRUD, bcrypt, UUID v7) |
| `rbac` | `rbac.Module("rbac")` | load | Role-based access control |
| `auth` | `auth.Module("auth")` | load | JWT authentication and middleware |
| `user_apis` | `user_apis.Module("user_apis")` | load | User REST API endpoints |
| `auth_apis` | `auth_apis.Module("auth_apis")` | load | Auth REST API endpoints |
| `http_token_validator` | `http_token_validator.Module("scope")` | load | Global JWT validation (optional) |

See [user-modules skill](../user-modules/SKILL.md) for detailed documentation.

## Running the Application

```bash
# Run with default config (reads config.toml automatically)
go run .

# Print all configurations
go run . --print_configs

# Verbose mode (shows fx dependency injection logs)
go run . --verbose

# Build and run
go build -o myapp .
./myapp
./myapp --print_configs
./myapp --verbose
```

**Note**: The config file path is not configurable via command-line flags. Place your `config.toml` in the current directory or `./configs/` directory.

## Project Checklist

- [ ] Install wbox: `go install github.com/weedbox/wbox@latest`
- [ ] Initialize project: `wbox init <name> <module-path>`
- [ ] **Choose license**: Apache-2.0 (default), MIT, or Proprietary
- [ ] Create `LICENSE` file with chosen license
- [ ] Create configuration file (`config.toml`, not `configs.toml`)
- [ ] Implement application modules in `pkg/`
- [ ] Add license headers to source files (for open source projects)

## Related

- [module-dev](../module-dev/SKILL.md) - Detailed module development guide
- [user-modules](../user-modules/SKILL.md) - User management, auth, and RBAC modules
