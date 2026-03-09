# Skills Engineering

Copilot Skills for engineering with emerging APIs. Hard-won integration guides distilled into reusable skill files that AI coding agents can consume.

## What Are Skills?

Skills are structured guides that teach AI coding agents (GitHub Copilot, Claude, etc.) how to work with specific APIs, frameworks, or patterns. They capture the pitfalls, workarounds, and correct integration sequences that would otherwise take hours of trial-and-error to discover.

## Available Skills

| Skill | Description |
|-------|-------------|
| [implement-agent-id](skills/implement-agent-id/SKILL.md) | Microsoft Entra Agent Identity APIs (Graph beta) — authentication, blueprints, agent identity provisioning, sponsors, permissions, and known pitfalls |

## Usage

### With GitHub Copilot (VS Code)

Copy a skill folder into your project's `.claude/skills/` or `.github/copilot/skills/` directory:

```bash
# Example: add the Agent ID skill to your project
mkdir -p .claude/skills/implement-agent-id
cp skills/implement-agent-id/SKILL.md .claude/skills/implement-agent-id/
```

The agent will automatically load the skill when it encounters a relevant task.

### Manual Reference

Each `SKILL.md` file is a standalone document you can read directly. No tooling required.

## Contributing

Have a hard-won integration guide? Add a new folder under `skills/` with a `SKILL.md` file. The frontmatter should include:

```yaml
---
name: your-skill-name
description: When to use this skill. Include keywords for agent discovery.
---
```

## License

MIT
