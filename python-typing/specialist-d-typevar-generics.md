## SPECIALIST D: TypeVar & Generics

### TypeVar — Classic Syntax (3.11 and below)

```python
from typing import TypeVar

T = TypeVar("T")                          # unconstrained
T_co = TypeVar("T_co", covariant=True)   # covariant
T_contra = TypeVar("T_contra", contravariant=True)

# Constrained — T must be exactly one of these types
AnyStr = TypeVar("AnyStr", str, bytes)

# Bounded — T must be a subtype of User
UserT = TypeVar("UserT", bound="User")
```

### TypeVar — New Syntax (Python 3.12+)

```python
# 3.12+ generic functions — no TypeVar declaration needed
def first[T](items: list[T]) -> T:
    return items[0]

def transform[T, U](items: list[T], fn: Callable[[T], U]) -> list[U]:
    return [fn(item) for item in items]

# 3.12+ generic classes
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

# 3.12+ with bound
def largest[T: (int, float)](items: list[T]) -> T:
    return max(items)
```

**Rule:** Use 3.12+ syntax for new code targeting Python 3.12+. Keep classic `TypeVar` for projects supporting 3.10/3.11.

### Variance

| Variance | Meaning | Use when |
|----------|---------|---------|
| Invariant (default) | `Container[Dog]` is NOT a `Container[Animal]` | Mutable containers |
| Covariant (`_co`) | `Container[Dog]` IS a `Container[Animal]` | Read-only/producer types |
| Contravariant (`_contra`) | `Container[Animal]` IS a `Container[Dog]` | Write-only/consumer types |

```python
# Covariant: you only READ T out of it
class ReadOnlyStack[T_co](covariant=True):  # 3.12+ syntax
    def peek(self) -> T_co: ...

# Contravariant: you only WRITE T into it
class Sink[T_contra](contravariant=True):
    def write(self, value: T_contra) -> None: ...
```

**Practical rule:** Most custom generic classes are invariant. Use variance only when you can articulate why the container is read-only or write-only.

### ParamSpec — Preserving Callable Signatures

`ParamSpec` captures the full signature (parameter names, types, defaults) of a callable. Essential for typing decorators that preserve the wrapped function's signature.

```python
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

# 3.12+ syntax
def retry[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        for attempt in range(3):
            try:
                return fn(*args, **kwargs)
            except Exception:
                if attempt == 2:
                    raise
        raise RuntimeError("unreachable")
    return wrapper

@retry
def fetch_user(user_id: int, *, timeout: float = 30.0) -> User:
    ...

# Type checker knows: fetch_user(user_id=1, timeout=5.0) is valid
# fetch_user("wrong") → type error
```

### TypeVarTuple — Variadic Generics

For functions that preserve the types of variable-length argument tuples (array shapes, pipeline stages):

```python
from typing import TypeVarTuple, Unpack

Ts = TypeVarTuple("Ts")

def zip_with[**Ts](*iterables: Unpack[Ts]) -> ...: ...
```

**Practical use:** Primarily for numpy shape typing and pipeline frameworks. Rarely needed in application code.

---
