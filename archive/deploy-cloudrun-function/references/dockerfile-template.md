# Dockerfile Template for Cloud Run Functions

## Standard Template

```dockerfile
FROM python:3.13-slim@sha256:{digest}

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/

WORKDIR /app

# Copy project files for dependency resolution
COPY projects/{function_name}/pyproject.toml ./pyproject.toml
COPY uv.lock ./uv.lock

# Install dependencies
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy Polylith bricks
COPY components/{namespace}/{component1}/ ./{namespace}/{component1}/
COPY components/{namespace}/{component2}/ ./{namespace}/{component2}/
COPY bases/{namespace}/{base_name}/ ./{namespace}/{base_name}/
RUN mv {namespace}/{base_name}/core.py main.py

ENV PORT=8080
EXPOSE 8080

# Enable venv in PATH so functions-framework can be found
ENV PATH="/app/.venv/bin:$PATH"

# Run with functions-framework
CMD exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}
```

## Key Principles

1. **Pin base image with digest** — Reproducible builds
2. **Copy pyproject.toml and uv.lock first** — Docker layer caching
3. **Use `--no-install-project`** — We copy bricks manually
4. **Use `--no-default-groups`** — Exclude dev/test dependencies
5. **Copy only needed bricks** — Match project's `[tool.polylith.bricks]`
6. **Build from workspace root** — Access all bricks
7. **Use trailing slashes on directory COPYs** — Ensures Docker treats destination as directory

## Differences from Service and Job

| Aspect | Function | Service | Job |
|--------|----------|---------|-----|
| Framework | `functions-framework` | App-specific (Flask, Streamlit, etc.) | Direct `python main.py` |
| CMD | `exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}` | Varies by framework | `["python", "main.py"]` |
| PORT/EXPOSE | Required | Required | Not needed |
| Entry point | Dynamic via `FUNCTION_TARGET` env var | Hardcoded | Hardcoded |

## Build Command

Always run from workspace root:

```bash
docker build -f projects/{function_name}/Dockerfile -t {image} .
```
