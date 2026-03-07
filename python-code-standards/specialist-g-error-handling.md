## SPECIALIST G: Error Handling

### Exception Hierarchy (Required for every library/application)

```python
# src/myproject/exceptions.py

class MyProjectError(Exception):
    """Base exception. Catching this catches all library-specific errors."""

class NotFoundError(MyProjectError):
    def __init__(self, resource_type: str, identifier: object) -> None:
        self.resource_type = resource_type
        self.identifier = identifier
        super().__init__(f"{resource_type} not found: {identifier!r}")

class ValidationError(MyProjectError):
    """Input data failed validation."""

class AuthorizationError(MyProjectError):
    """Operation not permitted for this caller."""

class ExternalServiceError(MyProjectError):
    """Third-party service call failed unexpectedly."""
```

**Hierarchy rules:**
- Inherit from `Exception`, never `BaseException`
- Keep hierarchy shallow (2–3 levels max)
- Always suffix with `Error`
- Store structured data as instance attributes, not just in the message

### Catch vs Propagate

```python
# NEVER — catches SystemExit, KeyboardInterrupt
try:
    ...
except:
    pass

# BAD — too broad, hides real errors
try:
    result = do_something()
except Exception:
    return None

# GOOD — specific, contextual, chained
try:
    result = call_external_api(user_id)
except httpx.TimeoutException as exc:
    logger.warning("API timeout for user %s", user_id)
    raise ExternalServiceError("API timed out") from exc
except httpx.HTTPStatusError as exc:
    if exc.response.status_code == 404:
        raise NotFoundError("user", user_id) from exc
    raise ExternalServiceError(f"API returned {exc.response.status_code}") from exc
```

**Decision rules:**
- Catch at the boundary where you can meaningfully handle or translate
- Always `raise X from exc` to preserve exception chain
- Never swallow silently — at minimum `logger.warning`
- Catch `Exception` only at process boundaries (API handlers, CLI entrypoints)
- Use `logger.exception()` inside `except` — auto-includes traceback

### Exception Groups (Python 3.11+)

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_users())
        tg.create_task(fetch_orders())
except* TimeoutError as eg:
    logger.error("Timeouts in %d tasks", len(eg.exceptions))
```

Note: `except*` and `except` cannot mix in the same `try` block.

### Logging Standards

- **Use `structlog`** for new production services. Standard `logging` requires too much configuration for structured output.
- **Use module-level logger:** `logger = structlog.get_logger(__name__)`
- **Never `print()`** for non-CLI output (ruff T201 flags this)

| Level | When |
|-------|------|
| DEBUG | Developer-facing detail; off in production |
| INFO | Normal operational events |
| WARNING | Unexpected but non-fatal; retry/fallback used |
| ERROR | Operation failed; requires investigation |
| CRITICAL | System-level failure |

---
