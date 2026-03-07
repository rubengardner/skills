## SPECIALIST A: Layer Definitions

### The Dependency Rule

Dependencies point inward only. Inner layers are unaware of outer layers.

```
┌─────────────────────────────────────────────┐
│  INFRASTRUCTURE                              │
│  (Django ORM, HTTP, S3, Redis, Email)        │
│  ┌───────────────────────────────────────┐   │
│  │  APPLICATION                          │   │
│  │  (Use Cases, Application Services)   │   │
│  │  ┌─────────────────────────────────┐  │   │
│  │  │  DOMAIN                         │  │   │
│  │  │  (Models, VOs, Domain Services, │  │   │
│  │  │   Repository Interfaces)        │  │   │
│  │  └─────────────────────────────────┘  │   │
│  └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Domain Layer — What Belongs Here

| Allowed | Forbidden |
|---------|-----------|
| Pydantic `BaseModel` domain models | Django ORM models |
| Value objects (immutable Pydantic) | Any `django.*` import |
| Domain services (pure functions/classes) | I/O of any kind |
| Repository interfaces (Protocol/ABC) | HTTP calls |
| Domain events (dataclasses or Pydantic) | File system access |
| Domain exceptions | Cache access |
| Business invariant enforcement | Database queries |
| Enum types for domain concepts | Framework utilities |

### Application Layer — What Belongs Here

| Allowed | Forbidden |
|---------|-----------|
| Use-case classes with `execute()` | Django ORM models |
| Application DTOs (Pydantic) | Any `django.*` import |
| Application exceptions | Direct I/O |
| Calls to domain services | HTTP calls |
| Calls through repository interfaces | Infrastructure implementations |
| Transaction boundary declarations | Pydantic v2 validators on infra schemas |
| Orchestration logic | |

### Infrastructure Layer — What Belongs Here

| Allowed | Forbidden |
|---------|-----------|
| Django ORM models (`<Domain>DB`) | Domain logic |
| Repository implementations | Business rules |
| Django Ninja routers and schemas | Application use-case logic |
| HTTP clients (httpx) | Cross-layer circular imports |
| Email, S3, Redis clients | |
| DI container / bootstrap | |
| Unit of Work implementation | |
| Mappers (ORM ↔ domain) | |

### Canonical Directory Structure

```
src/
└── myproject/
    ├── domain/
    │   ├── __init__.py
    │   ├── models.py              # Domain models (Pydantic)
    │   ├── value_objects.py       # Immutable, self-validating Pydantic models
    │   ├── events.py              # Domain events (dataclasses or Pydantic)
    │   ├── services.py            # Pure domain services
    │   ├── exceptions.py          # Domain exception hierarchy
    │   └── repositories.py        # Repository interfaces (Protocol/ABC)
    │
    ├── application/
    │   ├── __init__.py
    │   ├── use_cases/
    │   │   ├── __init__.py
    │   │   ├── create_order.py
    │   │   ├── cancel_order.py
    │   │   └── get_order.py
    │   ├── dtos.py                # Input/output DTOs for use cases
    │   └── exceptions.py          # Application-level exceptions
    │
    └── infrastructure/
        ├── __init__.py
        ├── django/
        │   ├── models.py          # ORM models (<Domain>DB)
        │   ├── repositories.py    # Repository implementations
        │   └── unit_of_work.py    # Django transaction UoW
        ├── http/
        │   └── payment_client.py  # httpx-based HTTP clients
        ├── email/
        │   └── email_service.py   # Django email adapter
        ├── cache/
        │   └── redis_service.py   # Cache adapter
        ├── api/
        │   ├── routers/
        │   │   └── order_router.py  # Django Ninja routers
        │   └── schemas/
        │       └── order_schemas.py # RequestAPI / ResponseAPI schemas
        └── container.py           # DI container — bootstrapped here
```

For a Django project, this maps to one Django app per bounded context:

```
apps/
└── orders/
    ├── domain/
    │   ├── models.py
    │   ├── value_objects.py
    │   ├── events.py
    │   ├── services.py
    │   ├── exceptions.py
    │   └── repositories.py
    ├── application/
    │   ├── use_cases/
    │   │   ├── create_order.py
    │   │   └── get_order.py
    │   ├── dtos.py
    │   └── exceptions.py
    ├── infrastructure/
    │   ├── models.py          # OrderDB
    │   ├── repositories.py    # DjangoOrderRepository
    │   ├── unit_of_work.py
    │   └── container.py
    ├── api/
    │   ├── router.py          # Django Ninja router
    │   └── schemas.py         # RequestAPI / ResponseAPI
    ├── __init__.py            # Exports: OrderClient, OrderId
    └── client.py              # Module boundary client
```

---
