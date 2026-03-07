## SPECIALIST E: Protocols

### Structural vs Nominal Subtyping

Python's type system supports both:
- **Nominal:** `class Dog(Animal)` — explicit inheritance
- **Structural (Protocol):** "if it has `.bark()` and `.name`, it's a Dog-like" — duck typing made explicit

### Defining a Protocol

```python
from typing import Protocol, runtime_checkable

class Closeable(Protocol):
    def close(self) -> None: ...

class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

# Composing protocols
class ReadableCloseable(Readable, Closeable, Protocol): ...
```

Any class that implements `close() -> None` satisfies `Closeable` — no explicit inheritance needed.

### `@runtime_checkable`

```python
@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...

# Enables isinstance checks — but only checks method existence, not signatures
assert isinstance(circle, Drawable)  # True if circle has .draw()
```

**Warning:** `@runtime_checkable` `isinstance` checks only verify attribute existence, not signature compatibility. Do not rely on it for security or correctness — it's a convenience, not a guarantee.

### Protocol vs ABC

| Use Protocol when | Use ABC when |
|-------------------|-------------|
| Defining a structural interface for duck typing | You need shared implementation via `super()` |
| Third-party types should satisfy it without modification | You control the class hierarchy |
| The interface crosses package boundaries | You want to enforce implementation via `@abstractmethod` |
| Describing existing behavior (e.g., file-like objects) | Building a framework with extension points |

### Standard Library Protocols to Know

From `collections.abc` — these are the Protocols Python itself uses:

```python
from collections.abc import (
    Iterable,      # __iter__
    Iterator,      # __iter__ + __next__
    Sequence,      # __getitem__ + __len__ (ordered, indexed)
    Mapping,       # __getitem__ + __len__ + __iter__
    MutableMapping,
    Callable,      # __call__
    Hashable,      # __hash__
    Sized,         # __len__
    Awaitable,     # __await__
    AsyncIterable, # __aiter__
    Generator,     # send + throw + close
)
```

**Rule:** Always prefer these standard protocols over inventing your own when the semantic fits.

---
