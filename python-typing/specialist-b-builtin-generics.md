## SPECIALIST B: Built-in Generics

### The Rule

**Use built-in collection types directly for all annotations on Python 3.9+. Never use `typing.List`, `typing.Dict`, `typing.Tuple`, `typing.Set`, `typing.FrozenSet`, `typing.Type` for new code.**

`typing.List` et al. are deprecated as of Python 3.9 (PEP 585) and will eventually be removed. Ruff rule `UP006` and `UP035` flag these.

### Migration Table

| Old (deprecated) | New (use this) | Notes |
|-----------------|----------------|-------|
| `List[X]` | `list[X]` | 3.9+ |
| `Dict[K, V]` | `dict[K, V]` | 3.9+ |
| `Tuple[X, Y]` | `tuple[X, Y]` | 3.9+ |
| `Set[X]` | `set[X]` | 3.9+ |
| `FrozenSet[X]` | `frozenset[X]` | 3.9+ |
| `Type[X]` | `type[X]` | 3.9+ |
| `Deque[X]` | `collections.deque[X]` | 3.9+ |
| `DefaultDict[K,V]` | `collections.defaultdict[K,V]` | 3.9+ |
| `Optional[X]` | `X \| None` | 3.10+ |
| `Union[X, Y]` | `X \| Y` | 3.10+ |
| `Callable[[A], R]` | `Callable[[A], R]` | still from `collections.abc` |

### Tuple Semantics

`tuple` has two distinct forms — the distinction is strict:

```python
# Fixed-length, heterogeneous — preferred for return values with multiple types
def get_coords() -> tuple[float, float]: ...
def parse() -> tuple[str, int, bool]: ...

# Variable-length, homogeneous — use when length is unknown
def get_tags() -> tuple[str, ...]: ...

# Empty tuple
def noop() -> tuple[()]: ...
```

### Use `collections.abc` for Abstract Types

Prefer abstract types in function parameters — accept the widest type that works:

```python
from collections.abc import Sequence, Mapping, Iterable, Iterator, Generator, Callable

# BAD — forces caller to use a list specifically
def process(items: list[str]) -> None: ...

# GOOD — accepts list, tuple, str, any sequence
def process(items: Sequence[str]) -> None: ...

# BAD — forces caller to use a dict specifically
def config(settings: dict[str, str]) -> None: ...

# GOOD
def config(settings: Mapping[str, str]) -> None: ...
```

**Concrete types (`list`, `dict`, `set`) in return types are fine** — they give callers more information about what they receive.

### `collections.abc` vs `typing` Aliases

```python
# Both correct on 3.9+, prefer collections.abc for runtime usability
from collections.abc import Sequence, Callable, Iterator  # preferred
from typing import Sequence, Callable, Iterator           # deprecated aliases, avoid
```

---
