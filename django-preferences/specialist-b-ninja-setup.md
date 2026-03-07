## SPECIALIST B: Django Ninja Setup

### NinjaAPI Configuration

```python
# api.py
from ninja import NinjaAPI
from common.auth import BearerAuth
from common.schemas import ErrorSchema

api = NinjaAPI(
    title="Acme API",
    version="1.0.0",
    description="Acme Corp public API",
    auth=BearerAuth(),           # default auth on all endpoints
    docs_url="/docs",            # disable in production: docs_url=None
    openapi_url="/openapi.json",
    csrf=False,                  # True only if using session auth with browser
)
```

### Router Registration — Use String Imports

```python
# api.py — string import avoids circular imports in large codebases
api.add_router("/users/", "apps.users.router.router")
api.add_router("/orders/", "apps.orders.router.router")
api.add_router("/products/", "apps.products.router.router")
```

```python
# apps/orders/router.py
from ninja import Router
router = Router(tags=["Orders"])

@router.get("/", response=list[OrderOut])
def list_orders(request): ...
```

### Versioning

Use separate `NinjaAPI` instances, each mounted at a distinct URL prefix. Each gets its own OpenAPI docs.

```python
# api_v1.py
api_v1 = NinjaAPI(version="1.0.0", urls_namespace="api_v1")
api_v1.add_router("/users/", "apps.users.router_v1.router")

# api_v2.py
api_v2 = NinjaAPI(version="2.0.0", urls_namespace="api_v2")
api_v2.add_router("/users/", "apps.users.router_v2.router")

# config/urls.py
urlpatterns = [
    path("api/v1/", api_v1.urls),
    path("api/v2/", api_v2.urls),
]
```

**Gotcha:** Multiple `NinjaAPI` instances require distinct `version` OR `urls_namespace`. Without one, Django raises a URL namespace conflict.

### Global Exception Handlers

```python
# api.py
from django.http import Http404
from ninja.errors import ValidationError

@api.exception_handler(Http404)
def not_found(request, exc):
    return api.create_response(request, {"detail": "Not found"}, status=404)

@api.exception_handler(Exception)
def unhandled(request, exc):
    import logging
    logging.getLogger(__name__).exception("Unhandled exception", exc_info=exc)
    return api.create_response(request, {"detail": "Internal server error"}, status=500)
```

### Flatten Pydantic v2 Validation Errors

Override the default 422 format by subclassing `NinjaAPI`:

```python
class AppNinjaAPI(NinjaAPI):
    def on_validation_error(self, request, exc: ValidationError):
        errors = [
            {"field": ".".join(str(loc) for loc in e["loc"]), "msg": e["msg"]}
            for e in exc.errors
        ]
        return self.create_response(request, {"errors": errors}, status=422)

api = AppNinjaAPI(title="Acme API")
```

---
