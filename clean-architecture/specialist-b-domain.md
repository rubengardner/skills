## SPECIALIST B: Domain Layer

### Domain Models

Domain models are pure Pydantic `BaseModel`. No ORM. No framework. No I/O. They enforce invariants through Pydantic validators.

```python
# apps/orders/domain/models.py
from __future__ import annotations

from datetime import datetime
from decimal import Decimal
from enum import StrEnum

from pydantic import BaseModel, Field, model_validator
from typing import Self

from apps.orders.domain.value_objects import Money, OrderId, CustomerId
from apps.orders.domain.exceptions import OrderCannotBeCancelledError


class OrderStatus(StrEnum):
    DRAFT = "draft"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"


class OrderLine(BaseModel):
    product_id: int = Field(gt=0)
    quantity: int = Field(gt=0)
    unit_price: Money

    model_config = {"frozen": True}

    @property
    def subtotal(self) -> Money:
        return Money(amount=self.unit_price.amount * self.quantity,
                     currency=self.unit_price.currency)


class Order(BaseModel):
    id: OrderId
    customer_id: CustomerId
    status: OrderStatus
    lines: list[OrderLine]
    created_at: datetime
    cancelled_at: datetime | None = None

    model_config = {"frozen": True}

    @model_validator(mode="after")
    def lines_not_empty_when_confirmed(self) -> Self:
        if self.status == OrderStatus.CONFIRMED and not self.lines:
            raise ValueError("A confirmed order must have at least one line")
        return self

    @property
    def total(self) -> Money:
        if not self.lines:
            return Money(amount=Decimal("0"), currency="USD")
        first_currency = self.lines[0].unit_price.currency
        total_amount = sum(line.subtotal.amount for line in self.lines)
        return Money(amount=total_amount, currency=first_currency)

    def cancel(self) -> Order:
        """Return a new Order in the cancelled state. Domain rule enforced here."""
        if self.status not in (OrderStatus.DRAFT, OrderStatus.CONFIRMED):
            raise OrderCannotBeCancelledError(
                order_id=self.id,
                current_status=self.status,
            )
        return self.model_copy(
            update={"status": OrderStatus.CANCELLED,
                    "cancelled_at": datetime.utcnow()}
        )
```

**Rules:**
- `model_config = {"frozen": True}` on all domain models — they are immutable values.
- Business operations that change state return a **new** instance (`model_copy`). Never mutate.
- All invariant enforcement lives in validators or domain methods.
- Domain models never hold database PKs in ORM format — they hold typed IDs from value objects.

### Value Objects

Value objects are immutable, self-validating, identity-free. Two value objects with the same data are equal.

```python
# apps/orders/domain/value_objects.py
from __future__ import annotations

from decimal import Decimal
from typing import Annotated

from pydantic import BaseModel, Field, field_validator

# Typed aliases — use NewType for IDs to prevent mixing
from typing import NewType

OrderId = NewType("OrderId", int)
CustomerId = NewType("CustomerId", int)
ProductId = NewType("ProductId", int)


class Money(BaseModel):
    """Value object: monetary amount + currency.

    Equal when amount and currency are equal, regardless of object identity.
    """

    amount: Decimal = Field(ge=Decimal("0"))
    currency: str = Field(min_length=3, max_length=3, pattern="^[A-Z]{3}$")

    model_config = {"frozen": True}

    @field_validator("amount", mode="before")
    @classmethod
    def coerce_to_decimal(cls, v: object) -> Decimal:
        if isinstance(v, float):
            return Decimal(str(v))
        return Decimal(v)  # type: ignore[arg-type]

    def __add__(self, other: Money) -> Money:
        if self.currency != other.currency:
            raise ValueError(
                f"Cannot add {self.currency} and {other.currency}"
            )
        return Money(amount=self.amount + other.amount, currency=self.currency)

    def __mul__(self, factor: int | Decimal) -> Money:
        return Money(amount=self.amount * Decimal(factor), currency=self.currency)


class EmailAddress(BaseModel):
    value: str = Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")

    model_config = {"frozen": True}

    @field_validator("value", mode="after")
    @classmethod
    def normalize(cls, v: str) -> str:
        return v.lower().strip()

    def __str__(self) -> str:
        return self.value
```

**Rules:**
- Always `frozen = True`.
- Override `__add__`, `__mul__`, etc. only for mathematically meaningful operations.
- Use `NewType` (not `TypeAlias`) for typed IDs — `NewType` creates a distinct type that mypy checks. `OrderId(5)` and `CustomerId(5)` are incompatible under mypy strict even though both are `int` at runtime.
- Never embed ORM PKs or database row concepts in a value object.

### Domain Services

A domain service contains domain logic that does not naturally fit on a single entity. It is a pure function or a stateless class. It never performs I/O.

```python
# apps/orders/domain/services.py
from __future__ import annotations

from apps.orders.domain.models import Order, OrderLine, OrderStatus
from apps.orders.domain.value_objects import Money
from apps.orders.domain.exceptions import InsufficientInventoryError


def calculate_order_discount(order: Order, discount_rate: float) -> Money:
    """Apply a percentage discount to an order total.

    This is domain logic that operates on the Order aggregate.
    Pure function — no I/O, no state.
    """
    if not 0.0 <= discount_rate <= 1.0:
        raise ValueError(f"discount_rate must be in [0, 1], got {discount_rate}")
    total = order.total
    discount_amount = total.amount * round(discount_rate, 4)  # type: ignore[arg-type]
    return Money(amount=discount_amount, currency=total.currency)


def apply_inventory_reservation(
    order: Order,
    available_quantities: dict[int, int],
) -> None:
    """Validate that all order lines can be fulfilled.

    Raises InsufficientInventoryError if any line exceeds available stock.
    This is a domain rule — it belongs in the domain layer, not in a use case.
    """
    for line in order.lines:
        available = available_quantities.get(line.product_id, 0)
        if line.quantity > available:
            raise InsufficientInventoryError(
                product_id=line.product_id,
                requested=line.quantity,
                available=available,
            )
```

**Rules:**
- Domain services are stateless. No `__init__` with dependencies.
- They receive domain objects and return domain objects or raise domain exceptions.
- If a service needs I/O (fetching data), it is an **application** use case, not a domain service.

### Domain Events

Domain events record something that happened. They are immutable facts. They do not carry behavior.

```python
# apps/orders/domain/events.py
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

from apps.orders.domain.value_objects import OrderId, CustomerId


@dataclass(frozen=True)
class DomainEvent:
    """Base class for all domain events."""

    event_id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default_factory=datetime.utcnow)


@dataclass(frozen=True)
class OrderCreated(DomainEvent):
    order_id: OrderId
    customer_id: CustomerId
    total_amount: str  # serialized as string to keep events framework-free


@dataclass(frozen=True)
class OrderCancelled(DomainEvent):
    order_id: OrderId
    reason: str


@dataclass(frozen=True)
class OrderShipped(DomainEvent):
    order_id: OrderId
    tracking_number: str
```

**How to raise domain events (without a framework):**

The use case collects events returned from domain operations and dispatches them after the transaction commits.

```python
# application/use_cases/cancel_order.py
from dataclasses import dataclass
from apps.orders.domain.events import OrderCancelled
from apps.orders.domain.repositories import OrderRepository
from apps.orders.application.dtos import CancelOrderCommand


@dataclass
class CancelOrderUseCase:
    order_repository: OrderRepository
    event_publisher: "EventPublisher"  # infrastructure interface, injected

    def execute(self, command: CancelOrderCommand) -> None:
        order = self.order_repository.get_by_id(command.order_id)
        if order is None:
            from apps.orders.application.exceptions import OrderNotFoundError
            raise OrderNotFoundError(command.order_id)

        cancelled_order = order.cancel()  # domain method raises if invalid
        self.order_repository.save(cancelled_order)

        event = OrderCancelled(order_id=cancelled_order.id, reason=command.reason)
        self.event_publisher.publish(event)
```

**Rules:**
- Domain events are immutable dataclasses with `frozen=True`.
- They use `uuid4` event IDs and `datetime.utcnow()` timestamps — no framework time utilities.
- Events are collected during the transaction and dispatched after commit. Never dispatch before the transaction commits.
- The `EventPublisher` interface lives in the domain or application layer. The implementation (Celery task, Redis Pub/Sub, Django signals wrapper) is infrastructure.

### Repository Interfaces

The repository interface is defined in the domain layer. It describes what the application needs — nothing more. The implementation is in the infrastructure layer.

```python
# apps/orders/domain/repositories.py
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import Protocol

from apps.orders.domain.models import Order
from apps.orders.domain.value_objects import OrderId, CustomerId


class OrderRepository(Protocol):
    """Interface for Order persistence.

    Any implementation must satisfy this structural type.
    The domain knows nothing about HOW orders are stored.
    """

    def get_by_id(self, order_id: OrderId) -> Order | None: ...

    def save(self, order: Order) -> None: ...

    def delete(self, order_id: OrderId) -> None: ...

    def find_by_customer(
        self,
        customer_id: CustomerId,
        *,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Order]: ...


class EventPublisher(Protocol):
    """Interface for publishing domain events."""

    def publish(self, event: "DomainEvent") -> None: ...  # type: ignore[name-defined]
```

**Protocol vs ABC:** Use `Protocol` for repository interfaces. Protocol enables structural subtyping — the implementation does not need to inherit from the interface. It only needs to satisfy the method signatures. Use `ABC` when you want `@abstractmethod` enforcement and shared base implementation.

### Domain Exceptions

```python
# apps/orders/domain/exceptions.py
from __future__ import annotations

from apps.orders.domain.value_objects import OrderId, ProductId


class DomainError(Exception):
    """Base for all domain errors. Never catch this in domain code."""


class OrderNotFoundError(DomainError):
    def __init__(self, order_id: OrderId) -> None:
        self.order_id = order_id
        super().__init__(f"Order {order_id} not found")


class OrderCannotBeCancelledError(DomainError):
    def __init__(self, order_id: OrderId, current_status: str) -> None:
        self.order_id = order_id
        self.current_status = current_status
        super().__init__(
            f"Order {order_id} cannot be cancelled in status '{current_status}'"
        )


class InsufficientInventoryError(DomainError):
    def __init__(
        self,
        product_id: ProductId,
        requested: int,
        available: int,
    ) -> None:
        self.product_id = product_id
        self.requested = requested
        self.available = available
        super().__init__(
            f"Product {product_id}: requested {requested}, only {available} available"
        )
```

**Rules:**
- All domain exceptions inherit from `DomainError`, not from application or infrastructure exceptions.
- Store structured data as instance attributes — not only in the message string.
- Domain exceptions are caught and translated at the infrastructure boundary (the router). They never propagate as HTTP errors from the domain itself.
- Hierarchy: `DomainError → ConcreteError`. Maximum 2–3 levels.

---
