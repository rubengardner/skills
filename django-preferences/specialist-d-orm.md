## SPECIALIST D: ORM Patterns

### `select_related` vs `prefetch_related`

| Scenario | Use |
|----------|-----|
| FK or OneToOne (forward) | `select_related("author")` |
| FK or OneToOne (reverse) | `prefetch_related("book_set")` |
| ManyToMany | `prefetch_related("tags")` |
| Filtered/annotated prefetch | `Prefetch("orders", queryset=...)` |

### Selectors — Always Define QuerySets Here

```python
# apps/orders/selectors.py
from django.db.models import Prefetch, QuerySet
from apps.orders.models import OrderDB, OrderItemDB

def get_order_list(*, customer_id: int) -> QuerySet[OrderDB]:
    return (
        OrderDB.objects.filter(customer_id=customer_id)
        .prefetch_related(
            Prefetch("items", queryset=OrderItemDB.objects.select_related("product"))
        )
        .order_by("-created_at")
    )

def get_order_detail(*, order_id: int, customer_id: int) -> OrderDB:
    return get_object_or_404(
        OrderDB.objects.prefetch_related("items__product"),
        pk=order_id,
        customer_id=customer_id,
    )
```

**Rule:** Routers call selectors. Selectors return QuerySets or model instances. Routers never build QuerySets directly.

### N+1 Prevention

The most common Django Ninja N+1: a `ModelSchema` with nested schemas but no prefetch.

```python
# WRONG — bare queryset, N+1 on items
@router.get("/", response=list[OrderDetailOut])
def list_orders(request):
    return Order.objects.all()

# CORRECT — selector handles prefetch
@router.get("/", response=list[OrderDetailOut])
def list_orders(request):
    return get_order_list(user=request.auth)
```

During development, enforce detection:
```python
# config/settings/local.py
INSTALLED_APPS += ["nplusone.ext.django"]
NPLUSONE_RAISE = True  # raises exception on any N+1 detected in tests
```

### `annotate()` with Schemas

```python
# selectors.py
from django.db.models import Count, Sum

def get_customer_stats():
    return Customer.objects.annotate(
        order_count=Count("orders"),
        total_spend=Sum("orders__total"),
    )

# schemas.py — must be plain Schema, not ModelSchema
class CustomerStatsOut(Schema):
    id: int
    name: str
    order_count: int
    total_spend: Decimal
```

`annotate()` adds dynamic attributes to model instances. `ModelSchema` does not pick them up — always use a plain `Schema` for annotated output.

### `values()` — When to Use

- Use `values()` / `values_list()` for export/reporting endpoints where you control the final shape and don't need model methods.
- **Never pass `values()` results to `ModelSchema`** — it receives dicts, not instances.

### Async ORM (Django 4.2+)

```python
@router.get("/{user_id}", response=UserOut)
async def get_user(request, user_id: int):
    try:
        return await User.objects.select_related("profile").aget(pk=user_id)
    except User.DoesNotExist:
        raise HttpError(404, "User not found")

@router.get("/", response=list[UserOut])
async def list_users(request):
    # Must materialize — cannot return unevaluated async QuerySet
    return [u async for u in User.objects.select_related("profile").all()]
```

**Rules:**
- Never return an unevaluated async QuerySet. Materialize with `[x async for x in qs]`.
- Do not call sync ORM methods (`objects.get()`) inside `async def` endpoints without `sync_to_async`.
- `prefetch_related` with `async for` is fully supported in Django 4.2+.

### Bulk Operations

```python
# Create — one SQL INSERT
Order.objects.bulk_create(
    [Order(customer=user, total=t) for t in totals],
    batch_size=500,
)

# Update all matching rows — one SQL UPDATE
Order.objects.filter(status="pending").update(status="processing")

# Update specific instances with per-row values
Order.objects.bulk_update(order_list, fields=["status", "updated_at"])
```

**Rule:** Never loop and call `.save()` for batch creates or updates. `bulk_create` / `bulk_update` / `.update()` are always correct.

---
