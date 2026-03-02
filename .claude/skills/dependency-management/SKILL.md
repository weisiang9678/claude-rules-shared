---
description: "Use when adding, removing, updating, or troubleshooting Python dependencies with uv."
---

# Dependency Management

Manage Python dependencies with uv following workspace conventions.

## Context

This skill implements the dependency management procedures for Core Rules §4. See Core Rules for the CRITICAL subset rule and workspace vs project configuration requirements.

## Quick Reference

| Action | Command |
|--------|---------|
| Add runtime dependency | `uv add <package>` |
| Add to group | `uv add <package> --group dev` |
| Remove dependency | `uv remove <package>` |
| Update all | `uv lock --upgrade` |
| Update specific | `uv lock --upgrade-package <package>` |
| Sync (local dev) | `uv sync` |
| Sync (CI/CD) | `uv sync --frozen` |
| Get package version from lock | `uv tree --package <package> --depth 0 --frozen` |
| Get/set package version | `uv version --package <package> [version]` |

## Procedure: Adding Dependencies

### Runtime Dependencies

1. **Add to workspace root:**
   ```bash
   uv add <package_name>
   ```

2. **If needed by specific project, also add to project's pyproject.toml:**
   ```toml
   [project]
   dependencies = [
       "new-package>=X.Y.Z",  # Must match workspace root version
   ]
   ```

### Development Dependencies

Add to appropriate group in workspace root only:

```bash
# Linters, formatters, type checkers
uv add <package_name> --group dev

# Test frameworks and helpers
uv add <package_name> --group test

# CI/CD and deployment tools
uv add <package_name> --group release
```

### Decision Tree

```
Is it needed at runtime in production?
├── YES → uv add <package> (runtime dependency)
└── NO → Continue...

Is it for running tests?
├── YES → uv add <package> --group test
└── NO → Continue...

Is it for deploying/releasing?
├── YES → uv add <package> --group release
└── NO → uv add <package> --group dev
```

## Procedure: Removing Dependencies

```bash
# Remove from workspace root
uv remove <package_name>

# Remove from specific project
cd projects/<project> && uv remove <package_name>

# Remove from group
uv remove <package_name> --group dev
```

## Procedure: Updating Dependencies

### Update All

```bash
uv lock --upgrade
uv sync
```

### Update Specific Package

```bash
uv lock --upgrade-package <package_name>
uv sync
```

### Post-Update Checklist

1. Check resolved versions in `uv.lock`
2. Update `pyproject.toml` constraints to match (recommended)
3. Run tests
4. Commit both `pyproject.toml` and `uv.lock` together

## Critical Rules

### Subset Rule (Visual Reference)

```
Workspace root: [A, B, C, D]
Project 1:      [A, B]      ✅ Valid subset
Project 2:      [B, C, D]   ✅ Valid subset
Project 3:      [A, E]      ❌ E not in workspace root!
```

### Version Constraints

Always use `>=` with tested version:
```toml
# Good
"google-cloud-bigquery>=3.30.0"

# Bad
"google-cloud-bigquery"        # No version
"google-cloud-bigquery>3.0"    # Untested minimum
```

## Standard Dependency Groups

| Group | Purpose | Examples |
|-------|---------|----------|
| (none) | Runtime | Flask, pydantic, google-cloud-* |
| dev | Development tools | ruff, pyright, pre-commit |
| test | Testing | pytest, pytest-cov |
| release | CI/CD deployment | pulumi, semantic-release |

## Troubleshooting

See [troubleshooting.md](references/troubleshooting.md) for common issues.
