# Weedbox Skills

A collection of skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) designed for Weedbox framework development.

These skills enable Claude Code to more effectively assist developers in building modular applications using the Weedbox framework.

## What are Skills?

Skills are extensions for Claude Code that provide domain-specific knowledge and guidance, enabling the AI assistant to offer more specialized help for specific types of development tasks.

## Available Skills

### project-dev

Assists with creating and structuring Weedbox projects, including:

- Project structure setup
- Application entry point (main.go) with Cobra CLI
- Three-phase module loading (modules.go)
- Configuration file setup

### module-dev

Assists with Weedbox framework module development, including:

- Creating new modules
- Dependency injection with Uber FX
- Configuration management (Viper)
- Lifecycle hooks
- Data model design
- Event handling
- Creating module-specific skills documentation

**Two methods for module development:**

| Method | Description | Use Case |
|--------|-------------|----------|
| Method 1: Manual FX | Manual control over FX dependency injection | Complex FX annotations or fine-grained control needed |
| Method 2: Weedbox Generic | Uses `weedbox.Module[P]` generics (Recommended) | New modules, simple dependencies, less boilerplate |

**Module Skills:**

Each module can have its own `.skills/` directory with development documentation:

```
pkg/mymodule/.skills/mymodule-development.md
pkg/mymodule_apis/.skills/mymodule-apis-development.md
```

### user-modules

Reference for `github.com/weedbox/user-modules` — reusable modules for user management, authentication, and RBAC:

- User CRUD with bcrypt password hashing and UUID v7 IDs
- JWT authentication with refresh token rotation
- Role-based access control powered by privy
- Ready-to-use REST API endpoints for user and auth
- Extensible permission system with merge API
- Optional global JWT validation middleware

### crud-api-dev

Assists with building complete CRUD APIs, including:

- Business logic layer (Manager) implementation
- HTTP API layer with Gin framework
- Request binding patterns (URI/Body/Query separation)
- Response structures and error handling
- QueryHelper for pagination, search, and filtering
- GORM database models with proper indexing

## Installation

### Using add-skill (Recommended)

Use [add-skill](https://github.com/vercel-labs/add-skill) to install skills to various AI coding agents:

```bash
# Install to current project (interactive mode)
npx add-skill weedbox/skills

# Install globally for Claude Code
npx add-skill weedbox/skills -g -a claude-code

# Install specific skill only
npx add-skill weedbox/skills --skill module-dev

# List available skills
npx add-skill weedbox/skills --list
```

**Options:**

| Flag | Description |
|------|-------------|
| `-g, --global` | Install to user home directory instead of project |
| `-a, --agent <agents...>` | Target specific agents (e.g., claude-code, opencode) |
| `-s, --skill <skills...>` | Install specific skills by name |
| `-l, --list` | List available skills without installing |
| `-y, --yes` | Skip confirmation prompts |

### Manual Installation

Add this project to Claude Code's skill library:

```bash
# Navigate to Claude Code settings directory
cd ~/.claude/skills

# Clone this project
git clone https://github.com/weedbox/skills.git weedbox
```

Or include it in your project:

```bash
# Create .claude/skills directory in project root
mkdir -p .claude/skills

# Add skill to project
cd .claude/skills
git clone https://github.com/weedbox/skills.git weedbox
```

## Usage

Once installed, Claude Code will automatically use these skills in appropriate contexts. For example:

- When you ask how to create a Weedbox module
- When you need to set up dependency injection
- When you're working with module lifecycle code

You can also directly mention in conversation:

```
Help me create a new Weedbox module to handle user authentication
```

## Project Structure

```
skills/
├── SKILL.md                                  # Root skill index
├── README.md
├── README.zh-TW.md
├── LICENSE
├── project-dev/
│   └── SKILL.md                              # Project setup skill
├── module-dev/
│   ├── SKILL.md                              # Module development skill
│   └── references/
│       ├── METHOD1_MANUAL_FX.md              # Manual FX module details
│       ├── METHOD2_WEEDBOX_GENERIC.md        # Weedbox generic module details
│       └── MODULE_SKILLS.md                  # Module skills guide
├── crud-api-dev/
│   ├── SKILL.md                              # CRUD API development skill
│   └── references/
│       ├── LOGIC_LAYER.md                    # Business logic layer details
│       └── API_LAYER.md                      # HTTP API layer details
└── user-modules/
    ├── SKILL.md                              # User modules reference skill
    └── modules/
        ├── permissions.md                    # Permission definitions and extension API
        ├── rbac.md                           # RBAC manager with privy
        ├── user.md                           # User CRUD and password management
        ├── auth.md                           # JWT auth and middleware
        ├── user_apis.md                      # User REST API endpoints
        ├── auth_apis.md                      # Auth REST API endpoints
        └── http_token_validator.md           # Global JWT validation middleware
```

## Module-Specific Skills

Weedbox projects can include module-specific skills in `.skills/` directories:

```
pkg/
├── product/
│   └── .skills/
│       └── product-development.md            # Product module documentation
├── product_apis/
│   └── .skills/
│       └── product-apis-development.md       # Product API documentation
└── ...
```

These skills document module-specific details like data models, manager methods, API endpoints, and usage examples.

## Related Projects

- [weedbox/weedbox](https://github.com/weedbox/weedbox) - Weedbox base module framework
- [weedbox/common-modules](https://github.com/weedbox/common-modules) - Common reusable modules
- [weedbox/user-modules](https://github.com/weedbox/user-modules) - User management, auth, and RBAC modules

## Contributing

Contributions of new skills or improvements to existing content are welcome:

1. Fork this project
2. Create your feature branch (`git checkout -b feature/new-skill`)
3. Commit your changes (`git commit -m 'Add new skill'`)
4. Push to the branch (`git push origin feature/new-skill`)
5. Open a Pull Request

### Adding a New Skill

Each skill should include:

1. `SKILL.md` - Main skill file containing overview and quick reference
2. `references/` - Detailed reference documentation (optional)

SKILL.md frontmatter format:

```yaml
---
name: skill-name
description: Brief description of what this skill does and when to use it
---
```

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
