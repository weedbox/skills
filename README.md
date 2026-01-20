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

**Two methods for module development:**

| Method | Description | Use Case |
|--------|-------------|----------|
| Method 1: Manual FX | Manual control over FX dependency injection | Complex FX annotations or fine-grained control needed |
| Method 2: Weedbox Generic | Uses `weedbox.Module[P]` generics (Recommended) | New modules, simple dependencies, less boilerplate |

## Installation

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
└── module-dev/
    ├── SKILL.md                              # Module development skill
    └── references/
        ├── METHOD1_MANUAL_FX.md              # Manual FX module details
        └── METHOD2_WEEDBOX_GENERIC.md        # Weedbox generic module details
```

## Related Projects

- [weedbox/weedbox](https://github.com/weedbox/weedbox) - Weedbox base module framework
- [weedbox/common-modules](https://github.com/weedbox/common-modules) - Common reusable modules

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
