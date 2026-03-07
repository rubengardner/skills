## SPECIALIST I: Patterns & Anti-patterns

### Common Violations

**Violation 1: Domain importing ORM**

```python
# WRONG — domain model importing ORM
# apps/orders/domain/models.py
from apps.orders.infrastructure.models import OrderDB  # NEVER

class Order(BaseModel):
    db_instance: OrderDB  # exposes ORM to domain
```

```python
# CORRECT — domain knows nothing of ORM
class Order(BaseModel):
    id: OrderId
    customer_id: CustomerId
    # Pure data; no ORM reference
```

**Violation 2: Use case importing Django**

```python
# WRONG — use case using Django ORM directly
from apps.orders.infrastructure.models import OrderDB

class CreateOrderUseCase:
    def execute(self, command):
        OrderDB.objects.create(...)  # NEVER — ORM in application layer
```

```python
# CORRECT — use case calls repository interface
class CreateOrderUseCase:
    order_repository: OrderRepository

    def execute(self, command):
        saved = self.order_repository.save(order)  # interface call only
```

**Violation 3: Router containing business logic**

```python
# WRONG — business logic in router
@router.post("/")
def create_order(request, payload: OrderRequestAPI):
    if not UserDB.objects.filter(pk=payload.customer_id).exists():
        raise HttpError(404, "Customer not found")
    if len(payload.lines) == 0:
        raise HttpError(400, "Need at least one line")
    # ... more logic
```

```python
# CORRECT — router delegates to use case immediately
@router.post("/")
def create_order(request, payload: OrderRequestAPI):
    use_case = order_container.create_order_use_case()
    result = use_case.execute(CreateOrderCommand(...))
    return 201, OrderResponseAPI.from_result(result)
```

**Violation 4: Anemic domain model**

An anemic domain model has data but no behavior. All logic lives in services or use cases. This is the most common violation.

```python
# WRONG — domain model is a dumb data container
class Order(BaseModel):
    id: int
    status: str
    lines: list[OrderLine]

# All logic elsewhere
def cancel_order(order: Order) -> Order:  # this is domain logic that belongs on Order
    if order.status not in ("draft", "confirmed"):
        raise ValueError("Cannot cancel")
    return order.model_copy(update={"status": "cancelled"})
```

```python
# CORRECT — domain model contains its own behavior
class Order(BaseModel):
    id: OrderId
    status: OrderStatus
    lines: list[OrderLine]

    def cancel(self) -> Order:
        """Domain logic lives on the aggregate. cancel() belongs here."""
        if self.status not in (OrderStatus.DRAFT, OrderStatus.CONFIRMED):
            raise OrderCannotBeCancelledError(self.id, self.status)
        return self.model_copy(update={"status": OrderStatus.CANCELLED})
```

**Violation 5: Repository returning ORM instances**

```python
# WRONG — leaks ORM into application
class DjangoOrderRepository:
    def get_by_id(self, order_id: int) -> OrderDB | None:  # NEVER return ORM
        return OrderDB.objects.get(pk=order_id)
```

```python
# CORRECT — always map to domain model before returning
class DjangoOrderRepository:
    def get_by_id(self, order_id: OrderId) -> Order | None:
        try:
            db = OrderDB.objects.prefetch_related("lines").get(pk=order_id)
        except OrderDB.DoesNotExist:
            return None
        return self._to_domain(db)  # always map
```

**Violation 6: Infrastructure exception escaping the boundary**

```python
# WRONG — Django DoesNotExist propagates to caller
class DjangoOrderRepository:
    def get_by_id(self, order_id: OrderId) -> Order:
        return self._to_domain(OrderDB.objects.get(pk=order_id))
        # ^ OrderDB.DoesNotExist can leak out — infrastructure exception in application
```

```python
# CORRECT — infrastructure exceptions are caught and mapped at the boundary
class DjangoOrderRepository:
    def get_by_id(self, order_id: OrderId) -> Order | None:
        try:
            return self._to_domain(OrderDB.objects.get(pk=order_id))
        except OrderDB.DoesNotExist:
            return None  # caller handles None, not infrastructure exceptions
```

### How to Enforce Layer Boundaries — import-linter

Install `import-linter` and declare contracts in `pyproject.toml`:

```toml
# pyproject.toml
[tool.importlinter]
root_packages = ["apps"]

[[tool.importlinter.contracts]]
name = "Domain layer has no outward imports"
type = "independence"
modules = [
    "apps.orders.domain",
]
ignore_imports = []

[[tool.importlinter.contracts]]
name = "Application layer does not import infrastructure"
type = "forbidden"
source_modules = [
    "apps.orders.application",
]
forbidden_modules = [
    "apps.orders.infrastructure",
    "django",
    "ninja",
]

[[tool.importlinter.contracts]]
name = "Domain layer does not import application or infrastructure"
type = "forbidden"
source_modules = [
    "apps.orders.domain",
]
forbidden_modules = [
    "apps.orders.application",
    "apps.orders.infrastructure",
    "django",
    "ninja",
    "httpx",
]
```

Add to CI:

```yaml
- run: uv run lint-imports
```

### When to Be Pragmatic vs Strict

| Situation | Be strict | Be pragmatic |
|-----------|-----------|-------------|
| Core business domain (billing, inventory, orders) | Yes | No |
| Peripheral CRUD (user profile photo URL) | Optional | Yes — thin use case or direct ORM in router |
| Rapidly-changing requirements | Strict boundaries survive change | — |
| Prototype / spike | — | Skip layers entirely; rebuild when committed |
| Thin resource with no business logic | — | Use a selector directly in the router |

The rule for pragmatism: make the shortcut explicit and localized. Add a `# ARCH: skipping use case — pure CRUD` comment so future readers know it was a deliberate choice.

### Mapping Overhead

Mapping between domain models and ORM models is boilerplate. Reduce it with factory methods, but never skip it by returning ORM instances.

```python
# Keep mappers co-located with the repository, not scattered
class DjangoOrderRepository:
    @staticmethod
    def _to_domain(db: OrderDB) -> Order:
        """Private mapper. Never called outside this class."""
        ...

    @staticmethod
    def _to_db(order: Order) -> dict[str, object]:
        """Mapping to ORM kwargs for create/update."""
        return {
            "customer_id": int(order.customer_id),
            "status": order.status.value,
            "cancelled_at": order.cancelled_at,
        }
```

If mapping becomes a significant proportion of code, consider a dedicated mapper file:

```
infrastructure/
    mappers.py   # All _to_domain and _to_db functions for this bounded context
    repositories.py   # Imports from mappers.py
```

---
