## SPECIALIST F: Special Forms

### `Literal` — Exact Value Types

```python
from typing import Literal

Direction = Literal["north", "south", "east", "west"]
HttpMethod = Literal["GET", "POST", "PUT", "DELETE", "PATCH"]

def move(direction: Direction) -> None: ...

move("north")   # OK
move("up")      # type error
```

Use `Literal` for: HTTP methods, status strings, mode flags, fixed string enumerations. Prefer `enum.Enum` when the values need runtime behavior.

### `Final` — Constants and Non-Override

```python
from typing import Final

MAX_RETRIES: Final = 3           # inferred as Final[int]
API_URL: Final[str] = "https://api.example.com"

class Base:
    @final
    def critical_method(self) -> None: ...  # cannot be overridden

@final
class Singleton:  # cannot be subclassed
    ...
```

**Rule:** Use `Final` for all module-level constants. Use `@final` (from `typing`) on methods and classes that must not be overridden or subclassed.

### `ClassVar` — Class-Level Attributes

```python
from typing import ClassVar

class Config:
    instance_count: ClassVar[int] = 0  # shared across all instances
    name: str                           # instance attribute

    def __init__(self, name: str) -> None:
        Config.instance_count += 1
        self.name = name
```

`ClassVar` tells mypy that accessing `self.instance_count` is valid but `instance.__dict__` won't have it.

### `Annotated` — Metadata Attachment

```python
from typing import Annotated
from pydantic import Field

# Attach validation metadata (Pydantic, FastAPI use this)
UserId = Annotated[int, Field(gt=0, description="User primary key")]
Email = Annotated[str, Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")]

def get_user(user_id: UserId) -> User: ...
```

`Annotated[T, metadata]` is transparent to type checkers — the type is `T`. The metadata is available at runtime via `get_type_hints(include_extras=True)`. Libraries like Pydantic, FastAPI, and `dataclasses-json` use it for validation rules.

### `TypedDict` — Typed Dictionaries

Use when you must work with dicts that have a known shape (JSON APIs, config blobs, legacy code).

```python
from typing import TypedDict, Required, NotRequired

class UserPayload(TypedDict):
    name: str
    email: str
    role: NotRequired[str]  # optional key

# Alternative functional syntax (for keys that aren't valid identifiers)
Config = TypedDict("Config", {"timeout-ms": int, "retry": bool})

# Inheritance
class AdminPayload(UserPayload, total=False):
    permissions: list[str]
```

**Decision table — always default to Pydantic:**

| | Pydantic BaseModel | dataclass | TypedDict |
|--|-------------------|-----------|-----------|
| Runtime validation | **Yes** | No | No |
| Serialization | **Built-in** | Manual | Manual |
| Interop with dicts | `.model_dump()` / `.model_validate()` | No | Yes (IS a dict) |
| mypy support | **Full (via mypy-pydantic plugin)** | Full | Full |
| Use when | **Default for all structured data** | Internal unvalidated containers | Typing raw dict-shaped external APIs (legacy only) |

**Rule: If you're modeling data, use Pydantic. Period.**

### Pydantic — Typing Integration

Pydantic is the primary data modeling tool. Key typing patterns:

```python
from pydantic import BaseModel, Field, model_validator, field_validator
from typing import Annotated, Self

# Reusable constrained types via Annotated + Field
UserId = Annotated[int, Field(gt=0, description="User primary key")]
Email = Annotated[str, Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")]
NonEmptyStr = Annotated[str, Field(min_length=1)]

class User(BaseModel):
    id: UserId
    email: Email
    name: NonEmptyStr
    role: Literal["admin", "viewer"] = "viewer"
    deleted_at: datetime | None = None  # nullable only where domain requires it

    @field_validator("email")
    @classmethod
    def email_must_be_lowercase(cls, v: str) -> str:
        return v.lower()

    @model_validator(mode="after")
    def check_admin_has_no_deletion(self) -> Self:
        if self.role == "admin" and self.deleted_at is not None:
            raise ValueError("Admin accounts cannot be soft-deleted")
        return self

# Generic Pydantic models
class Page[T](BaseModel):
    items: list[T]
    total: int
    page: int
    size: int

# mypy plugin resolves Pydantic field types correctly — enable it:
# [tool.mypy]
# plugins = ["pydantic.mypy"]
```

**mypy + Pydantic plugin setup:**
```toml
[tool.mypy]
plugins = ["pydantic.mypy"]

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
warn_required_dynamic_aliases = true
```

### `NamedTuple` — Typed Named Tuples

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    label: str = ""  # with default

p = Point(1.0, 2.0)
p.x          # float
x, y, _ = p  # unpacking still works
```

Use `NamedTuple` only when tuple unpacking semantics are required. Otherwise use Pydantic.

### `Self` — Return Type for the Current Class

```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:
        self._name = name
        return self

    def set_age(self, age: int) -> Self:
        self._age = age
        return self

class SpecialBuilder(Builder):
    pass

# Without Self: Builder.set_name() would return Builder, not SpecialBuilder
# With Self: SpecialBuilder().set_name("Alice") → SpecialBuilder ✓
```

**Rule:** Use `Self` for all fluent/builder patterns and classmethods that return an instance.

```python
@classmethod
def from_config(cls, config: dict) -> Self:
    return cls(**config)
```

### `Never` / `NoReturn` — Unreachable Code

```python
from typing import Never, NoReturn

# NoReturn — function never returns (raises or loops forever)
def crash(msg: str) -> NoReturn:
    raise RuntimeError(msg)

# Never — a value that can never exist (exhaustiveness checking)
def assert_never(value: Never) -> Never:
    raise AssertionError(f"Unexpected value: {value!r}")
```

Use `assert_never` for exhaustive `match`/`if-elif` chains (see Specialist H).

### `LiteralString` — SQL Injection Prevention

```python
from typing import LiteralString

def execute_query(sql: LiteralString) -> list[dict]:
    # Only accepts string literals, not runtime-constructed strings
    ...

execute_query("SELECT * FROM users")          # OK
execute_query(f"SELECT * FROM {table_name}")  # type error — could be injection
```

---
