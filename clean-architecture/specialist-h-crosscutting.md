## SPECIALIST H: Cross-Cutting Concerns

### Logging

**Where it lives:** Infrastructure. Loggers are injected into use cases or domain services if needed.

**Rule:** The domain and application layers do not import `logging` or `structlog` directly. If a use case needs to log, it receives a logger through dependency injection.

```python
# In the application layer — accept a logger Protocol
from typing import Protocol

class Logger(Protocol):
    def info(self, msg: str, **kwargs: object) -> None: ...
    def warning(self, msg: str, **kwargs: object) -> None: ...
    def error(self, msg: str, **kwargs: object) -> None: ...


# In the DI container (infrastructure) — inject structlog
import structlog

class OrderContainer:
    @cached_property
    def logger(self) -> structlog.BoundLogger:
        return structlog.get_logger("orders")

    def create_order_use_case(self) -> CreateOrderUseCase:
        return CreateOrderUseCase(
            order_repository=self.order_repository,
            user_repository=self.user_repository,
            event_publisher=self.event_publisher,
            logger=self.logger,
        )
```

For most use cases, logging at the infrastructure boundary (the router) is sufficient. Log before calling the use case and on exception. Do not log inside use cases unless the domain logic itself warrants it.

```python
# apps/orders/api/router.py
import structlog

logger = structlog.get_logger(__name__)


@router.post("/", response={201: OrderResponseAPI})
def create_order(request, payload: OrderRequestAPI):
    log = logger.bind(customer_id=request.auth.pk)
    use_case = order_container.create_order_use_case()
    try:
        result = use_case.execute(...)
        log.info("order_created", order_id=result.id)
        return 201, OrderResponseAPI.from_result(result)
    except CustomerNotFoundError as exc:
        log.warning("customer_not_found", customer_id=exc.customer_id)
        raise HttpError(404, str(exc)) from exc
    except DomainError as exc:
        log.error("domain_error", error=str(exc))
        raise HttpError(400, str(exc)) from exc
```

### Error Mapping — Domain to HTTP

Error translation is the responsibility of the infrastructure boundary (the router). Domain and application exceptions never escape as raw HTTP errors.

```python
# The complete mapping pattern in a router

from apps.orders.domain.exceptions import (
    DomainError,
    OrderCannotBeCancelledError,
    InsufficientInventoryError,
)
from apps.orders.application.exceptions import (
    ApplicationError,
    CustomerNotFoundError,
    OrderAccessDeniedError,
)


# Per-endpoint exception handling
@router.post("/", response={201: OrderResponseAPI, 409: ErrorSchema})
def create_order(request, payload: OrderRequestAPI):
    use_case = order_container.create_order_use_case()
    try:
        result = use_case.execute(...)
    except CustomerNotFoundError as exc:
        raise HttpError(404, str(exc)) from exc
    except InsufficientInventoryError as exc:
        raise HttpError(409, f"Insufficient stock for product {exc.product_id}") from exc
    except DomainError as exc:
        raise HttpError(400, str(exc)) from exc
    return 201, OrderResponseAPI.from_result(result)
```

For project-wide exception mapping, register handlers on the `NinjaAPI` instance:

```python
# api.py
from apps.orders.domain.exceptions import DomainError
from apps.orders.application.exceptions import ApplicationError

@api.exception_handler(DomainError)
def handle_domain_error(request, exc: DomainError):
    return api.create_response(request, {"detail": str(exc)}, status=400)

@api.exception_handler(ApplicationError)
def handle_application_error(request, exc: ApplicationError):
    return api.create_response(request, {"detail": str(exc)}, status=422)
```

**Do not expose exception class names or stack traces in HTTP responses.** Map to human-readable messages.

### Validation Placement

| Validation type | Where it lives |
|----------------|---------------|
| HTTP input format (field types, required vs optional, string length) | Infrastructure — `<Domain>RequestAPI` (Pydantic) |
| Business invariants (order must have lines, money must be positive) | Domain — Pydantic validators on domain models |
| Cross-aggregate consistency (customer must exist) | Application — use case `execute()` method |
| Database constraints (unique, FK integrity) | Infrastructure — ORM model `Meta.constraints` |

```python
# WRONG — business invariant in infrastructure schema
class OrderRequestAPI(Schema):
    lines: list[OrderLineRequestAPI]

    @field_validator("lines")
    @classmethod
    def lines_must_not_be_empty(cls, v: list) -> list:
        if not v:
            raise ValueError("Order must have at least one item")
        return v
    # ^ This is a domain rule. Move it to Order domain model.


# CORRECT — infrastructure schema handles HTTP input only
class OrderRequestAPI(Schema):
    currency: str = Field(min_length=3, max_length=3, pattern="^[A-Z]{3}$")
    lines: list[OrderLineRequestAPI] = Field(min_length=1)  # HTTP input constraint

# CORRECT — domain model enforces business invariant
class Order(BaseModel):
    lines: list[OrderLine]

    @model_validator(mode="after")
    def confirmed_order_must_have_lines(self) -> Self:
        if self.status == OrderStatus.CONFIRMED and not self.lines:
            raise ValueError("A confirmed order must have at least one line")
        return self
```

---
