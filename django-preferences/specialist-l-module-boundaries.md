## SPECIALIST L: Module Boundaries

### The Rule

Each Django app is a **bounded context**. Modules never import each other's internals. All cross-module interaction goes through a typed `Client` class. The public surface of a module is:

1. The `Client` class (`apps/<module>/client.py`)
2. Types explicitly re-exported via `apps/<module>/__init__.py`

Everything else (models, schemas, services, selectors, routers) is **internal** — invisible to the rest of the project.

### The Client Pattern

```python
# apps/users/client.py
from __future__ import annotations
from dataclasses import dataclass
from apps.users.schemas import User          # domain model — flat, no suffix
from apps.users.selectors import get_user_by_id, user_exists
from apps.users.services import deactivate_user

# Export ID type through the client module
UserId = int


@dataclass(frozen=True)
class UserClient:
    """Public interface for the users module.

    Other modules ONLY interact with this class. Never import from
    apps.users.models, apps.users.services, or apps.users.selectors directly.
    """

    @staticmethod
    def get(user_id: UserId) -> User | None:
        """Return the domain model (not UserDB) or None."""
        db_instance = get_user_by_id(user_id)
        if db_instance is None:
            return None
        return User(
            id=db_instance.pk,
            username=db_instance.username,
            email=db_instance.email,
            is_active=db_instance.is_active,
        )

    @staticmethod
    def exists(user_id: UserId) -> bool:
        return user_exists(user_id)

    @staticmethod
    def deactivate(user_id: UserId) -> None:
        deactivate_user(user_id)
```

### `__init__.py` — The Public Surface

```python
# apps/users/__init__.py
from apps.users.client import UserClient, UserId

__all__ = ["UserClient", "UserId"]
```

This is the **only** file other modules are allowed to import from:

```python
# apps/orders/services.py — CORRECT cross-module import
from apps.users import UserClient, UserId

def create_order(customer_id: UserId, ...) -> Order:
    if not UserClient.exists(customer_id):
        raise ValueError(f"User {customer_id} does not exist")
    ...
```

```python
# apps/orders/services.py — WRONG — imports internal module detail
from apps.users.models import UserDB          # NEVER
from apps.users.selectors import get_user    # NEVER
from apps.users.schemas import UserResponseAPI  # NEVER
```

### What the Client Returns

The `Client` always returns **domain models** (flat Pydantic, no `DB` suffix) — never ORM instances. This prevents leaking ORM-specific behaviour (lazy loading, QuerySet chaining) across module boundaries.

```python
# UserClient.get() → User (domain model)  ✓
# UserClient.get() → UserDB (ORM instance) ✗  — leaks internals
```

### ID Types Through the Client

Each module exports its ID types through `__init__.py`:

```python
# apps/users/__init__.py
from apps.users.client import UserClient, UserId

# apps/orders/__init__.py
from apps.orders.client import OrderClient, OrderId
```

Other modules import ID types from the module's public surface:

```python
from apps.users import UserId    # correct
from apps.orders import OrderId  # correct
```

### No ForeignKeys Across Modules

Because modules are isolated bounded contexts, cross-module ForeignKeys are forbidden. They create hidden coupling via Django migrations and prevent modules from being extracted later.

```python
# WRONG — cross-module ForeignKey
class OrderDB(models.Model):
    customer = models.ForeignKey("users.UserDB", on_delete=models.CASCADE)  # NEVER

# CORRECT — store ID only; validate existence via client
class OrderDB(models.Model):
    customer_id = models.IntegerField(db_index=True)  # plain ID field

# Validate at service layer:
from apps.users import UserClient
if not UserClient.exists(customer_id):
    raise ValueError("Customer not found")
```

### Intra-module Imports Are Free

Inside a module, import anything freely:

```python
# apps/orders/services.py — within the orders module — fine
from apps.orders.models import OrderDB
from apps.orders.schemas import Order, OrderRequestAPI
from apps.orders.selectors import get_order_by_id
```

### Summary: Module Boundary Contract

```
Module boundary = __init__.py surface

Allowed imports from another module:
  from apps.users import UserClient   ✓
  from apps.users import UserId       ✓

Forbidden:
  from apps.users.models import UserDB        ✗
  from apps.users.services import ...         ✗
  from apps.users.selectors import ...        ✗
  from apps.users.schemas import ...          ✗
  from apps.users.router import ...           ✗
```

---

## GOTCHA REFERENCE

| Gotcha | Fix |
|--------|-----|
| Multiple `NinjaAPI()` without unique `version`/`urls_namespace` | Assign distinct `version` or `urls_namespace` to each instance |
| `ModelSchema` with `fields = "__all__"` exposes password | Always use explicit `fields = [...]` on `<Domain>ResponseAPI` |
| Returning unevaluated async QuerySet | Materialize: `[x async for x in qs]` |
| Nested schema triggers N+1 | `select_related` / `prefetch_related` in selector |
| `@permission_required` above `@router.get` — no `request.auth` yet | Place permission decorator **below** route decorator |
| PATCH silently nulls columns | Use `payload.model_dump(exclude_unset=True)` |
| `annotate()` fields missing from `ModelSchema` | Use plain `Schema` with explicit annotated field |
| PUT/PATCH file uploads empty | Add `ninja.compatibility.files.fix_request_files_middleware` |
| Async test hangs | Use `@pytest.mark.django_db(transaction=True)` |
| `CONN_MAX_AGE > 0` with Django 5.1 native pooling | Must set `CONN_MAX_AGE = 0` |
| Replacing `objects` with custom manager | Keep `objects = models.Manager()`; add scoped managers as named attributes |

---
