---
description: "Use when setting up, configuring, or troubleshooting pre-commit hooks for code quality enforcement."
---

# Pre-commit Hooks

Set up and manage pre-commit hooks for code quality enforcement.

**Note**: Core Rules §7 mandates that pre-commit hooks must be installed before committing. This skill provides setup procedures.

## Standard Configuration

Create `.pre-commit-config.yaml` at workspace root:

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-format
        name: ruff-format
        entry: "uv run ruff format bases components projects"
        language: system
        always_run: true
        types: [python]

  - repo: local
    hooks:
      - id: ruff-linter
        name: ruff-linter
        entry: "uv run ruff check --fix bases components projects"
        language: system
        always_run: true
        types: [python]

  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: "uv run pyright"
        pass_filenames: false
        language: system
        always_run: true
        types: [python]

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: "uv run pytest"
        pass_filenames: false
        language: system
        always_run: true
        types: [python]
```

## Procedure: Initial Setup

### Step 1: Ensure pre-commit is installed

Pre-commit is included in the `dev` dependency group:
```bash
uv sync
```

### Step 2: Install git hooks

```bash
uv run pre-commit install
```

This installs hooks that run automatically before each `git commit`.

### Step 3: Verify setup

```bash
# Run hooks manually on all files
uv run pre-commit run --all-files
```

## Hook Descriptions

| Hook | Purpose | Behavior |
|------|---------|----------|
| **ruff-format** | Format Python code | Auto-fixes formatting issues |
| **ruff-linter** | Lint and fix code | Auto-fixes linting issues |
| **pyright** | Static type checking | Validates type annotations and catches type errors |
| **pytest** | Run test suite | Ensures tests pass before commit |

## Handling Hook Failures

### When hooks auto-fix files

The commit will fail but files are fixed. Stage changes and retry:

```bash
git add -u
git commit -m "your commit message"
```

### When type checking fails

Fix the type errors reported by pyright:

```bash
# Run pyright to see detailed errors
uv run pyright

# Fix type issues, then commit
git add -u
git commit -m "your commit message"
```

Common fixes:
- Add missing type hints to function signatures
- Use generic TypeVars for functions that preserve types
- Convert Pydantic dataclasses to BaseModel for validation schemas
- Use proper type narrowing with isinstance checks

### When tests fail

Fix the failing tests, then commit again:

```bash
# See which tests failed
uv run pytest -v

# Fix issues, then commit
git add -u
git commit -m "your commit message"
```

## Optional: Commit Message Validation

Add conventional commit validation:

```yaml
# Add to .pre-commit-config.yaml
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.0.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

Then install commit-msg hook:
```bash
uv run pre-commit install --hook-type commit-msg
```

## Useful Commands

```bash
# Run specific hook
uv run pre-commit run ruff-format --all-files
uv run pre-commit run pyright --all-files

# Run individual tools directly
uv run pyright bases/tp_data_pipeline/confluence/core.py

# Update hook versions
uv run pre-commit autoupdate

# Uninstall hooks
uv run pre-commit uninstall
```

