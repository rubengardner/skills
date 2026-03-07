## SPECIALIST E: Endpoints

### Parameter Location — Inferred by Type

```python
from ninja import Router, Query, Body
from typing import Annotated

router = Router()

# Path parameter — part of URL pattern
@router.get("/{order_id}")
def get_order(request, order_id: int): ...

# Query parameters — typed args not in path pattern
@router.get("/")
def list_orders(request, status: str | None = None, page: int = 1): ...

# Body — Schema subclass is automatically request body
@router.post("/")
def create_order(request, payload: OrderCreateIn): ...

# Annotated form with validation
@router.get("/search")
def search(request, q: Annotated[str, Query(min_length=2, max_length=100)]): ...
```

### Response Schemas — Always Explicit

```python
from ninja.responses import codes_4xx

@router.post(
    "/",
    response={201: OrderOut, 400: ErrorSchema, codes_4xx: ErrorSchema},
    summary="Create an order",
)
def create_order(request, payload: OrderCreateIn):
    if not payload.items:
        return 400, ErrorSchema(detail="Order must have at least one item")
    order = OrderService.create(user=request.auth, payload=payload)
    return 201, order
```

**Rules:**
- Always define `response=` explicitly. Never rely on implicit dict returns.
- Return `(status_code, data)` tuple for non-200 codes.
- Document all error codes in the `response=` dict for accurate OpenAPI output.

### Pagination

```python
from ninja.pagination import paginate, LimitOffsetPagination

@router.get("/", response=list[OrderOut])
@paginate(LimitOffsetPagination)
def list_orders(request, **kwargs):
    return get_order_list(user=request.auth)  # return QS, not a list
```

`@paginate` wraps the response in `{"items": [...], "count": N}`. The raw `response=` type stays `list[OrderOut]` — Django Ninja rewrites it.

### Custom Pagination

```python
# common/pagination.py
from ninja.pagination import PaginationBase
from ninja import Schema

class PagePagination(PaginationBase):
    class Input(Schema):
        page: int = 1
        page_size: int = Field(20, ge=1, le=100)

    class Output(Schema):
        results: list
        total: int
        page: int
        pages: int

    items_attribute = "results"

    def paginate_queryset(self, queryset, pagination, **params):
        offset = (pagination.page - 1) * pagination.page_size
        total = queryset.count()
        return {
            "results": list(queryset[offset: offset + pagination.page_size]),
            "total": total,
            "page": pagination.page,
            "pages": -(-total // pagination.page_size),  # ceiling division
        }
```

### File Uploads

```python
from ninja import File, Form
from ninja.files import UploadedFile

@router.post("/avatar")
def upload_avatar(request, file: File[UploadedFile]) -> dict:
    if file.content_type not in ("image/jpeg", "image/png", "image/webp"):
        raise HttpError(400, "Only JPEG, PNG, WebP accepted")
    request.auth.profile.avatar.save(file.name, file)
    return {"filename": file.name}

# File + form data together
@router.put("/profile")
def update_profile(
    request,
    data: Form[ProfileUpdateIn],
    avatar: File[UploadedFile] | None = None,
):
    ...
```

**Gotcha for PUT/PATCH:** Django does not populate `request.FILES` for PUT/PATCH by default. Add to `MIDDLEWARE`:
```python
"ninja.compatibility.files.fix_request_files_middleware"
```

---
