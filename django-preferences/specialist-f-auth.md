## SPECIALIST F: Auth & Permissions

### HttpBearer — Standard Token Auth

```python
# common/auth.py
from ninja.security import HttpBearer
from django.contrib.auth.models import User

class BearerAuth(HttpBearer):
    def authenticate(self, request, token: str) -> User | None:
        try:
            return User.objects.select_related("profile").get(
                auth_token__key=token,
                is_active=True,
            )
        except User.DoesNotExist:
            return None  # → 401
```

Return value rules:
- Return a truthy object (typically the `User`) → auth succeeds; stored as `request.auth`
- Return `None` → auth fails → 401 automatically

### Async Auth

```python
class AsyncBearerAuth(HttpBearer):
    async def authenticate(self, request, token: str) -> User | None:
        try:
            return await User.objects.aget(auth_token__key=token, is_active=True)
        except User.DoesNotExist:
            return None
```

When using async auth, the endpoint must also be `async def`.

### Auth Scope — Global, Router, Endpoint

```python
# Global default on NinjaAPI
api = NinjaAPI(auth=BearerAuth())

# Override per router (more restrictive)
api.add_router("/admin/", admin_router, auth=AdminBearerAuth())

# Override per endpoint — opt out of global auth
@router.get("/health", auth=None)
def health_check(request): ...

# Multiple accepted auth methods — tried in order
@router.get("/flexible", auth=[BearerAuth(), APIKeyAuth()])
def flexible(request): ...
```

### Permission Checking

Django Ninja has no built-in permission classes. Use a decorator applied **below** the route decorator:

```python
# common/permissions.py
from functools import wraps
from ninja.errors import HttpError
from http import HTTPStatus

def require_staff(func):
    @wraps(func)
    def wrapper(request, *args, **kwargs):
        if not (request.auth and request.auth.is_staff):
            raise HttpError(HTTPStatus.FORBIDDEN, "Staff access required")
        return func(request, *args, **kwargs)
    return wrapper

def require_owner(get_object):
    """Check that request.auth owns the object identified by kwargs."""
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            obj = get_object(**kwargs)
            if obj.owner_id != request.auth.pk:
                raise HttpError(HTTPStatus.FORBIDDEN, "Not the owner")
            return func(request, *args, **kwargs)
        return decorator
    return decorator
```

```python
# CORRECT order — @require_staff BELOW the route decorator
@router.delete("/{order_id}")
@require_staff
def delete_order(request, order_id: int):
    ...
```

**Gotcha:** Permission decorators must be placed **below** `@router.get/post/...`. The route decorator wraps last (Python decorates bottom-up), so an outer permission wrapper runs before `request.auth` is populated if placed above.

---
