## SPECIALIST C: Pydantic Schemas

> **Naming is governed by the `api-conventions` skill.** Read it for the full naming rules (RequestAPIModel/ResponseAPIModel with verb prefixes, mapper pattern, file structure). This specialist covers Django Ninja-specific schema patterns only.

### Naming — Quick Reference

| Role | Pattern | Example |
|------|---------|---------|
| API request body | `[Verb]<Domain>RequestAPIModel` | `CreateOrderRequestAPIModel`, `UpdateOrderRequestAPIModel` |
| API response | `[PastTense]<Domain>ResponseAPIModel` | `CreatedOrderResponseAPIModel`, `OrderResponseAPIModel` |
| Domain model (internal Pydantic) | `<Domain>` — flat | `Order`, `User`, `Record` |
| DB model (Django ORM) | `<Domain>DB` | `OrderDB`, `UserDB` |

**No `In` / `Out` / `Schema` suffixes. Names are role-explicit. `APIModel` suffix is mandatory.**

### Three Distinct Schema Layers

```python
# apps/orders/schemas.py
from ninja import Schema, ModelSchema
from pydantic import Field, field_validator
from apps.orders.models import OrderDB, OrderStatus
from decimal import Decimal


# ── 1. DOMAIN MODEL ─────────────────────────────────────────────────────────
# Flat Pydantic model — the canonical representation of the domain concept.
# Used internally: services return this, clients expose this.

class Order(Schema):
    id: int
    status: str
    total: Decimal
    customer_id: int
    created_at: datetime


# ── 2. API INPUT ─────────────────────────────────────────────────────────────
# What the HTTP client sends. Always plain Schema with explicit validation.
# Never reused for output.

class CreateOrderRequestAPIModel(Schema):
    customer_id: int = Field(gt=0)
    items: list[CreateOrderItemRequestAPIModel]

    @field_validator("items")
    @classmethod
    def items_not_empty(cls, v: list) -> list:
        if not v:
            raise ValueError("Order must have at least one item")
        return v


class UpdateOrderRequestAPIModel(Schema):
    """All fields optional — PATCH semantics."""
    status: OrderStatus | None = None
    notes: str | None = None


# ── 3. API OUTPUT ────────────────────────────────────────────────────────────
# What the API returns. Always ModelSchema with explicit fields.
# Never expose internal columns.

class OrderResponseAPIModel(ModelSchema):
    class Meta:
        model = OrderDB
        fields = ["id", "status", "total", "created_at", "updated_at"]
        # Never: fields = "__all__"


class OrdersResponseAPIModel(ModelSchema):
    """Lighter schema for list endpoints — omit heavy/nested fields."""
    class Meta:
        model = OrderDB
        fields = ["id", "status", "total", "created_at"]
```

### ModelSchema Rules

- `ModelSchema` → **output (`ResponseAPIModel`) only**. Input (`RequestAPIModel`) schemas are always plain `Schema`.
- **Never use `fields = "__all__"`** — exposes columns you have not reviewed.
- Annotated fields from `annotate()` do NOT appear in `ModelSchema` automatically — declare them on a plain `Schema`.

### PATCH with `exclude_unset`

```python
@router.patch("/{order_id}", response=UpdatedOrderResponseAPIModel)
def patch_order(request, order_id: int, payload: UpdateOrderRequestAPIModel):
    order = get_object_or_404(OrderDB, id=order_id, customer=request.auth)
    for attr, value in payload.model_dump(exclude_unset=True).items():
        setattr(order, attr, value)
    order.save()
    return order
```

**`exclude_unset=True` is mandatory for PATCH.** Without it, unset fields default to `None` and silently null out columns.

### Nested Response Schemas

```python
class OrderItemResponseAPIModel(ModelSchema):
    class Meta:
        model = OrderItemDB
        fields = ["id", "product_id", "quantity", "unit_price"]

class OrderDetailResponseAPIModel(ModelSchema):
    class Meta:
        model = OrderDB
        fields = ["id", "status", "total", "created_at"]

    items: list[OrderItemResponseAPIModel] = []
```

**Rule:** Any endpoint returning nested `ResponseAPIModel` schemas **must** have `prefetch_related` in its selector. Nested schemas without prefetch trigger N+1 queries during serialization. See Specialist D.

### Computed Fields with Resolvers

```python
class OrderResponseAPIModel(ModelSchema):
    class Meta:
        model = OrderDB
        fields = ["id", "status", "total"]

    is_editable: bool = False

    @staticmethod
    def resolve_is_editable(obj: OrderDB) -> bool:
        return obj.status in (OrderStatus.DRAFT, OrderStatus.PENDING)
```

### Schema Inheritance

```python
class BaseResponseAPIModel(Schema):
    id: int
    created_at: datetime

class UserResponseAPIModel(BaseResponseAPIModel):
    username: str
    email: str

class AdminUserResponseAPIModel(UserResponseAPIModel):
    is_staff: bool
    last_login: datetime | None
```

Keep inheritance shallow (≤3 levels).

### CamelCase APIs

```python
from pydantic import ConfigDict
from ninja import Schema
from ninja.utils import to_camel

class CamelSchema(Schema):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )

@api.get("/users/{id}", response=CamelSchema, by_alias=True)
def get_user(request, id: int): ...
```

---
