## SPECIALIST G: Testing Strategy

### Layer Testing Matrix

| Layer | Test Type | Dependencies | Speed | DB? |
|-------|-----------|-------------|-------|-----|
| Domain | Pure unit | None — no mocks | ~1 ms | No |
| Application | Unit with fakes | Fake repositories | ~5 ms | No |
| Infrastructure | Integration | Real DB (pytest-django) | ~100 ms | Yes |
| API (router) | Integration | TestClient + real DB | ~200 ms | Yes |

### Domain Tests — Pure Unit

No mocks, no fixtures, no DB. Just call domain code and assert.

```python
# tests/unit/domain/test_order.py
from decimal import Decimal
from datetime import datetime
import pytest

from apps.orders.domain.models import Order, OrderLine, OrderStatus
from apps.orders.domain.value_objects import Money, OrderId, CustomerId, ProductId
from apps.orders.domain.exceptions import OrderCannotBeCancelledError


def make_order(status: OrderStatus = OrderStatus.DRAFT) -> Order:
    return Order(
        id=OrderId(1),
        customer_id=CustomerId(10),
        status=status,
        lines=[
            OrderLine(
                product_id=ProductId(100),
                quantity=2,
                unit_price=Money(amount=Decimal("9.99"), currency="USD"),
            )
        ],
        created_at=datetime(2026, 1, 1),
    )


def test_order_total_is_sum_of_lines() -> None:
    order = make_order()
    assert order.total == Money(amount=Decimal("19.98"), currency="USD")


def test_cancel_draft_order_returns_cancelled_order() -> None:
    order = make_order(status=OrderStatus.DRAFT)
    cancelled = order.cancel()
    assert cancelled.status == OrderStatus.CANCELLED
    assert cancelled.cancelled_at is not None


def test_cancel_shipped_order_raises() -> None:
    order = make_order(status=OrderStatus.SHIPPED)
    with pytest.raises(OrderCannotBeCancelledError) as exc_info:
        order.cancel()
    assert exc_info.value.current_status == "shipped"


def test_money_addition_same_currency() -> None:
    a = Money(amount=Decimal("10.00"), currency="USD")
    b = Money(amount=Decimal("5.50"), currency="USD")
    assert a + b == Money(amount=Decimal("15.50"), currency="USD")


def test_money_addition_different_currencies_raises() -> None:
    a = Money(amount=Decimal("10.00"), currency="USD")
    b = Money(amount=Decimal("10.00"), currency="EUR")
    with pytest.raises(ValueError, match="Cannot add"):
        _ = a + b
```

### Fake Repositories — Application Tests

Fakes are hand-written in-memory implementations of repository interfaces. Never use `unittest.mock.MagicMock` for repositories in application tests — mocks do not verify behaviour, only call counts.

```python
# tests/fakes/order_repository.py
from __future__ import annotations

from apps.orders.domain.models import Order
from apps.orders.domain.value_objects import OrderId, CustomerId


class FakeOrderRepository:
    """In-memory implementation of OrderRepository for use-case tests.

    Behaviour: incrementing integer IDs, persistence by reference.
    """

    def __init__(self) -> None:
        self._store: dict[int, Order] = {}
        self._next_id: int = 1
        self.saved_orders: list[Order] = []  # audit list for assertions

    def get_by_id(self, order_id: OrderId) -> Order | None:
        return self._store.get(int(order_id))

    def save(self, order: Order) -> Order:
        if order.id == 0:
            # New entity — assign ID
            real_id = OrderId(self._next_id)
            self._next_id += 1
            saved = order.model_copy(update={"id": real_id})
        else:
            saved = order
        self._store[int(saved.id)] = saved
        self.saved_orders.append(saved)
        return saved

    def delete(self, order_id: OrderId) -> None:
        self._store.pop(int(order_id), None)

    def find_by_customer(
        self,
        customer_id: CustomerId,
        *,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Order]:
        matches = [
            o for o in self._store.values()
            if o.customer_id == customer_id
        ]
        return matches[offset : offset + limit]

    def exists(self, order_id: OrderId) -> bool:
        return int(order_id) in self._store


# tests/fakes/user_repository.py
from apps.orders.domain.value_objects import CustomerId


class FakeUserRepository:
    def __init__(self) -> None:
        self._existing_ids: set[int] = set()

    def add_existing_user_id(self, user_id: CustomerId) -> None:
        self._existing_ids.add(int(user_id))

    def exists(self, user_id: CustomerId) -> bool:
        return int(user_id) in self._existing_ids


# tests/fakes/event_publisher.py
from apps.orders.domain.events import DomainEvent


class FakeEventPublisher:
    def __init__(self) -> None:
        self.published_events: list[DomainEvent] = []

    def publish(self, event: DomainEvent) -> None:
        self.published_events.append(event)
```

### Infrastructure Integration Tests

Use `pytest-django` and `@pytest.mark.django_db`. Test the real ORM repository against a real test database. Do not mock the ORM.

```python
# tests/integration/infrastructure/test_order_repository.py
import pytest
from decimal import Decimal
from datetime import datetime

from apps.orders.domain.models import Order, OrderLine, OrderStatus
from apps.orders.domain.value_objects import Money, OrderId, CustomerId, ProductId
from apps.orders.infrastructure.repositories import DjangoOrderRepository


@pytest.fixture
def repo() -> DjangoOrderRepository:
    return DjangoOrderRepository()


@pytest.fixture
def sample_order() -> Order:
    return Order(
        id=OrderId(0),  # 0 = new
        customer_id=CustomerId(1),
        status=OrderStatus.DRAFT,
        lines=[
            OrderLine(
                product_id=ProductId(10),
                quantity=3,
                unit_price=Money(amount=Decimal("15.00"), currency="USD"),
            )
        ],
        created_at=datetime(2026, 1, 1),
    )


@pytest.mark.django_db
def test_save_and_get_order(repo: DjangoOrderRepository, sample_order: Order) -> None:
    saved = repo.save(sample_order)

    assert saved.id != 0
    retrieved = repo.get_by_id(saved.id)
    assert retrieved is not None
    assert retrieved.customer_id == CustomerId(1)
    assert len(retrieved.lines) == 1
    assert retrieved.total == Money(amount=Decimal("45.00"), currency="USD")


@pytest.mark.django_db
def test_get_nonexistent_order_returns_none(repo: DjangoOrderRepository) -> None:
    result = repo.get_by_id(OrderId(999999))
    assert result is None


@pytest.mark.django_db
def test_save_updated_order(repo: DjangoOrderRepository, sample_order: Order) -> None:
    saved = repo.save(sample_order)
    cancelled = saved.cancel()
    updated = repo.save(cancelled)

    retrieved = repo.get_by_id(updated.id)
    assert retrieved is not None
    assert retrieved.status == OrderStatus.CANCELLED
```

---
