## SPECIALIST A: Annotation Mechanics

### Runtime vs Type-Check Time

Annotations serve two distinct audiences that must not be conflated:

- **Runtime (CPython):** Annotations are evaluated eagerly at definition time and stored in `__annotations__`. Forward references to undefined names raise `NameError` immediately.
- **Type checkers (mypy, pyright):** Annotations are read as text and resolved symbolically. Forward references are always safe from their perspective.

```python
# WRONG on Python ≤3.13 without future import — NameError at load time
class Node:
    def children(self) -> list[Node]: ...  # Node not yet fully defined
```

### PEP 563 — `from __future__ import annotations`

Makes all annotations strings (lazy). Defers evaluation to inspection time only.

**Rule: Add this import to every module that uses forward references or TYPE_CHECKING.**

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myproject.models import User  # never imported at runtime
```

**Status (2026):** PEP 563 is stable and safe on 3.10–3.13. PEP 649 (lazy evaluation without stringification) was accepted for 3.14 but is not yet widely deployed — do not rely on it.

**Critical caveat:** `from __future__ import annotations` breaks any code that introspects annotations at runtime (e.g., `get_type_hints()` called without `localns`/`globalns`, Pydantic v1 model definitions without `model_rebuild()`). Pydantic v2 and `dataclasses` handle this correctly.

### When to Annotate

**Always annotate:**
- Every public function signature (parameters + return type)
- Every public class attribute
- Every module-level constant that is not obvious from the literal value

**Skip annotation when:**
- Local variables with obvious types: `count = 0`, `items = []` ← type is inferred
- `self` and `cls` parameters (mypy strict disables `ANN101`/`ANN102`)
- Comprehension loop variables

**Annotate local variables when:**
- The type is not inferable: `result: list[User] = []`
- You want to constrain a broader inference: `x: int | None = some_call()`

### Variable Annotations

```python
# Module-level constant
MAX_CONNECTIONS: Final[int] = 100

# Class attribute (not set in __init__)
class Config:
    debug: ClassVar[bool] = False

# Instance attribute declared outside __init__ — use __init__ instead when possible
class Service:
    client: httpx.AsyncClient  # declared here, assigned in __init__

    def __init__(self) -> None:
        self.client = httpx.AsyncClient()
```

---
