## SPECIALIST C: Application Layer

### Use-Case Class Pattern

Every use case is a class with a single public method: `execute()`. It receives a typed command or query object, calls domain services and repositories through injected interfaces, and returns a typed result or raises.

```python
# apps/orders/application/use_cases/create_order.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime

from apps.orders.application.dtos import CreateOrderCommand, OrderResult
from apps.orders.application.exceptions import CustomerNotFoundError
from apps.orders.domain.models import Order, OrderLine, OrderStatus
from apps.orders.domain.repositories import OrderRepository, EventPublisher
from apps.orders.domain.value_objects import OrderId, CustomerId, Money
from apps.orders.domain.events import OrderCreated

# Interface — only the interface is imported; implementation is injected
from apps.users.domain.repositories import UserRepository


@dataclass
class CreateOrderUseCase:
    """Create a new order for a customer.

    Dependencies are injected through the DI container.
    This class has zero knowledge of Django, databases, or HTTP.
    """

    order_repository: OrderRepository
    user_repository: UserRepository
    event_publisher: EventPublisher

    def execute(self, command: CreateOrderCommand) -> OrderResult:
        # 1. Validate preconditions using injected interfaces
        if not self.user_repository.exists(command.customer_id):
            raise CustomerNotFoundError(command.customer_id)

        # 2. Build domain model — no ORM here
        order = Order(
            id=OrderId(0),  # 0 = unsaved; the repository assigns the real ID
            customer_id=command.customer_id,
            status=OrderStatus.DRAFT,
            lines=[
                OrderLine(
                    product_id=line.product_id,
                    quantity=line.quantity,
                    unit_price=Money(
                        amount=line.unit_price,
                        currency=command.currency,
                    ),
                )
                for line in command.lines
            ],
            created_at=datetime.utcnow(),
        )

        # 3. Persist — repository returns the saved order with real ID
        saved_order = self.order_repository.save(order)

        # 4. Publish domain events after persistence
        self.event_publisher.publish(
            OrderCreated(
                order_id=saved_order.id,
                customer_id=saved_order.customer_id,
                total_amount=str(saved_order.total.amount),
            )
        )

        # 5. Return application DTO
        return OrderResult.from_domain(saved_order)
```

**Rules for use-case classes:**
- Always a `@dataclass`. Dependencies injected through `__init__` fields.
- Single public method: `execute(command: CommandType) -> ResultType`.
- No Django imports. No ORM imports. No HTTP client calls directly.
- Never instantiate infrastructure classes inside a use case.
- `execute()` should be testable by passing fake repositories.

### Command vs Query — CQRS Lite

Commands mutate state. Queries read state. Keep them separate.

```python
# apps/orders/application/dtos.py
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from datetime import datetime

from pydantic import BaseModel, Field

from apps.orders.domain.models import Order, OrderStatus
from apps.orders.domain.value_objects import OrderId, CustomerId


# ── COMMANDS (write side) ────────────────────────────────────────────────────

@dataclass(frozen=True)
class CreateOrderLineCommand:
    product_id: int
    quantity: int
    unit_price: Decimal


@dataclass(frozen=True)
class CreateOrderCommand:
    customer_id: CustomerId
    currency: str
    lines: list[CreateOrderLineCommand]


@dataclass(frozen=True)
class CancelOrderCommand:
    order_id: OrderId
    reason: str


# ── QUERIES (read side) ──────────────────────────────────────────────────────

@dataclass(frozen=True)
class GetOrderQuery:
    order_id: OrderId
    requesting_customer_id: CustomerId


@dataclass(frozen=True)
class ListOrdersQuery:
    customer_id: CustomerId
    limit: int = 50
    offset: int = 0


# ── RESULTS ──────────────────────────────────────────────────────────────────

class OrderResult(BaseModel):
    """Read-side DTO returned by queries and command results."""

    id: int
    customer_id: int
    status: OrderStatus
    total_amount: Decimal
    total_currency: str
    created_at: datetime

    @classmethod
    def from_domain(cls, order: Order) -> OrderResult:
        return cls(
            id=order.id,
            customer_id=order.customer_id,
            status=order.status,
            total_amount=order.total.amount,
            total_currency=order.total.currency,
            created_at=order.created_at,
        )


class OrderListResult(BaseModel):
    items: list[OrderResult]
    total: int
    limit: int
    offset: int
```

**Rules:**
- Commands are `@dataclass(frozen=True)`. They represent intent — what the caller wants to do.
- Queries are `@dataclass(frozen=True)`. They carry filtering/pagination parameters.
- Results are Pydantic `BaseModel`. They are the output contract of the use case.
- Application DTOs never expose ORM types. They map from domain models using `from_domain()`.

### Application Exceptions

Application exceptions are distinct from domain exceptions. They handle orchestration-level failures — preconditions the use case must check that do not belong in the domain.

```python
# apps/orders/application/exceptions.py
from __future__ import annotations

from apps.orders.domain.value_objects import CustomerId, OrderId


class ApplicationError(Exception):
    """Base for application-level errors."""


class CustomerNotFoundError(ApplicationError):
    """The customer referenced in a command does not exist.

    This is an application-level check, not a domain invariant.
    The domain model does not know how to verify customer existence —
    that requires a repository, which is infrastructure.
    """

    def __init__(self, customer_id: CustomerId) -> None:
        self.customer_id = customer_id
        super().__init__(f"Customer {customer_id} not found")


class OrderAccessDeniedError(ApplicationError):
    """The requesting customer does not own this order."""

    def __init__(self, order_id: OrderId, requesting_customer_id: CustomerId) -> None:
        self.order_id = order_id
        self.requesting_customer_id = requesting_customer_id
        super().__init__(
            f"Customer {requesting_customer_id} cannot access order {order_id}"
        )
```

### Transaction Management — Abstraction

The application layer declares transaction boundaries through a `UnitOfWork` interface. The infrastructure layer implements it using Django's transaction utilities.

```python
# apps/orders/domain/repositories.py  (or application/ports.py)
from __future__ import annotations

from typing import Protocol


class UnitOfWork(Protocol):
    """Declares a transactional boundary.

    The application layer calls begin/commit/rollback through this interface.
    The implementation (Django, SQLAlchemy, etc.) is in infrastructure.
    """

    def __enter__(self) -> "UnitOfWork": ...

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: object,
    ) -> bool | None: ...

    def commit(self) -> None: ...

    def rollback(self) -> None: ...
```

Use cases that span multiple repositories use UoW:

```python
# apps/orders/application/use_cases/ship_order.py
from __future__ import annotations

from dataclasses import dataclass

from apps.orders.application.dtos import ShipOrderCommand
from apps.orders.application.exceptions import OrderNotFoundError
from apps.orders.domain.repositories import OrderRepository, EventPublisher, UnitOfWork
from apps.inventory.domain.repositories import InventoryRepository


@dataclass
class ShipOrderUseCase:
    order_repository: OrderRepository
    inventory_repository: InventoryRepository
    event_publisher: EventPublisher
    unit_of_work: UnitOfWork

    def execute(self, command: ShipOrderCommand) -> None:
        with self.unit_of_work:
            order = self.order_repository.get_by_id(command.order_id)
            if order is None:
                raise OrderNotFoundError(command.order_id)

            shipped_order = order.ship(tracking_number=command.tracking_number)
            self.order_repository.save(shipped_order)

            # Adjust inventory atomically in the same transaction
            self.inventory_repository.reserve(
                product_ids=[line.product_id for line in shipped_order.lines]
            )

            self.unit_of_work.commit()

        # Publish events AFTER transaction commit — outside the `with` block
        from apps.orders.domain.events import OrderShipped
        self.event_publisher.publish(
            OrderShipped(
                order_id=shipped_order.id,
                tracking_number=command.tracking_number,
            )
        )
```

### Application DTOs vs Domain Models — When to Map

| Scenario | Decision |
|----------|----------|
| Use case returns data consumed only by one infrastructure adapter | Skip mapping — return domain model |
| Use case returns data that multiple adapters consume differently | Map to DTO — decouple from domain model shape |
| Query result needs pagination metadata | Map to DTO — domain model has none |
| Command write: simple save and return ID | Return `OrderId` directly — no full DTO needed |
| Complex projection (aggregate data from multiple models) | DTO — domain model cannot represent it |

Do not create DTOs for the sake of it. If the domain model shape is the correct return value, return it directly. Only introduce a DTO when the shapes diverge or the use case result needs metadata the domain model cannot carry.

---
