---
description: "Use when creating new Polylith components, bases, or projects in the workspace."
---

# Polylith New Brick

Create new Polylith bricks (components, bases) and projects following workspace conventions.

## Prerequisites

Before creating any brick:
1. Confirm the **brick type**: component, base, or project
2. Confirm the **brick name** (e.g., `logging`, `data_processor`)

The namespace is defined in `workspace.toml` and used automatically by the CLI.

**Note**: Core Rules §1 defines the brick creation policy. This skill provides step-by-step procedures.

## Procedure: Creating Components or Bases

### Step 1: Create Brick with Polylith CLI

```bash
# Create a component
uv run poly create component --name {brick_name}

# Create a base
uv run poly create base --name {brick_name}
```

The CLI automatically:
- Creates directory at `components/{namespace}/{brick_name}/` or `bases/{namespace}/{brick_name}/`
- Adds `__init__.py` at brick level (NOT at namespace level - namespace packages don't need it)
- Creates `core.py` for implementation
- Registers brick in workspace root `pyproject.toml` under `[tool.polylith.bricks]`
- Creates test directory `test/components/{namespace}/{brick_name}/test_core.py` (when `[tool.polylith.test] enabled = true`)

### Step 2: Add Dependencies (If Needed)

If the brick needs new dependencies not in workspace root:

1. **Add to workspace root first:**
   ```bash
   uv add {package_name}
   ```

2. **Sync to install:**
   ```bash
   uv sync
   ```

### Step 3: Implement the Brick

Edit `{brick_type}s/{namespace}/{brick_name}/core.py` following Python code standards.

## Procedure: Creating Projects

### Step 1: Create Project with Polylith CLI

```bash
uv run poly create project --name {project_name}
```

### Step 2: Configure Project pyproject.toml

Edit `projects/{project_name}/pyproject.toml` following the template in [pyproject-templates.md](references/pyproject-templates.md).

Key requirements:
- `[build-system]` at the top
- `[tool.polylith.bricks]` uses relative paths (`../../bases/...`, `../../components/...`)
- Follow project pyproject.toml restrictions from Core Rules §4

### Step 3: Add Project-Specific Dependencies

Add only dependencies already in workspace root (subset rule — see Core Rules §4).

### Step 4: Sync and Verify

```bash
uv sync
uv run poly info
```

## Directory Structure Reference

See [directory-structure.md](references/directory-structure.md) for complete examples.

## pyproject.toml Templates

See [pyproject-templates.md](references/pyproject-templates.md) for workspace and project templates.

## Code Sharing Principles

**Component-first Thinking**
- Prioritize creating new functionality as components
- Design components to be independent and reusable
- Ask: "Will another project need this?"

**Avoid Duplication**
- Actively seek to reuse existing components
- Don't copy-paste logic between projects
- Extract common patterns into shared components

**Clear Boundaries**
- Components should have well-defined responsibilities
- Minimize implicit dependencies between components
- Use explicit interfaces (function signatures, type hints)

## Common Mistakes to Avoid

- Using absolute paths in project's `[tool.polylith.bricks]`
