## SPECIALIST E: Dependency Injection

### Manual DI Container

No third-party DI library. No FastAPI `Depends`. The container is a plain Python class that constructs and caches dependencies manually.

```python
# apps/orders/infrastructure/container.py
from __future__ import annotations

from functools import cached_property

from apps.orders.application.use_cases.create_order import CreateOrderUseCase
from apps.orders.application.use_cases.cancel_order import CancelOrderUseCase
from apps.orders.application.use_cases.get_order import GetOrderUseCase
from apps.orders.infrastructure.repositories import DjangoOrderRepository
from apps.orders.infrastructure.unit_of_work import DjangoUnitOfWork
from apps.orders.infrastructure.http.stripe_payment_gateway import StripePaymentGateway
from apps.orders.infrastructure.event_publisher import SyncEventPublisher

# Cross-module dependency — import through public interface only
from apps.users.infrastructure.repositories import DjangoUserRepository


class OrderContainer:
    """Manual DI container for the orders bounded context.

    Singletons: repositories (stateless), external clients (maintain connection pools).
    Transient: use cases (cheap to construct, should not hold request state).
    """

    # ── Singletons (cached_property = lazy singleton) ────────────────────────

    @cached_property
    def order_repository(self) -> DjangoOrderRepository:
        return DjangoOrderRepository()

    @cached_property
    def user_repository(self) -> DjangoUserRepository:
        return DjangoUserRepository()

    @cached_property
    def event_publisher(self) -> SyncEventPublisher:
        return SyncEventPublisher()

    @cached_property
    def payment_gateway(self) -> StripePaymentGateway:
        from django.conf import settings
        return StripePaymentGateway(
            api_key=settings.STRIPE_API_KEY.get_secret_value()
        )

    # ── Transient use cases (new instance per call) ──────────────────────────

    def create_order_use_case(self) -> CreateOrderUseCase:
        return CreateOrderUseCase(
            order_repository=self.order_repository,
            user_repository=self.user_repository,
            event_publisher=self.event_publisher,
        )

    def cancel_order_use_case(self) -> CancelOrderUseCase:
        return CancelOrderUseCase(
            order_repository=self.order_repository,
            unit_of_work=DjangoUnitOfWork(),  # new UoW per use case
            event_publisher=self.event_publisher,
        )

    def get_order_use_case(self) -> GetOrderUseCase:
        return GetOrderUseCase(
            order_repository=self.order_repository,
        )


# Module-level singleton — one container per process
order_container = OrderContainer()
```

### Container Scopes

| Scope | How to Implement | Use When |
|-------|-----------------|---------|
| **Singleton** | `@cached_property` on container | Stateless: repositories, HTTP clients, loggers |
| **Transient** | Plain method (returns new instance) | Use cases, commands |
| **Request-scoped** | Pass container method result to each request; do not cache on container | Anything that holds request state |

**Never cache use cases as singletons.** Use cases may evolve to hold state during a request. Keep them transient.

### Django Ninja Endpoint Injection

The endpoint calls the container method. No DI magic — explicit and readable.

```python
@router.post("/", response={201: OrderResponseAPI})
def create_order(request, payload: OrderRequestAPI):
    use_case = order_container.create_order_use_case()  # transient
    result = use_case.execute(...)
    return 201, OrderResponseAPI.from_result(result)
```

### Testing with DI — Swapping Real for Fake

In tests, do not touch the module-level `order_container`. Create a test container with fake implementations injected directly.

```python
# tests/unit/application/test_create_order.py
import pytest
from decimal import Decimal

from apps.orders.application.use_cases.create_order import CreateOrderUseCase
from apps.orders.application.dtos import CreateOrderCommand, CreateOrderLineCommand
from apps.orders.application.exceptions import CustomerNotFoundError
from apps.orders.domain.value_objects import CustomerId

from tests.fakes.order_repository import FakeOrderRepository
from tests.fakes.user_repository import FakeUserRepository
from tests.fakes.event_publisher import FakeEventPublisher


@pytest.fixture
def fake_order_repo() -> FakeOrderRepository:
    return FakeOrderRepository()


@pytest.fixture
def fake_user_repo() -> FakeUserRepository:
    repo = FakeUserRepository()
    repo.add_existing_user_id(CustomerId(42))
    return repo


@pytest.fixture
def fake_publisher() -> FakeEventPublisher:
    return FakeEventPublisher()


@pytest.fixture
def use_case(
    fake_order_repo: FakeOrderRepository,
    fake_user_repo: FakeUserRepository,
    fake_publisher: FakeEventPublisher,
) -> CreateOrderUseCase:
    return CreateOrderUseCase(
        order_repository=fake_order_repo,
        user_repository=fake_user_repo,
        event_publisher=fake_publisher,
    )


def test_create_order_succeeds(
    use_case: CreateOrderUseCase,
    fake_order_repo: FakeOrderRepository,
    fake_publisher: FakeEventPublisher,
) -> None:
    # Arrange
    command = CreateOrderCommand(
        customer_id=CustomerId(42),
        currency="USD",
        lines=[
            CreateOrderLineCommand(
                product_id=1,
                quantity=2,
                unit_price=Decimal("19.99"),
            )
        ],
    )

    # Act
    result = use_case.execute(command)

    # Assert
    assert result.customer_id == 42
    assert len(fake_order_repo.saved_orders) == 1
    assert len(fake_publisher.published_events) == 1


def test_create_order_raises_when_customer_not_found(
    use_case: CreateOrderUseCase,
) -> None:
    command = CreateOrderCommand(
        customer_id=CustomerId(999),  # does not exist in fake repo
        currency="USD",
        lines=[
            CreateOrderLineCommand(product_id=1, quantity=1, unit_price=Decimal("9.99"))
        ],
    )
    with pytest.raises(CustomerNotFoundError):
        use_case.execute(command)
```

---
