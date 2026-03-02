# Dockerfile Template for Cloud Run Services

## Standard Template

```dockerfile
FROM python:3.13-slim@sha256:{digest}

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/

WORKDIR /app

# Copy project files for dependency resolution
COPY projects/{service_name}/pyproject.toml ./pyproject.toml
COPY uv.lock ./uv.lock

# Install dependencies
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy Polylith bricks
COPY components/{namespace}/{component1}/ ./{namespace}/{component1}/
COPY components/{namespace}/{component2}/ ./{namespace}/{component2}/
COPY bases/{namespace}/{base_name}/ ./{namespace}/{base_name}/

# Copy static assets (if needed)
# COPY .streamlit/ ./.streamlit/

RUN mv {namespace}/{base_name}/core.py main.py

EXPOSE 8080

# Enable venv in PATH
ENV PATH="/app/.venv/bin:$PATH"
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

## System Dependencies

If the service needs system packages, add `apt-get` before uv installation:

```dockerfile
FROM python:3.13-slim@sha256:{digest}

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends ca-certificates curl

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/
# ... rest of Dockerfile
```

## CMD Variations

Services may use different entry points depending on the framework:

```dockerfile
# Direct Python execution
CMD ["python", "main.py"]

# Streamlit
CMD ["streamlit", "run", "main.py", "--server.port=8080", "--server.address=0.0.0.0"]

# Flask / Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "main:app"]
```

## Differences from Function and Job

| Aspect        | Service                               | Function                                                              | Job                     |
| ------------- | ------------------------------------- | --------------------------------------------------------------------- | ----------------------- |
| Framework     | App-specific (Flask, Streamlit, etc.) | `functions-framework`                                                 | Direct `python main.py` |
| CMD           | Varies by framework                   | `exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}` | `["python", "main.py"]` |
| System deps   | May need `apt-get` packages           | Typically none                                                        | Typically none          |
| Static assets | May need config files, templates      | None                                                                  | None                    |
| PORT/EXPOSE   | Required                              | Required                                                              | Not needed              |

## Build Command

Always run from workspace root:

```bash
docker build -f projects/{service_name}/Dockerfile -t {image} .
```
