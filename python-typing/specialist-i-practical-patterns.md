## SPECIALIST I: Practical Patterns

### `Any` — When Allowed, When Forbidden

`Any` disables all type checking for a value. It is contagious: `Any + X = Any`.

**Allowed uses of `Any`:**
- Interfacing with genuinely untyped third-party code where stubs don't exist
- `**kwargs: Any` when kwargs are passed through to another untyped function
- Prototype code with `# type: ignore[misc]` and a TODO
- `json.loads()` return value — the type IS `Any` because JSON is dynamic

**Forbidden uses:**
- Avoiding the work of writing a real type
- Working around a type error you don't understand
- Annotating a collection when `TypedDict` or a dataclass would be correct

```python
# BAD — suppresses all checking
def process(data: Any) -> Any: ...

# GOOD — use object for "any type, but I won't call methods on it"
def log(value: object) -> None:
    print(repr(value))

# GOOD — use TypeVar for "any type that I'll return"
def identity[T](value: T) -> T:
    return value
```

**`object` vs `Any`:** Use `object` when you accept any value but don't call methods on it. `object` is checked; `Any` is not.

### `cast()` — Asserting a Type to the Checker

```python
from typing import cast

# Use when you know the type but the checker can't infer it
result = cast(User, db.get("user:42"))

# Use sparingly — cast() does nothing at runtime, it's a lie to the checker
# Prefer isinstance + narrowing when possible
```

**Policy:** `cast()` is acceptable at system boundaries (deserialization, ORMs, FFI). Never use it to silence a type error in business logic — that's a signal the code or types are wrong.

### `reveal_type()` — Debugging

```python
x = some_complex_expression()
reveal_type(x)  # mypy prints: Revealed type is "list[str]"
```

Available in both mypy and pyright. Remove before committing (ruff `RUF010` flags it).

### `TYPE_CHECKING` Block

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myproject.heavy import ExpensiveClass
    from collections.abc import Sequence
```

**Rules:**
- Imports inside `TYPE_CHECKING` are never executed at runtime
- Pair with `from __future__ import annotations` to make forward references safe
- Do not put in `TYPE_CHECKING` anything needed at runtime (e.g., for `isinstance`)
- Type checkers validate these imports — they must be real, resolvable paths

### Stub Files (`.pyi`)

Stub files provide type information for modules that cannot be annotated directly (C extensions, generated code, third-party libraries).

```python
# myproject/fast_core.pyi — types for a C extension
def compute(values: list[float], precision: int = ...) -> float: ...

class Optimizer:
    def __init__(self, lr: float) -> None: ...
    def step(self) -> None: ...
```

**When to write stubs:**
- Wrapping a C extension you own
- Adding types to a third-party package for your team (private stubs in your own package)
- Contributing to `typeshed` for widely-used untyped libraries

**When to use `py.typed`:** Add an empty `py.typed` file to your package's root (in `src/mypackage/`) to signal to type checkers that the package is typed (PEP 561). Without it, mypy ignores your inline annotations when the package is installed.

### Typing Third-Party Libraries Without Stubs

When a library has no stubs and no `py.typed`:

```python
# pyproject.toml
[[tool.mypy.overrides]]
module = ["untyped_lib.*", "another_untyped.*"]
ignore_missing_imports = true
```

Search for community stubs first: `pip install types-<libname>` (from `typeshed`). Examples: `types-requests`, `types-PyYAML`, `types-redis`.

---
