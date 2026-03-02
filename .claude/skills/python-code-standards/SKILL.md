---
description: "Use when writing or reviewing Python code for style, type hints, docstrings, naming conventions, or structured logging."
---

# Python Code Standards

Python code quality standards and conventions for this workspace.

## Quality Tools

- **PEP 8** compliance enforced via `ruff` (line length: 88)
- **Type hints** required on all functions, methods, class attributes
- **Docstrings** in Google style for all public APIs
- **Type checking** via `pyright` (standard mode)

## Code Formatting

| Rule | Standard |
|------|----------|
| Indentation | 4 spaces (never tabs) |
| Line length | 88 characters max |
| Blank lines | 2 between top-level definitions, 1 between methods |
| Whitespace | Space around operators (`=`, `+`), after commas, none inside brackets |

**Line wrapping**: Break after opening parenthesis or before binary operators.

## Naming Conventions

| Element | Style | Example |
|---------|-------|---------|
| Modules | `lowercase_snake_case` | `api_client.py` |
| Functions/Variables | `lowercase_snake_case` | `fetch_data`, `user_count` |
| Classes | `CamelCase` | `DataProcessor` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Private | `_leading_underscore` | `_internal_helper` |

## Import Order

Three groups separated by blank lines:
1. Standard library
2. Third-party
3. Local (from workspace)

## Type Hints

Use built-in generics (Python 3.13):

```python
items: list[str]           # not List[str]
mapping: dict[str, int]    # not Dict[str, int]
optional: str | None       # not Optional[str]
handler: Callable[[str, int], bool]
```

Always annotate return types explicitly, including `-> None`.

## Constants

```python
from typing import Final

MAX_RETRIES: Final[int] = 3
DEFAULT_REGION: Final[str] = "asia-southeast1"
```

## Module Exports

Define `__all__` in `__init__.py` for bricks with multiple public symbols:

```python
__all__ = ["fetch_data", "DataProcessor"]
```

## Function Design

- Keep functions under ~150 lines (including docstrings, type hints, and blank lines)
- Single return type (avoid `str | dict | None` unions where possible)
- Use keyword-only args for 3+ parameters:

```python
def fetch(*, url: str, timeout: int, retries: int) -> dict[str, Any]:
```

## Google Style Docstrings

```python
def fetch_data(url: str, timeout: int = 30) -> dict[str, Any]:
    """Fetch data from the specified URL.

    Args:
        url: The endpoint URL to fetch from.
        timeout: Request timeout in seconds.

    Returns:
        Parsed JSON response as dictionary.

    Raises:
        ConnectionError: If the request fails.
    """
```

## Pydantic Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")

    gcp_project_id: str
    environment: str = "sandbox"
    debug: bool = False
```

- Use `BaseSettings` for environment-driven configuration
- Use `model_config` (Pydantic v2), not `Config` inner class
- Type all fields

## Error Handling

```python
# Catch specific exceptions, not bare except
try:
    result = client.query(sql)
except google.api_core.exceptions.NotFound:
    logging.error(msg={"event": "Table not found", "payload": {"table": table_id}})
    raise
except Exception as e:
    logging.error(msg={"event": str(e), "payload": {"traceback": traceback.format_exc()}})
    raise
```

- Never use bare `except:`
- Always log with traceback before re-raising
- Prefer specific exception types over generic `Exception`

## Structured Logging

```python
logging.info(msg={"event": "Past-tense verb + object", "payload": {...}})
logging.error(msg={"event": str(e), "payload": {"traceback": traceback.format_exc()}})
```

- **event**: Past tense ("Fetched data", NOT "Fetching data")
- **payload**: Contextual data (no secrets/PII)
- **traceback**: Required in error logs
