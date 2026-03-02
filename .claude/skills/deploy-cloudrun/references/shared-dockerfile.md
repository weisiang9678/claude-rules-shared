# Shared Dockerfile Template

Common Dockerfile pattern for all Cloud Run workload types (Job, Service, Function).

## Standard Template

```dockerfile
FROM python:3.13-slim@sha256:{digest}

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/

WORKDIR /app

# Copy project files for dependency resolution
COPY projects/{name}/pyproject.toml ./pyproject.toml
COPY uv.lock ./uv.lock

# Install dependencies
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy Polylith bricks
COPY components/{namespace}/{component1}/ ./{namespace}/{component1}/
COPY components/{namespace}/{component2}/ ./{namespace}/{component2}/
COPY bases/{namespace}/{base_name}/ ./{namespace}/{base_name}/
RUN mv {namespace}/{base_name}/core.py main.py

# Service/Function only: expose HTTP port
# ENV PORT=8080     # Function only: set PORT env var
# EXPOSE 8080

# Enable venv in PATH
ENV PATH="/app/.venv/bin:$PATH"

# CMD varies by type — see type-specific reference
CMD ["python", "main.py"]
```

## Key Principles

1. **Pin base image with digest** — Reproducible builds
2. **Copy pyproject.toml and uv.lock first** — Docker layer caching
3. **Use `--no-install-project`** — We copy bricks manually
4. **Use `--no-default-groups`** — Exclude dev/test dependencies
5. **Copy only needed bricks** — Match project's `[tool.polylith.bricks]`
6. **Build from workspace root** — Access all bricks
7. **Use trailing slashes on directory COPYs** — Ensures Docker treats destination as directory

## Type-Specific Differences

| Aspect | Job | Service | Function |
|--------|-----|---------|----------|
| Framework | Direct `python main.py` | App-specific (Flask, Streamlit, etc.) | `functions-framework` |
| CMD | `["python", "main.py"]` | Varies by framework | `exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}` |
| PORT/EXPOSE | Not needed | Required | Required |
| Entry point | Hardcoded | Hardcoded | Dynamic via `FUNCTION_TARGET` env var |
| System deps | Typically none | May need `apt-get` packages | Typically none |
| Static assets | None | May need config files, templates | None |

For type-specific details, see:
- [job-specifics.md](job-specifics.md) — no PORT/EXPOSE, direct `python main.py`
- [service-specifics.md](service-specifics.md) — CMD variations, system dependencies, static assets
- [function-specifics.md](function-specifics.md) — `functions-framework` CMD, `FUNCTION_TARGET`

## Build Command

Always run from workspace root:

```bash
docker build -f projects/{name}/Dockerfile -t {image} .
```
