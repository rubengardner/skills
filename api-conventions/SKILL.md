---
name: api-conventions
description: Framework-agnostic Python API conventions. Covers file structure (infrastructure/api/vX/api_models/), naming patterns (RequestAPIModel/ResponseAPIModel with verb prefixes), the mapper pattern, and endpoint signature rules. Use when designing API endpoints, naming request/response models, or structuring API files in any Python backend.
---

## IDENTITY

These are the shared API conventions for Python backends — independent of framework.

For framework-specific wiring, see the relevant skill:
- Django Ninja → `django-preferences`
- FastAPI → `fastapi-preferences` (when available)

---

## File Structure

API code lives under `infrastructure/api/vX/` within each module. Each API model gets its own folder containing the model definition and its mapper:

```
<module>/
├── clients/
│   ├── __init__.py        # exports
│   └── client.py
└── infrastructure/
    └── api/
        └── v1/
            ├── api_models/
            │   ├── create_order_request_api_model/
            │   │   ├── api_model.py
            │   │   └── api_mapper.py
            │   ├── order_response_api_model/
            │   │   ├── api_model.py
            │   │   └── api_mapper.py
            │   └── orders_response_api_model/
            │       ├── api_model.py
            │       └── api_mapper.py
            └── api.py
```

Each `api_model.py` contains the Pydantic model. Each `api_mapper.py` maps between the API model and the domain model. The mapper never contains business logic.

---

## Naming Conventions

### Request Models

Pattern: `[Create | Update | Replace]?<DomainName>RequestAPIModel`

- **Omit the verb prefix** for read/search-style requests (no mutation implied).
- **If create and update models are identical**, use `NewType` — do not duplicate:

```python
UpdateCompanyRequestAPIModel = NewType("UpdateCompanyRequestAPIModel", CreateCompanyRequestAPIModel)
```

| Operation | Example |
|-----------|---------|
| Create | `CreateOrderRequestAPIModel` |
| Partial update | `UpdateOrderRequestAPIModel` |
| Full replace | `ReplaceOrderRequestAPIModel` |
| Search / read | `SearchOrdersRequestAPIModel` (no verb prefix) |

### Response Models

Pattern: `[Created | Updated | Deleted]?<DomainName>ResponseAPIModel`

- **Use past-tense prefix** only when the response is specific to a mutating operation.
- **Omit the verb prefix** for generic read responses.

| Case | Example |
|------|---------|
| After create | `CreatedOrderResponseAPIModel` |
| After update | `UpdatedOrderResponseAPIModel` |
| After delete | `DeletedOrderResponseAPIModel` |
| Read single | `OrderResponseAPIModel` |
| Read list | `OrdersResponseAPIModel` |

---

## Endpoint Signature Pattern

```python
@router.verb("/path", ...)
def endpoint(request: CreateDomainRequestAPIModel) -> CreatedDomainResponseAPIModel:
    pass
```

For path parameters (GET by ID), pass the ID directly — no wrapper request model:

```python
@router.get("/orders/{id}", ...)
def get_order(id: OrderId) -> OrderResponseAPIModel:
    pass
```

---

## Examples by HTTP Verb

### GET — fetch by ID

```python
@router.get("/orders/{id}", ...)
def get_order(id: OrderId) -> OrderResponseAPIModel:
    pass
```

### POST — create a resource

```python
@router.post("/orders", ...)
def create_order(request: CreateOrderRequestAPIModel) -> CreatedOrderResponseAPIModel:
    pass
```

### POST — search

```python
@router.post("/orders/search", ...)
def search_orders(request: SearchOrdersRequestAPIModel) -> OrdersResponseAPIModel:
    pass
```

### PUT — replace a resource

```python
@router.put("/orders/{id}", ...)
def replace_order(id: OrderId, request: ReplaceOrderRequestAPIModel) -> OrderResponseAPIModel:
    pass
```

### PATCH — partial update

```python
@router.patch("/orders/{id}", ...)
def update_order(id: OrderId, request: UpdateOrderRequestAPIModel) -> UpdatedOrderResponseAPIModel:
    pass
```

### DELETE

```python
@router.delete("/orders/{id}", ...)
def delete_order(id: OrderId) -> DeletedOrderResponseAPIModel:
    pass
```

---

## Mapper Pattern

Each `api_mapper.py` converts between the API model and domain model. Keep mappers co-located with their model — never scatter them.

```python
# infrastructure/api/v1/api_models/order_response_api_model/api_mapper.py
from apps.orders.domain.models import Order
from .api_model import OrderResponseAPIModel


def to_order_response(order: Order) -> OrderResponseAPIModel:
    return OrderResponseAPIModel(
        id=order.id,
        status=order.status,
        total=order.total.amount,
        created_at=order.created_at,
    )
```

```python
# infrastructure/api/v1/api_models/create_order_request_api_model/api_mapper.py
from apps.orders.application.dtos import CreateOrderCommand
from .api_model import CreateOrderRequestAPIModel


def to_create_order_command(request: CreateOrderRequestAPIModel) -> CreateOrderCommand:
    return CreateOrderCommand(
        customer_id=request.customer_id,
        currency=request.currency,
        lines=[...],
    )
```

**Rules:**
- Mappers are pure functions — no I/O, no side effects.
- `to_<response_model>` for domain → API direction.
- `to_<command_or_domain>` for API → domain direction.
- Never put validation logic in mappers — validation belongs in the Pydantic model.

---

## Rules Summary

| Situation | Rule |
|-----------|------|
| Create and update models are identical | Use `NewType` alias, don't duplicate |
| GET with a path param | Pass `id: DomainId` directly — no request model |
| POST that mutates (create) | Past-tense prefix on response: `Created...` |
| POST that reads (search) | No verb prefix on response |
| New API model | Dedicated folder under `api_models/` with `api_model.py` + `api_mapper.py` |
| Verb prefix on request | Only for mutating operations: Create, Update, Replace |
| List responses | Plural domain name: `OrdersResponseAPIModel` |
