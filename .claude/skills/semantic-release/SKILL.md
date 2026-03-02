---
description: "Use when writing commit messages, creating releases, or understanding version bumps with conventional commits."
---

# Semantic Release

Manage automated versioning and releases using conventional commits.

## Commit Message Format

See Core Rules §7 for format, types, version bumps, and subject rules.

Additional type for this project:
- `perf` → PATCH (1.0.0 → 1.0.1)

## Examples

### Feature
```
feat(pipeline): add CDC event deduplication

Implement deduplication logic to prevent duplicate CDC events from
being published to Pub/Sub. Uses in-memory cache with LRU eviction.

Closes #142
```

### Bug Fix
```
fix(logging): correct sentry DSN configuration

Previously used wrong DSN for production environment.

Fixes #156
```

### Breaking Change
```
feat(settings)!: migrate to pydantic v2 settings

BREAKING CHANGE: Settings class now uses Pydantic v2 API.
Update all code using `Config` inner class to use `model_config`.
```

### Simple Changes (No Body)
```
docs: update installation guide
ci: add type checking workflow
chore: update dependencies
```

## How Releases Work

1. Commits pushed to `main` branch
2. Semantic release analyzes commit messages
3. Version bump determined from commit types
4. **Workspace members updated** with `uv version --package`
5. **Root version updated** by `semantic-release version`
6. CHANGELOG.md generated
7. Git tag created (e.g., v1.2.3)
8. GitHub release published

### Workspace Version Management

This project uses a **uv workspace** with multiple projects in `projects/*`. Version updates happen in two stages within the Release workflow:

**Stage 1 — Sync workspace member versions:**
```bash
NEW_VERSION=$(semantic-release --noop version --print)
for project in projects/*; do
  uv version --package $(basename "$project") "$NEW_VERSION" --frozen
done
```

`semantic-release --noop version --print` determines the next version from commit history without making changes. `uv version --package` stamps that version into each project's `pyproject.toml`.

**Stage 2 — Bump root, commit, tag, and publish:**
```bash
semantic-release version  # Updates root pyproject.toml, commits all changes, creates tag
semantic-release publish  # Creates GitHub release
```

`semantic-release version` stages and commits **all** modified `pyproject.toml` files (root + projects), so the version sync is atomic in a single release commit.

**Ordering is critical:** Stage 1 must run before Stage 2. See the Release workflow in the `github-actions-cicd` skill for the complete integration.

## Configuration

In workspace root `pyproject.toml`:

```toml
[tool.semantic_release]
version_variable = ["pyproject.toml:project.version"]
version_toml = ["pyproject.toml:project.version"]
branch = "main"
upload_to_pypi = false
upload_to_release = true
```

## Pull Request Conventions

PR titles follow the same format as commit messages:

```
<type>(<scope>): <description>
```

**PR Description Template:**
```markdown
## Summary
Brief description of what this PR does.

## Changes
- Bullet point list of changes

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass

## Related Issues
Closes #123
```

## Common Mistakes

See [common-mistakes.md](references/common-mistakes.md) for examples.
