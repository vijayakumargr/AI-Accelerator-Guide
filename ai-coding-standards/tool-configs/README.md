# Tool Configuration Guide

This directory contains instruction file templates for various AI coding assistants. Use these as starting points and customize for your organization's needs.

## Quick Reference

| Tool | File Name | Location |
|------|-----------|----------|
| GitHub Copilot | `copilot-instructions.md` | `.github/copilot-instructions.md` |
| Claude Code | `CLAUDE.md` or `claude.md` | Repository root or any directory |
| Cursor | `.cursorrules` | Repository root |
| Cursor (advanced) | Multiple files | `.cursor/rules/` directory |

## Usage Instructions

### GitHub Copilot

1. Copy `copilot-instructions.md` to `.github/copilot-instructions.md` in your repository
2. Customize the content for your project's needs
3. Copilot will automatically read this file for context

```bash
mkdir -p .github
cp tool-configs/copilot-instructions.md .github/copilot-instructions.md
```

### Claude Code

1. Copy `claude.md` to `CLAUDE.md` in your repository root
2. For directory-specific rules, place additional `CLAUDE.md` files in subdirectories
3. Claude Code supports hierarchical instructions (subdirectory files override parent)

```bash
cp tool-configs/claude.md CLAUDE.md
```

### Cursor

1. Copy `cursorrules` to `.cursorrules` in your repository root
2. For more complex setups, use the `.cursor/rules/` directory

```bash
cp tool-configs/cursorrules .cursorrules
```

## Combining with Role-Specific Instructions

For maximum effectiveness, combine tool configurations with role-specific instructions:

### Example: Data Engineer using Claude Code

```bash
# Copy base Claude config
cp tool-configs/claude.md CLAUDE.md

# Append data engineering specific rules
cat roles/data-engineer.md >> CLAUDE.md
```

### Example: Python Developer using Copilot

```bash
# Copy base Copilot config
cp tool-configs/copilot-instructions.md .github/copilot-instructions.md

# Append language-specific rules
cat languages/python.md >> .github/copilot-instructions.md
```

## Composing Instructions

You can compose instructions from multiple sources:

```bash
# Create a comprehensive instruction file
cat << 'EOF' > CLAUDE.md
# Project Instructions

EOF

# Add role-specific instructions
cat roles/software-engineer.md >> CLAUDE.md

# Add language-specific instructions
echo -e "\n---\n" >> CLAUDE.md
cat languages/typescript.md >> CLAUDE.md

# Add platform-specific instructions
echo -e "\n---\n" >> CLAUDE.md
cat platforms/aws.md >> CLAUDE.md
```

## Best Practices

1. **Start Simple**: Begin with the base configuration and add more as needed
2. **Be Specific**: Add project-specific conventions and patterns
3. **Version Control**: Track changes to instruction files in git
4. **Review Regularly**: Update instructions as your project evolves
5. **Team Alignment**: Ensure all team members use the same instruction files

## Directory Structure in Your Project

```
your-project/
├── .github/
│   └── copilot-instructions.md    # For GitHub Copilot
├── .cursorrules                    # For Cursor
├── CLAUDE.md                       # For Claude Code
├── src/
│   └── CLAUDE.md                   # Directory-specific (optional)
└── tests/
    └── CLAUDE.md                   # Directory-specific (optional)
```

## Customization Tips

### Adding Project-Specific Patterns

```markdown
## Project-Specific Patterns

### API Endpoints
- All endpoints follow REST conventions
- Use kebab-case for URLs
- Version prefix: /api/v1/

### Database
- Use PostgreSQL with TypeORM
- Migrations required for schema changes
- Soft deletes for user data
```

### Adding Team Conventions

```markdown
## Team Conventions

### Code Review
- All PRs require at least one approval
- CI must pass before merge
- Squash commits on merge

### Git
- Branch naming: feature/TICKET-123-description
- Commit messages: conventional commits format
```

## Maintenance

Keep your instruction files updated:

1. **Quarterly Review**: Review and update instructions quarterly
2. **Post-Incident**: Add learnings from production incidents
3. **New Patterns**: Document new patterns as they emerge
4. **Deprecations**: Remove outdated guidance
