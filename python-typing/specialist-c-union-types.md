## SPECIALIST C: Union Types

### The Rule

**Use `X | Y` syntax for all union types on Python 3.10+. Use `X | None` instead of `Optional[X]`.**

```python
# BAD — deprecated
from typing import Optional, Union

def find(id: int) -> Optional[User]: ...
def process(value: Union[str, int]) -> None: ...

# GOOD — 3.10+
def find(id: int) -> User | None: ...
def process(value: str | int) -> None: ...
```

Ruff rules `UP007` (Union → |) and `UP045` (Optional → | None) enforce this.

### With `from __future__ import annotations` (3.7–3.9 compat)

If you must support Python 3.9 but want `X | Y` syntax in annotations, add:
```python
from __future__ import annotations
```
This makes all annotations strings, so `X | Y` doesn't fail at runtime on 3.9.

### `None` Is Not Optional by Default

**Do not use `X | None` as a default for "might not be set." Design types that are non-nullable where possible.**

```python
# BAD design — forces callers to handle None everywhere
class User:
    name: str | None
    email: str | None
    created_at: datetime | None

# GOOD — nullable only where the domain requires it
class User:
    name: str
    email: str
    created_at: datetime
    deleted_at: datetime | None  # legitimately nullable: user might not be deleted
```

### Union Order

Order union members from most specific to least specific. Type checkers don't require this, but it aids human readers:

```python
# Preferred order: specific → general
def serialize(value: int | float | str | None) -> str: ...
```

### Never Union with `Any`

`X | Any` collapses to `Any` — the union is meaningless. If you find yourself writing this, the problem is elsewhere.

---
