## SPECIALIST F: Repository Pattern

### Interface Contract Rules

```python
# apps/orders/domain/repositories.py

class OrderRepository(Protocol):

    # ── Naming Conventions ───────────────────────────────────────────────────

    def get_by_id(self, order_id: OrderId) -> Order | None: ...
    # get_by_<field> — returns single entity or None; never raises

    def find_by_customer(
        self,
        customer_id: CustomerId,
        *,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Order]: ...
    # find_by_<field> — returns a collection; never raises; returns empty list if none

    def save(self, order: Order) -> Order: ...
    # save — handles both insert and update; returns the persisted entity with real ID

    def delete(self, order_id: OrderId) -> None: ...
    # delete — idempotent; does not raise if not found

    def exists(self, order_id: OrderId) -> bool: ...
    # exists — cheap existence check without loading the full entity
```

**Method naming convention:**

| Pattern | Semantics |
|---------|-----------|
| `get_by_<field>` | Return single entity or `None` |
| `find_by_<field>` | Return list (may be empty) |
| `save` | Insert or update; return persisted entity |
| `delete` | Remove; idempotent |
| `exists` | Boolean existence check |
| `count_by_<field>` | Numeric aggregate |

**Return types:**
- Always return domain models (`Order`), never ORM instances (`OrderDB`).
- `get_by_*` returns `Entity | None`. The caller decides whether to raise on `None`.
- Never raise a domain `NotFoundError` from a repository. Raise from the use case.

### Generic Repository Base

Use a generic base for repositories that share common behaviour across multiple entities.

```python
# domain/repositories.py
from __future__ import annotations

from typing import TypeVar, Generic, Protocol

from apps.orders.domain.value_objects import OrderId

EntityT = TypeVar("EntityT")
IdT = TypeVar("IdT")


class Repository(Protocol[EntityT, IdT]):
    """Generic repository protocol."""

    def get_by_id(self, entity_id: IdT) -> EntityT | None: ...
    def save(self, entity: EntityT) -> EntityT: ...
    def delete(self, entity_id: IdT) -> None: ...
    def exists(self, entity_id: IdT) -> bool: ...
```

### Async Repository

For async Django endpoints, the repository interface and implementation must be async.

```python
# apps/orders/domain/repositories.py

class AsyncOrderRepository(Protocol):
    async def get_by_id(self, order_id: OrderId) -> Order | None: ...
    async def save(self, order: Order) -> Order: ...
    async def delete(self, order_id: OrderId) -> None: ...
    async def find_by_customer(
        self,
        customer_id: CustomerId,
        *,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Order]: ...


# apps/orders/infrastructure/repositories.py

class AsyncDjangoOrderRepository:
    async def get_by_id(self, order_id: OrderId) -> Order | None:
        try:
            db_order = await (
                OrderDB.objects
                .prefetch_related("lines")
                .aget(pk=order_id)
            )
        except OrderDB.DoesNotExist:
            return None
        return self._to_domain(db_order)

    async def save(self, order: Order) -> Order:
        if order.id == 0:
            db_order = await OrderDB.objects.acreate(
                customer_id=order.customer_id,
                status=order.status,
            )
            await OrderLineDB.objects.abulk_create([
                OrderLineDB(order=db_order, ...)
                for line in order.lines
            ])
        else:
            await OrderDB.objects.filter(pk=order.id).aupdate(status=order.status)
            db_order = await OrderDB.objects.prefetch_related("lines").aget(pk=order.id)
        return self._to_domain(db_order)
```

**Rule:** Do not mix sync and async repository methods. If any use case is async, all repositories it uses must be async. Do not call sync ORM methods from async code without `sync_to_async`.

---
