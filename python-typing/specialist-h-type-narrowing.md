## SPECIALIST H: Type Narrowing

### `isinstance` Narrowing

The most common and reliable narrowing mechanism:

```python
def process(value: str | int | None) -> str:
    if value is None:
        return "empty"
    if isinstance(value, int):
        return str(value)
    return value.upper()  # value is str here — mypy knows this
```

### `assert` Narrowing

```python
def get_user(id: int) -> User | None: ...

user = get_user(42)
assert user is not None  # narrows to User
user.name  # OK
```

**Caution:** `assert` is stripped by `-O` (optimize flag). Use explicit `if user is None: raise` for production code paths.

### `TypeGuard` — Custom Narrowing Functions (3.10+)

```python
from typing import TypeGuard

def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

items: list[object] = get_items()
if is_string_list(items):
    items  # narrowed to list[str]
```

**`TypeGuard` semantics:** When the function returns `True`, the argument is narrowed. The narrowed type must be a subtype of the original (or not — TypeGuard is intentionally permissive about this, which is sometimes a footgun).

### `TypeIs` — Stricter Narrowing (3.13+)

`TypeIs` is the safer replacement for `TypeGuard`:

```python
from typing import TypeIs

def is_str(val: object) -> TypeIs[str]:
    return isinstance(val, str)
```

**Difference from `TypeGuard`:** `TypeIs` requires the narrowed type to be a subtype of the original. It also narrows in the negative branch. Prefer `TypeIs` over `TypeGuard` on 3.13+.

### Exhaustiveness Checking with `Never`

Force the type checker to verify you've handled all cases in a union:

```python
from typing import Never

type Status = Literal["active", "inactive", "pending"]

def handle_status(status: Status) -> str:
    match status:
        case "active":
            return "running"
        case "inactive":
            return "stopped"
        case "pending":
            return "waiting"
        case _ as unreachable:
            assert_never(unreachable)  # type error if any case is missing

def assert_never(value: Never) -> Never:
    raise AssertionError(f"Unhandled value: {value!r}")
```

If you add `"archived"` to `Status` without adding a `case "archived"` branch, mypy reports an error.

### Narrowing with `Literal`

```python
Mode = Literal["read", "write"]

def open_file(path: str, mode: Mode) -> IO[bytes]:
    if mode == "read":
        mode  # narrowed to Literal["read"]
    ...
```

---
