# AI Accelerator Guide

Centralized, governed AI coding standards for modern development teams. This repository provides markdown instruction files that work across AI coding assistants including GitHub Copilot, Claude Code, Cursor, Kiro, and others.

## Why Markdown Instructions?

AI coding assistants excel at generating code that compiles. Getting code to meet organizational standards is a different challenge entirely. Without structured governance, these tools become a source of technical debt rather than a solution to it.

**Common failure modes without governance:**
- Inconsistent code styles across developers, teams, and tools
- Validation of poor technical decisions rather than constructive pushback
- Missing domain-specific context (data engineering vs. backend vs. DevOps)
- Skipped security checks, quality gates, and compliance requirements
- Placeholder code (`TODO`, `FIXME`) that ships to production

## Repository Structure

```
ai-coding-standards/
├── roles/                    # Role-specific instructions
│   ├── data-engineer.md      # SQL, pipelines, data quality, dbt
│   ├── software-engineer.md  # Code quality, architecture, testing
│   ├── devops-engineer.md    # IaC, CI/CD, containers, monitoring
│   ├── qa-engineer.md        # Testing strategy, automation
│   └── security-engineer.md  # AppSec, compliance, incident response
│
├── platforms/                # Cloud platform guidelines
│   ├── aws.md                # IAM, EC2, S3, RDS, Lambda
│   ├── gcp.md                # BigQuery, GKE, Cloud Run
│   └── azure.md              # ARM, AKS, App Service, Key Vault
│
├── languages/                # Language-specific standards
│   ├── python.md             # PEP 8, type hints, testing
│   ├── typescript.md         # Strict mode, React patterns
│   ├── java.md               # Spring Boot, modern Java
│   └── go.md                 # Error handling, concurrency
│
└── tool-configs/             # AI tool configuration templates
    ├── copilot-instructions.md   # GitHub Copilot
    ├── claude.md                 # Claude Code
    ├── cursorrules               # Cursor
    └── README.md                 # Usage guide
```

## Quick Start

### 1. Choose Your Tool Configuration

Copy the appropriate template to your project:

```bash
# GitHub Copilot
mkdir -p .github
cp ai-coding-standards/tool-configs/copilot-instructions.md .github/copilot-instructions.md

# Claude Code
cp ai-coding-standards/tool-configs/claude.md CLAUDE.md

# Cursor
cp ai-coding-standards/tool-configs/cursorrules .cursorrules
```

### 2. Add Role-Specific Instructions

Append role-specific guidance to your instruction file:

```bash
# Example: Add data engineering standards to Claude Code config
cat ai-coding-standards/roles/data-engineer.md >> CLAUDE.md
```

### 3. Add Language/Platform Standards (Optional)

```bash
# Add Python standards
cat ai-coding-standards/languages/python.md >> CLAUDE.md

# Add AWS guidelines
cat ai-coding-standards/platforms/aws.md >> CLAUDE.md
```

## Supported AI Tools

| Tool | Configuration File | Location |
|------|-------------------|----------|
| GitHub Copilot | `copilot-instructions.md` | `.github/copilot-instructions.md` |
| Claude Code | `CLAUDE.md` | Repository root (hierarchical support) |
| Cursor | `.cursorrules` | Repository root |
| Kiro | Spec markdown files | Project specs directory |
| Amazon Q | Context files | Project root |

## Key Principles

All instruction files follow these core principles:

### Development Process
1. **Plan First** - Discuss approach before writing code
2. **Surface Decisions** - Identify all implementation choices
3. **Present Trade-offs** - Show pros/cons of different approaches
4. **Confirm Alignment** - Ensure agreement before implementing
5. **Then Implement** - Write code only after alignment

### Anti-Patterns to Avoid
- Never use `TODO`/`FIXME` in production code
- Never implement partial solutions without acknowledgment
- Never agree with incorrect technical decisions
- Never skip error handling or validation

### Communication Style
- Direct, professional feedback
- Challenge flawed approaches
- Present options objectively
- Admit knowledge gaps honestly

## Composing Instructions

Create comprehensive instruction files by combining modules:

```bash
# Create a full instruction file for a Python data engineer on AWS
cat << 'EOF' > CLAUDE.md
# Project AI Instructions
EOF

cat ai-coding-standards/tool-configs/claude.md >> CLAUDE.md
echo -e "\n---\n" >> CLAUDE.md
cat ai-coding-standards/roles/data-engineer.md >> CLAUDE.md
echo -e "\n---\n" >> CLAUDE.md
cat ai-coding-standards/languages/python.md >> CLAUDE.md
echo -e "\n---\n" >> CLAUDE.md
cat ai-coding-standards/platforms/aws.md >> CLAUDE.md
```

## Benefits

- **Tool-Agnostic** - Same standards work across AI assistants
- **Version Controlled** - Track changes via git
- **Peer Reviewed** - Standards changes go through PR review
- **Role-Specific** - Tailored guidance for each discipline
- **Composable** - Mix and match modules as needed
- **Centrally Published** - Single source of truth for the organization

## Contributing

1. Fork this repository
2. Add or modify instruction files
3. Submit a pull request with clear description
4. Ensure consistency with existing format

---

**Questions or suggestions?** Open an issue in this repository.
