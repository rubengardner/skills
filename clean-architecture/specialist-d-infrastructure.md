## SPECIALIST D: Infrastructure Layer

### Repository Implementation

The infrastructure repository implements the domain `Protocol`. It maps between ORM models (`<Domain>DB`) and domain models.

```python
# apps/orders/infrastructure/repositories.py
from __future__ import annotations

from apps.orders.domain.models import Order, OrderLine, OrderStatus
from apps.orders.domain.value_objects import OrderId, CustomerId, Money, ProductId
from apps.orders.infrastructure.models import OrderDB, OrderLineDB


class DjangoOrderRepository:
    """Implements OrderRepository using Django ORM.

    This class satisfies the OrderRepository Protocol structurally —
    no explicit inheritance required.
    """

    def get_by_id(self, order_id: OrderId) -> Order | None:
        try:
            db_order = (
                OrderDB.objects
                .prefetch_related("lines")
                .get(pk=order_id)
            )
        except OrderDB.DoesNotExist:
            return None
        return self._to_domain(db_order)

    def save(self, order: Order) -> Order:
        """Insert or update the order. Returns the saved domain model with real ID."""
        if order.id == 0:
            # New order — insert
            db_order = OrderDB.objects.create(
                customer_id=order.customer_id,
                status=order.status,
                cancelled_at=order.cancelled_at,
                created_at=order.created_at,
            )
            OrderLineDB.objects.bulk_create([
                OrderLineDB(
                    order=db_order,
                    product_id=line.product_id,
                    quantity=line.quantity,
                    unit_price=line.unit_price.amount,
                    currency=line.unit_price.currency,
                )
                for line in order.lines
            ])
        else:
            # Existing order — update
            OrderDB.objects.filter(pk=order.id).update(
                status=order.status,
                cancelled_at=order.cancelled_at,
            )
            db_order = OrderDB.objects.prefetch_related("lines").get(pk=order.id)

        return self._to_domain(db_order)

    def delete(self, order_id: OrderId) -> None:
        OrderDB.objects.filter(pk=order_id).delete()

    def find_by_customer(
        self,
        customer_id: CustomerId,
        *,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Order]:
        qs = (
            OrderDB.objects
            .filter(customer_id=customer_id)
            .prefetch_related("lines")
            .order_by("-created_at")[offset : offset + limit]
        )
        return [self._to_domain(db_order) for db_order in qs]

    def _to_domain(self, db_order: OrderDB) -> Order:
        """Map ORM instance to pure domain model. No business logic here."""
        return Order(
            id=OrderId(db_order.pk),
            customer_id=CustomerId(db_order.customer_id),
            status=OrderStatus(db_order.status),
            lines=[
                OrderLine(
                    product_id=ProductId(line.product_id),
                    quantity=line.quantity,
                    unit_price=Money(
                        amount=line.unit_price,
                        currency=line.currency,
                    ),
                )
                for line in db_order.lines.all()
            ],
            created_at=db_order.created_at,
            cancelled_at=db_order.cancelled_at,
        )
```

### ORM Models

ORM models live entirely in infrastructure. They are suffixed with `DB`.

```python
# apps/orders/infrastructure/models.py
from __future__ import annotations

import uuid
from django.db import models

from common.models import TimestampedModel


class OrderDB(TimestampedModel):
    """Persistence model for Order aggregate.

    Never imported outside the orders infrastructure layer.
    """

    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        CONFIRMED = "confirmed", "Confirmed"
        SHIPPED = "shipped", "Shipped"
        CANCELLED = "cancelled", "Cancelled"

    customer_id = models.IntegerField(db_index=True)
    status = models.CharField(
        max_length=20,
        choices=Status,
        default=Status.DRAFT,
        db_index=True,
    )
    cancelled_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = "orders_order"
        indexes = [
            models.Index(fields=["customer_id", "status"]),
            models.Index(fields=["status", "created_at"]),
        ]

    def __str__(self) -> str:
        return f"OrderDB {self.pk} ({self.status})"


class OrderLineDB(models.Model):
    order = models.ForeignKey(
        OrderDB,
        on_delete=models.CASCADE,
        related_name="lines",
    )
    product_id = models.IntegerField(db_index=True)
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=12, decimal_places=4)
    currency = models.CharField(max_length=3)

    class Meta:
        db_table = "orders_order_line"
```

### Unit of Work — Django Implementation

```python
# apps/orders/infrastructure/unit_of_work.py
from __future__ import annotations

from django.db import transaction


class DjangoUnitOfWork:
    """Wraps Django atomic() for use-case-level transaction control.

    Satisfies the UnitOfWork Protocol structurally.
    """

    def __init__(self) -> None:
        self._atomic: transaction.Atomic | None = None

    def __enter__(self) -> DjangoUnitOfWork:
        self._atomic = transaction.atomic()
        self._atomic.__enter__()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: object,
    ) -> bool | None:
        if self._atomic is not None:
            return self._atomic.__exit__(exc_type, exc_val, exc_tb)
        return None

    def commit(self) -> None:
        # Django atomic() commits on clean __exit__. For explicit control:
        # set a savepoint or use on_commit hooks here.
        pass

    def rollback(self) -> None:
        if self._atomic is not None:
            transaction.set_rollback(True)
```

### Infrastructure for External Services

External service clients are adapters behind interfaces defined in the domain or application layer.

```python
# Domain interface (application/ports.py or domain/repositories.py)
class PaymentGateway(Protocol):
    def charge(
        self,
        amount: Money,
        card_token: str,
        idempotency_key: str,
    ) -> str: ...  # returns transaction_id


# Infrastructure implementation
# apps/orders/infrastructure/http/stripe_payment_gateway.py
from __future__ import annotations

import httpx
from apps.orders.domain.value_objects import Money


class StripePaymentGateway:
    """Implements PaymentGateway using Stripe HTTP API."""

    def __init__(self, api_key: str, timeout: float = 10.0) -> None:
        self._client = httpx.Client(
            base_url="https://api.stripe.com/v1",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=timeout,
        )

    def charge(
        self,
        amount: Money,
        card_token: str,
        idempotency_key: str,
    ) -> str:
        response = self._client.post(
            "/charges",
            data={
                "amount": int(amount.amount * 100),  # Stripe uses cents
                "currency": amount.currency.lower(),
                "source": card_token,
            },
            headers={"Idempotency-Key": idempotency_key},
        )
        try:
            response.raise_for_status()
        except httpx.HTTPStatusError as exc:
            from apps.orders.domain.exceptions import PaymentFailedError
            raise PaymentFailedError(
                f"Stripe charge failed: {exc.response.status_code}"
            ) from exc
        return response.json()["id"]
```

### Django Ninja Routers — Infrastructure Only

Routers live in infrastructure. They receive HTTP input, map to use-case commands, invoke the use case through the DI container, and map the result to response schemas. They never contain business logic.

```python
# apps/orders/api/router.py
from __future__ import annotations

from ninja import Router
from ninja.errors import HttpError

from apps.orders.api.schemas import (
    OrderRequestAPI,
    OrderResponseAPI,
    CancelOrderRequestAPI,
)
from apps.orders.application.dtos import (
    CreateOrderCommand,
    CreateOrderLineCommand,
    CancelOrderCommand,
)
from apps.orders.application.exceptions import (
    CustomerNotFoundError,
    OrderAccessDeniedError,
)
from apps.orders.domain.exceptions import (
    DomainError,
    OrderCannotBeCancelledError,
)
from apps.orders.infrastructure.container import order_container

router = Router(tags=["Orders"])


@router.post("/", response={201: OrderResponseAPI}, summary="Create an order")
def create_order(request, payload: OrderRequestAPI):
    use_case = order_container.create_order_use_case()
    try:
        result = use_case.execute(
            CreateOrderCommand(
                customer_id=request.auth.pk,
                currency=payload.currency,
                lines=[
                    CreateOrderLineCommand(
                        product_id=line.product_id,
                        quantity=line.quantity,
                        unit_price=line.unit_price,
                    )
                    for line in payload.lines
                ],
            )
        )
    except CustomerNotFoundError as exc:
        raise HttpError(404, str(exc)) from exc
    return 201, OrderResponseAPI.from_result(result)


@router.delete("/{order_id}", response={204: None}, summary="Cancel an order")
def cancel_order(request, order_id: int, payload: CancelOrderRequestAPI):
    use_case = order_container.cancel_order_use_case()
    try:
        use_case.execute(
            CancelOrderCommand(order_id=order_id, reason=payload.reason)
        )
    except OrderNotFoundError as exc:  # type: ignore[name-defined]
        raise HttpError(404, str(exc)) from exc
    except OrderCannotBeCancelledError as exc:
        raise HttpError(409, str(exc)) from exc
    return 204, None
```

---
