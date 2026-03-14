# Specialist C — Performance & Data Fetching

## THE CORE CONFLICT

The backend wants efficient, general-purpose queries. The client wants all the data for a screen in one round trip, shaped exactly right.

---

## N+1: THE CLASSIC STANDOFF

**⚙ BACKEND SAYS**

> My `GET /orders` endpoint returns a list of orders. If you want order items, call `GET /orders/{id}/items`. I have clean, RESTful, cacheable endpoints. The query is fast. Don't ask me to JOIN everything together — that's a different query for every possible screen.

**📱 CLIENT SAYS**

> The order list screen shows order ID, date, status, and the first 2 items with thumbnails. To render 20 orders I call `GET /orders` then 20 calls to `GET /orders/{id}/items`. That's 21 requests. On a slow network that's seconds of loading. This is the N+1 problem. Give me the items in the list response.

**⚡ THE REAL TENSION**

Clean resource separation causes N+1 waterfalls. Embedding everything causes over-fetching on screens that don't need it.

---

## PATTERNS

### Option 1: Include parameter (explicit embedding)

```
GET /orders?include=items,customer
GET /orders?include=items.product
```

```python
@router.get("/orders")
def list_orders(request, include: str = ""):
    includes = set(include.split(",")) if include else set()
    orders = order_repo.list_for_user(request.user.id)

    return [
        serialize_order(order, include_items="items" in includes)
        for order in orders
    ]

def serialize_order(order: Order, include_items: bool = False) -> dict:
    data = {
        "id": str(order.id),
        "status": order.status.value,
        "total": str(order.total),
        "created_at": order.created_at.isoformat(),
    }
    if include_items:
        data["items"] = [serialize_item(item) for item in order.items]
    return data
```

**Trade-off:** Client opts in. Backend loads items only when requested. Good balance. Client must know to add `?include=items`. Can get complex with deep nesting.

---

### Option 2: Dedicated list DTO with summary fields

The list endpoint returns a compact summary; the detail endpoint returns the full object.

```python
# List response: compact, no items
class OrderSummaryDTO(BaseModel):
    id: str
    status: str
    total: Decimal
    item_count: int           # just the count, not the items
    first_item_name: str      # denormalized for display in list
    created_at: datetime

# Detail response: full object
class OrderDetailDTO(BaseModel):
    id: str
    status: str
    total: Decimal
    items: list[OrderItemDTO]
    customer: CustomerDTO
    created_at: datetime
```

**Trade-off:** Clean separation of list vs detail. No query param magic. The list view has exactly what it needs. If the list view evolves to need more fields, you change the DTO.

---

### Option 3: Pagination contract

Never return unbounded lists. Agree on a cursor-based pagination contract:

```json
{
  "items": [...],
  "pagination": {
    "cursor": "eyJpZCI6IjEyMyJ9",
    "has_next": true,
    "count": 20
  }
}
```

```python
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    pagination: PaginationMeta

class PaginationMeta(BaseModel):
    cursor: str | None
    has_next: bool
    count: int

@router.get("/orders", response=PaginatedResponse[OrderSummaryDTO])
def list_orders(request, cursor: str | None = None, limit: int = 20):
    orders, next_cursor = order_repo.list_paginated(
        user_id=request.user.id,
        cursor=cursor,
        limit=min(limit, 100),  # enforce max
    )
    return PaginatedResponse(
        items=[to_summary_dto(o) for o in orders],
        pagination=PaginationMeta(
            cursor=next_cursor,
            has_next=next_cursor is not None,
            count=len(orders),
        )
    )
```

**Rules:**
- Cursor pagination over offset pagination. Offset breaks on inserts. Cursor is stable.
- Always enforce a max `limit`. Never allow `limit=0` or `limit=99999`.
- Always return `has_next` so the client knows when to stop fetching.

---

## OVER-FETCHING vs UNDER-FETCHING

| Problem | Symptom | Fix |
|---------|---------|-----|
| Over-fetching | Response has 40 fields, screen uses 5 | Sparse fieldsets or dedicated DTO |
| Under-fetching | Screen requires 3 API calls to render | `include` param or composite endpoint |
| N+1 | 1 list call + N detail calls per item | Eager load in the query, embed in list DTO |

### Detecting over-fetching in code review

If the frontend developer has written:
```typescript
const name = order.customer.address.city  // only one field used from a 20-field response
```
...and the API returns the full customer object, that's over-fetching. Fix it in the DTO.

---

## QUERY OPTIMIZATION RULES (backend)

```python
# WRONG: N+1 — 1 query for orders + N queries for items
orders = Order.objects.filter(user=user)
for order in orders:
    items = order.items.all()  # new query per order

# RIGHT: 2 queries total (or 1 with prefetch_related)
orders = Order.objects.filter(user=user).prefetch_related("items__product")
```

**Rule:** If you expose an endpoint that returns a list, you must ensure the backing query does not produce N+1. Use `select_related` for FK, `prefetch_related` for reverse FK / M2M. Verify with `django-debug-toolbar` or `LOGGING` query count in tests.
