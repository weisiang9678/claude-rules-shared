---
description: "Use when creating or modifying GitHub Actions workflows and CI/CD pipelines."
---

# GitHub Actions CI/CD

Create and maintain GitHub Actions workflows following project standards.

## Standard Setup Pattern

Every Python workflow MUST include these steps:

```yaml
steps:
  - name: "Check out GitHub repository"
    uses: actions/checkout@v4

  - name: "Set up Python"
    uses: actions/setup-python@v5
    with:
      python-version-file: "pyproject.toml"  # Never hardcode version

  - name: "Set up uv"
    uses: astral-sh/setup-uv@v5
    with:
      version: "0.7.8"  # Always pin exact version

  - name: "Install dependencies"
    run: uv sync --frozen  # Always use --frozen in CI/CD

  - name: "Activate virtualenv"
    run: echo "$PWD/.venv/bin" >> $GITHUB_PATH
```

**Exception:** Pulumi workflows using `toolchain: uv` (see [pulumi-infrastructure skill](../pulumi-infrastructure/SKILL.md)) do not need the "Install dependencies" and "Activate virtualenv" steps — `pulumi install` handles this automatically.

## Key Rules

| Rule | Do | Don't |
|------|-----|-------|
| Python version | `python-version-file: "pyproject.toml"` | `python-version: "3.13"` |
| uv version | `version: "0.7.8"` | `version: "latest"` |
| Sync deps | `uv sync --frozen` | `uv sync` |
| Caching | Don't cache uv | Use `actions/cache` |

## Workflow Templates

### Type Check

```yaml
name: Type Check

on:
  pull_request:
    branches: [main]

jobs:
  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"
      - uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"
      - run: uv sync --frozen
      - run: echo "$PWD/.venv/bin" >> $GITHUB_PATH
      - uses: jakebailey/pyright-action@v2
```

### Test

```yaml
name: Test

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"
      - uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"
      - run: uv sync --frozen
      - run: echo "$PWD/.venv/bin" >> $GITHUB_PATH
      - run: |
          uv run pytest test/ \
            --junitxml=pytest.xml \
            --cov-report=xml:coverage.xml \
            --cov=components \
            --cov=bases
```

### Release

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for semantic release
      - uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"
      - uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"
      - run: uv sync --frozen --group release
      - run: echo "$PWD/.venv/bin" >> $GITHUB_PATH

      - name: "Sync workspace member versions"
        run: |
          NEW_VERSION=$(semantic-release --noop version --print)
          if [ -n "$NEW_VERSION" ]; then
            echo "Syncing workspace members to version: ${NEW_VERSION}"
            for project in projects/*; do
              uv version --package $(basename "$project") "$NEW_VERSION" --frozen
            done
          fi

      - run: semantic-release version
      - run: semantic-release publish
```

## Dependency Groups

| Group | Use Case | Command |
|-------|----------|---------|
| (default) | Standard dev | `uv sync --frozen` |
| release | CI/CD deployment | `uv sync --frozen --group release` |
| test | Test-only jobs | `uv sync --frozen --group test` |

## Workflow Naming

All workflows MUST have both `name` and `run-name`:

```yaml
name: Infrastructure Provision
run-name: Provision ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }} infrastructure
```

## Step ID Convention

Use `verb-subject` in kebab-case when steps have outputs:

```yaml
- name: "Determine stack and version"
  id: determine-stack-version
  run: |
    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT
```

## Extracting Package Version from Lock File

Use `uv tree` to get a package version from `uv.lock` (e.g., for Pulumi version pinning):

```yaml
- name: "Extract Pulumi version from lock file"
  id: extract-pulumi-version
  run: |
    PULUMI_VERSION=$(uv tree --package pulumi --depth 0 --frozen | sed 's/.* v//')
    echo "PULUMI_VERSION=${PULUMI_VERSION}" >> $GITHUB_OUTPUT
    echo "Using Pulumi version: ${PULUMI_VERSION}"
```

This approach:
- Uses uv's native TOML parsing (more robust than grep)
- Uses `--frozen` to read from lock file without modification
- Works without running `uv sync` first

## Reference Files

- [workflow-templates.md](references/workflow-templates.md) - Complete workflow examples
- [common-patterns.md](references/common-patterns.md) - Docker build, GCP auth patterns
