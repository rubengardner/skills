## SPECIALIST I: Testing

### Setup

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.local"
addopts = "--reuse-db --no-migrations -ra --strict-markers"

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-django",
    "pytest-asyncio",
    "pytest-mock",
    "factory-boy",
    "faker",
]
```

### conftest.py — Root

```python
# conftest.py
import pytest
from ninja.testing import TestClient, TestAsyncClient
from api import api

@pytest.fixture(scope="session")
def ninja_api():
    return api

@pytest.fixture
def client():
    return TestClient(api)

@pytest.fixture
def async_client():
    return TestAsyncClient(api)
```

### Factories

```python
# apps/users/tests/factories.py
import factory
from factory.django import DjangoModelFactory
from faker import Faker
from django.contrib.auth.models import User
from apps.orders.models import Order, OrderStatus

fake = Faker()

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
        skip_postgeneration_save = True  # required in factory_boy 3.3+

    username = factory.LazyFunction(fake.user_name)
    email = factory.LazyAttribute(lambda obj: f"{obj.username}@example.com")
    password = factory.PostGenerationMethodCall("set_password", "testpassword123")
    is_active = True


class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    customer = factory.SubFactory(UserFactory)
    status = OrderStatus.DRAFT
    total = factory.LazyFunction(lambda: Decimal("99.99"))
```

### App conftest.py

```python
# apps/orders/tests/conftest.py
import pytest
from .factories import UserFactory, OrderFactory

@pytest.fixture
def user(db):
    return UserFactory()

@pytest.fixture
def order(db, user):
    return OrderFactory(customer=user)
```

### Testing Endpoints

```python
# apps/orders/tests/test_router.py
import pytest
from ninja.testing import TestClient
from apps.orders.router import router

client = TestClient(router)

@pytest.mark.django_db
class TestOrderEndpoints:

    def test_list_unauthenticated(self):
        response = client.get("/")
        assert response.status_code == 401

    def test_list(self, user, order):
        response = client.get("/", user=user)  # user= bypasses auth entirely
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 1

    def test_create_valid(self, user):
        payload = {"customer_id": user.pk, "items": [{"product_id": 1, "quantity": 2}]}
        response = client.post("/", json=payload, user=user)
        assert response.status_code == 201
        assert response.json()["status"] == "draft"

    def test_create_invalid_empty_items(self, user):
        response = client.post("/", json={"customer_id": user.pk, "items": []}, user=user)
        assert response.status_code == 422

    def test_patch_excludes_unset(self, user, order):
        response = client.patch(f"/{order.pk}", json={"status": "confirmed"}, user=user)
        assert response.status_code == 200
        order.refresh_from_db()
        assert order.status == "confirmed"
        assert order.notes == ""  # was not overwritten with None
```

### `user=` Shortcut vs Mocked Auth

```python
# PREFER: user= shortcut — sets request.auth = user, skips auth callable
response = client.get("/protected", user=user)

# USE when testing auth logic itself:
def test_invalid_token(mocker):
    mocker.patch("common.auth.BearerAuth.authenticate", return_value=None)
    response = client.get("/protected", headers={"Authorization": "Bearer bad"})
    assert response.status_code == 401
```

### Async Endpoint Tests

```python
@pytest.mark.django_db(transaction=True)   # required for async tests
@pytest.mark.asyncio
async def test_async_get_user(async_client, user):
    response = await async_client.get(f"/users/{user.pk}", user=user)
    assert response.status_code == 200
```

**Gotcha:** Async tests require `@pytest.mark.django_db(transaction=True)`. Without it, some async ORM operations hang or leak state.

---
