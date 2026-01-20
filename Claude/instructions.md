# How to Create a Global CLAUDE.md File

You can create a global `CLAUDE.md` file at `~/.claude/CLAUDE.md`. This applies to all projects.

## Setup

```bash
# Create the directory if it doesn't exist
mkdir -p ~/.claude

# Create/edit the global CLAUDE.md file
touch ~/.claude/CLAUDE.md
```

## How It Works

| Location | Scope |
|----------|-------|
| `~/.claude/CLAUDE.md` | Global - applies to all projects |
| `<project>/CLAUDE.md` | Project-specific - overrides/extends global |

**Hierarchy:** Claude Code reads both files. Project-level instructions take precedence when there's a conflict.

## Best Practice

Put your general conventions and preferences in the global file, then use project-level `CLAUDE.md` files only for project-specific details:

- **Global file (`~/.claude/CLAUDE.md`)**: General coding style, language preferences, common patterns
- **Project file (`<project>/CLAUDE.md`)**: Specific tech stack, unique patterns, project-specific rules