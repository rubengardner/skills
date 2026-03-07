## SPECIALIST J: Performance & Production

### ASGI vs WSGI

| Scenario | Choice |
|----------|--------|
| All sync endpoints | WSGI + Gunicorn |
| Mix of sync + async | **ASGI + Uvicorn** (recommended) |
| Heavy async I/O, external APIs | ASGI + Uvicorn |
| WebSockets or SSE | ASGI required |

```bash
# Production
gunicorn config.asgi:application -w 4 -k uvicorn.workers.UvicornWorker
```

```python
# config/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")
application = get_asgi_application()
```

### Connection Pooling

**Django 5.1+ native pooling (psycopg3 driver required):**
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "CONN_MAX_AGE": 0,    # REQUIRED — must be 0 with native pooling
        "OPTIONS": {
            "pool": {
                "min_size": 4,
                "max_size": 16,
                "timeout": 10,
            }
        },
    }
}
```

**PgBouncer (Django < 5.1 or high concurrency):**
- Point `DATABASE_URL` at PgBouncer's port
- Use transaction pooling mode
- Set `CONN_MAX_AGE = 0`

### Database Indexes

Every field used in `filter()`, `exclude()`, `order_by()`, or a JOIN condition must have an index.

```python
class Meta:
    indexes = [
        models.Index(fields=["status", "created_at"], name="orders_status_created"),
        # Partial index (PostgreSQL) — high selectivity
        models.Index(
            fields=["customer"],
            condition=models.Q(status="open"),
            name="orders_open_by_customer",
        ),
    ]
```

**Rules:**
- ForeignKey fields get an index automatically — do not add `db_index=True` manually.
- Add `db_index=True` to `CharField` / `IntegerField` columns used in `filter()`.
- Composite indexes for patterns that combine two or more fields.
- Use `EXPLAIN ANALYZE` in PostgreSQL to verify index usage.

### Caching

```python
# Cross-request caching with Redis
from django.core.cache import cache

def get_active_products() -> list:
    key = "products:active"
    data = cache.get(key)
    if data is None:
        data = list(Product.objects.filter(is_active=True).values("id", "name", "price"))
        cache.set(key, data, timeout=300)
    return data
```

```python
# config/settings/production.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": _settings.REDIS_URL,
    }
}
```

### Production Settings Checklist

```python
# config/settings/production.py
DEBUG = False
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
STATIC_ROOT = BASE_DIR / "staticfiles"
STORAGES = {
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"
    }
}
```

---
