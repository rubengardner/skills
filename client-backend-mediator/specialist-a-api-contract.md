# Specialist A — API Contract Shape

## THE CORE CONFLICT

The backend models the world correctly. The frontend renders a specific screen.
These are different jobs, and the data shape that satisfies one rarely satisfies the other.

---

## MEDIATION

### Scenario: Order detail endpoint

**⚙ BACKEND SAYS**

> I have an `Order` domain model with a normalized `OrderItem` collection and a separate `Customer` aggregate. My repository returns these cleanly. I'll expose `GET /orders/{id}` which returns the `Order`, and `GET /orders/{id}/items` for the line items. That's two clean endpoints. The schema matches the domain model exactly. No duplication.

**📱 CLIENT SAYS**

> The order detail screen renders the order header, line items, customer name, and total — all on one page, first render. Two API calls means a waterfall: I wait for the first to finish, then fire the second. On a 3G connection that's 600ms of spinner. Give me one endpoint that returns everything the screen needs, shaped the way the component tree consumes it. I don't care about your domain model.

**⚡ THE REAL TENSION**

Backend wants normalized resources that mirror domain boundaries. Client wants a single payload shaped for a single screen, not a general-purpose resource.

---

## PATTERNS

### Option 1: Response DTO per endpoint (recommended starting point)

The backend keeps its domain model clean and maps to a screen-specific DTO at the controller/serializer layer.

```python
# Domain model (internal, never returned directly)
@dataclass(frozen=True)
class Order:
    id: OrderId
    customer: Customer
    items: list[OrderItem]
    status: OrderStatus

# Response DTO for the order detail screen
class OrderDetailResponse(BaseModel):
    id: str
    status: str
    customer_name: str          # flattened from Customer aggregate
    customer_email: str
    items: list[OrderItemDTO]
    subtotal: Decimal
    total: Decimal
    created_at: datetime

class OrderItemDTO(BaseModel):
    product_name: str
    quantity: int
    unit_price: Decimal
    line_total: Decimal

# Serializer maps domain → response DTO
def to_order_detail_response(order: Order) -> OrderDetailResponse:
    return OrderDetailResponse(
        id=str(order.id),
        status=order.status.value,
        customer_name=order.customer.full_name,
        customer_email=order.customer.email,
        items=[to_item_dto(item) for item in order.items],
        subtotal=order.subtotal(),
        total=order.total(),
        created_at=order.created_at,
    )
```

**Trade-off:** One DTO per screen. If you have 5 screens showing orders, you have 5 DTOs. Backend has more serializer code. Client gets exactly what it needs.

---

### Option 2: Sparse fieldsets (frontend selects fields)

The backend returns the full object; the client specifies which fields it wants via query param.

```
GET /orders/123?fields=id,status,customer.name,items.product_name,items.quantity,total
```

```python
# Django Ninja example
@router.get("/orders/{order_id}")
def get_order(request, order_id: str, fields: str | None = None):
    order = order_service.get(order_id)
    response = to_full_order_response(order)
    if fields:
        return filter_fields(response.model_dump(), fields.split(","))
    return response
```

**Trade-off:** Flexible — client controls payload size. Backend needs a field-filtering utility. Complex nested field paths are awkward. Good for reducing bandwidth; not great for strong typing on the client.

---

### Option 3: Composite endpoint for high-traffic screens

Keep normalized endpoints. Add a dedicated composite endpoint only for screens where latency is measurable and painful.

```
GET /orders/{id}              → order header only (fast, cacheable)
GET /orders/{id}/items        → line items
GET /screens/order-detail/{id} → full composite (all in one, for the detail screen)
```

**Trade-off:** Explicit purpose. The composite endpoint is clearly labelled as screen-specific. Maintains clean resource endpoints. Backend has more endpoints to maintain.

---

## RULES

- **Never return ORM model instances directly.** Always serialize through a DTO. The domain model is not the contract.
- **Field naming follows the client's convention, not the backend's.** If the frontend is TypeScript and uses `camelCase`, the JSON uses `camelCase`. Not `snake_case` because Django uses it.
- **Null vs absent.** Agree on whether missing optional fields are `null` or omitted. Pick one and never mix.
- **Envelope or no envelope.** Agree on whether responses are wrapped: `{"data": {...}}` vs bare object. Either is fine; inconsistency is not.

```python
# Consistent naming: use response_model_alias_generator in Ninja/FastAPI
from ninja import Schema
from ninja.schema import Schema
import humps

class OrderDetailResponse(Schema):
    class Config:
        alias_generator = humps.camelize  # snake_case → camelCase
        populate_by_name = True

    order_id: str          # serialized as "orderId"
    customer_name: str     # serialized as "customerName"
```
