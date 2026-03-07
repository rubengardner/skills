## SPECIALIST G: Callable Typing

### Basic Callable

```python
from collections.abc import Callable

# Callable[[arg_types], return_type]
def apply(fn: Callable[[int, str], bool], value: int) -> bool:
    return fn(value, "default")

# No arguments
def run(task: Callable[[], None]) -> None:
    task()

# Unknown/variable arguments — use with caution
def call_any(fn: Callable[..., int]) -> int:
    return fn()
```

### Async Callables

```python
from collections.abc import Callable, Coroutine
from typing import Any

# Async function type
AsyncHandler = Callable[[Request], Coroutine[Any, Any, Response]]

# Or use the Awaitable shorthand
from collections.abc import Awaitable
AsyncTask = Callable[[int], Awaitable[str]]
```

### `overload` — Multiple Signatures

Use `@overload` when a function returns different types based on argument types. Never use `Union` in the return type when the return type depends on which argument type was passed.

```python
from typing import overload

@overload
def parse(value: str) -> int: ...
@overload
def parse(value: bytes) -> str: ...
@overload
def parse(value: int) -> float: ...

def parse(value: str | bytes | int) -> int | str | float:
    if isinstance(value, str):
        return int(value)
    elif isinstance(value, bytes):
        return value.decode()
    else:
        return float(value)
```

**Rules for `@overload`:**
- At least two `@overload` variants required; the implementation is not exported
- The implementation signature must be broad enough to accept all overloaded signatures
- Overloads must be in the same file, directly above the implementation
- Do not put logic in overload stubs — they contain `...` only

### `ParamSpec` for Decorator Typing

See Specialist D. Essential pattern:

```python
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def logged[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper
```

### `Concatenate` — Prepending Arguments to ParamSpec

```python
from typing import Concatenate

# For decorators that inject an extra argument (e.g., request object)
def with_auth[**P, R](
    fn: Callable[Concatenate[User, P], R]
) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        user = get_current_user()
        return fn(user, *args, **kwargs)
    return wrapper

@with_auth
def delete_post(user: User, post_id: int) -> None: ...

# Caller does not pass user — it's injected
delete_post(post_id=42)
```

---
