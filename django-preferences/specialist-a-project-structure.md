## SPECIALIST A: Project Structure

### Canonical Layout

```
myproject/
├── config/
│   ├── asgi.py                  # ASGI entry — use for production
│   ├── wsgi.py
│   ├── urls.py                  # Only mounts api.urls + admin
│   └── settings/
│       ├── __init__.py
│       ├── base.py              # All non-environment settings
│       ├── local.py             # DEBUG=True, dev tools
│       └── production.py        # HTTPS, secure cookies, strict hosts
├── api.py                       # Single NinjaAPI() instance — ONLY here
├── apps/
│   ├── users/
│   │   ├── __init__.py          # Exports: UserClient, UserId (public surface)
│   │   ├── models.py            # UserDB — Django ORM models only
│   │   ├── schemas.py           # UserRequestAPI, UserResponseAPI, User (domain)
│   │   ├── router.py            # Router() — endpoints only, calls services
│   │   ├── services.py          # Business logic — writes, mutations
│   │   ├── selectors.py         # Read-only QuerySet functions
│   │   ├── client.py            # UserClient — the ONLY cross-module interface
│   │   ├── admin.py
│   │   └── tests/
│   │       ├── conftest.py
│   │       ├── test_router.py
│   │       ├── test_client.py   # Test the public client interface
│   │       └── factories.py
│   └── orders/
│       └── ...  (same structure)
└── common/
    ├── models.py                # TimestampedModel, SoftDeleteModel, UUIDModel
    ├── schemas.py               # Shared: ErrorSchema, PageSchema
    ├── auth.py                  # Project-wide auth backends
    └── pagination.py            # Custom pagination classes
```

### Rules

1. **One `NinjaAPI()` instance** — always in `api.py` at project root. Never inside an app.
2. **Schemas in `schemas.py`** — never in `models.py`. Prevents circular imports.
3. **Business logic in `services.py`** — routers call services; services own the ORM writes. A router that contains complex ORM mutations is wrong.
4. **Read-only QuerySets in `selectors.py`** — keeps routers thin and QuerySets testable in isolation.
5. **Name app router files `router.py`** — not `api.py` (that name belongs to the root).

### `config/urls.py`

```python
from django.contrib import admin
from django.urls import path
from api import api

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", api.urls),
]
```

### `INSTALLED_APPS` Order in `base.py`

```python
INSTALLED_APPS = [
    # Django built-ins
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # Third-party
    "ninja",
    # Local apps
    "apps.users",
    "apps.orders",
    "apps.products",
    "common",
]
```

---
