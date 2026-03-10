# Specialist G: Testing Hard Things

## Scope
Time and clocks, randomness, async/concurrent code, external APIs, databases, message queues, file systems, and legacy code.

---

## 1. Time and Clocks

**The problem:** Tests that depend on `Date.now()`, `datetime.now()`, or `time.time()` are non-deterministic. They fail or produce different results based on when they run.

**The solution:** Never call the system clock directly. Inject a clock abstraction.

```python
# Python — Clock abstraction
from datetime import datetime
from typing import Protocol

class Clock(Protocol):
    def now(self) -> datetime: ...

class SystemClock:
    def now(self) -> datetime:
        return datetime.utcnow()

class FakeClock:
    def __init__(self, fixed_time: datetime):
        self._time = fixed_time

    def now(self) -> datetime:
        return self._time

    def advance(self, **kwargs) -> None:
        from datetime import timedelta
        self._time += timedelta(**kwargs)

# Production
class SubscriptionService:
    def __init__(self, clock: Clock):
        self._clock = clock

    def is_expired(self, subscription) -> bool:
        return subscription.expires_at < self._clock.now()

# Test
def test_subscription_is_expired_when_past_expiry_date():
    clock = FakeClock(fixed_time=datetime(2025, 6, 1))
    service = SubscriptionService(clock=clock)
    subscription = Subscription(expires_at=datetime(2025, 5, 31))

    assert service.is_expired(subscription) is True

def test_subscription_advances_with_time():
    clock = FakeClock(fixed_time=datetime(2025, 5, 30))
    service = SubscriptionService(clock=clock)
    subscription = Subscription(expires_at=datetime(2025, 5, 31))

    assert service.is_expired(subscription) is False
    clock.advance(days=2)
    assert service.is_expired(subscription) is True
```

---

## 2. Randomness

**The problem:** Functions using `random()`, `uuid()`, or cryptographic generators produce different values each run. You can't assert on specific values.

**Solutions:**

### Option A — Inject the random source
```python
class TokenGenerator(Protocol):
    def generate(self) -> str: ...

class UUIDTokenGenerator:
    def generate(self) -> str:
        return str(uuid.uuid4())

class FakeTokenGenerator:
    def __init__(self, tokens: list[str]):
        self._tokens = iter(tokens)

    def generate(self) -> str:
        return next(self._tokens)

# Test
def test_user_receives_unique_token_on_registration():
    tokens = FakeTokenGenerator(tokens=["token-abc-123"])
    service = UserService(token_generator=tokens)

    user = service.register("alice@example.com")

    assert user.verification_token == "token-abc-123"
```

### Option B — Assert on properties, not values
When you can't inject the source, assert on structure/format:
```python
def test_generated_token_is_uuid_format():
    token = service.generate_token()
    assert re.match(r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$', token)
```

### Option C — Seed the random number generator
For algorithms that use random internally and can't be refactored:
```python
import random
random.seed(42)  # deterministic sequence
result = run_stochastic_algorithm(data)
assert result == expected_for_seed_42
```

---

## 3. Async / Concurrent Code

**The problem:** Async code introduces timing dependencies and race conditions that are hard to reproduce.

**The solution:** Test async behavior deterministically by controlling the event loop in tests.

```typescript
// TypeScript / Jest
describe('UserService', () => {
  it('fetches user and caches result', async () => {
    const repo = new FakeUserRepository();
    repo.seed(makeUser({ id: 'u-1', name: 'Alice' }));
    const service = new UserService(repo);

    const result = await service.getUser('u-1');

    expect(result.name).toBe('Alice');
    expect(repo.callCount('findById')).toBe(1);

    // Second call should hit cache
    await service.getUser('u-1');
    expect(repo.callCount('findById')).toBe(1); // not called again
  });
});
```

```python
# Python / pytest-asyncio
@pytest.mark.asyncio
async def test_async_email_send_retries_on_failure():
    email_service = FakeEmailService(fail_first_n=2)
    sender = ResilientEmailSender(email_service, max_retries=3)

    await sender.send(Email(to="alice@example.com", subject="Hi"))

    assert email_service.call_count == 3  # 2 failures + 1 success
    assert len(email_service.outbox) == 1
```

**Rules for async tests:**
- Never use `asyncio.sleep()` in tests — use advance-able fake clocks or cooperative mocking
- Always `await` properly — forgotten awaits are a common source of false greens
- For concurrency bugs, write a test that exercises the race condition explicitly (e.g., run two concurrent operations)

---

## 4. External APIs (HTTP)

**The problem:** Calling real external APIs in tests makes them slow, flaky, and dependent on network/service availability.

**Approach A — Fake (preferred):**
```python
class FakePaymentGateway:
    def __init__(self):
        self.charges: list[Charge] = []
        self._should_fail = False

    def will_fail(self):
        self._should_fail = True
        return self  # fluent

    def charge(self, amount: Money, card: str) -> ChargeResult:
        if self._should_fail:
            raise PaymentDeclined("Card declined")
        charge = Charge(id=f"ch_{uuid4()}", amount=amount)
        self.charges.append(charge)
        return ChargeResult(success=True, charge_id=charge.id)
```

**Approach B — Record/Replay (VCR / Polly):**
Record real HTTP interactions once, replay them in tests. Use when you can't build a fake (complex protocol, binary format).

```python
# Python — vcrpy
@pytest.mark.vcr  # records first run, replays after
def test_weather_api_returns_temperature():
    client = WeatherAPIClient(api_key="test")
    weather = client.get_current("London")
    assert weather.temperature_celsius > -50
```

**Approach C — Contract Tests:**
For your own external services (microservices), use Pact (see Specialist F).

---

## 5. Databases

**The problem:** Real databases are slow to set up and can leave state between tests.

**Strategy:**

| Pattern | When to use |
|---------|------------|
| **In-memory fake** | Application-layer tests. Fast. `FakeOrderRepository`. |
| **Real DB in transaction** | Integration tests. Each test runs in a transaction that rolls back after. |
| **Real DB with cleanup** | When transactions aren't viable (e.g., testing rollback behavior itself). |
| **Test containers** | CI. Spin up a real DB instance per test run. |

```python
# Python / Django — transaction rollback pattern
@pytest.mark.django_db(transaction=False)  # wraps in transaction, rolls back
def test_order_is_persisted_to_database():
    repo = DjangoOrderRepository()
    order = make_order()

    repo.save(order)

    assert repo.find_by_id(order.id) is not None
    # Transaction rolls back after test — no cleanup needed
```

**Rules:**
- Never share database state between tests
- Never test business logic against the real database (that's an integration test concern)
- Use database factories (factory_boy, FactoryBot) for test data, not raw SQL

---

## 6. File System

**The problem:** File I/O is slow and leaves artifacts.

**Approach A — In-memory fake:**
```python
class FakeFileStorage:
    def __init__(self):
        self._files: dict[str, bytes] = {}

    def write(self, path: str, content: bytes) -> None:
        self._files[path] = content

    def read(self, path: str) -> bytes:
        if path not in self._files:
            raise FileNotFoundError(path)
        return self._files[path]

    def exists(self, path: str) -> bool:
        return path in self._files
```

**Approach B — Temp directory (for integration tests):**
```python
def test_report_is_written_to_disk(tmp_path):  # pytest's built-in tmp_path
    generator = ReportGenerator(output_dir=tmp_path)
    generator.generate(report_data)
    assert (tmp_path / "report.pdf").exists()
```

---

## 7. Message Queues / Event Buses

```python
class FakeEventBus:
    def __init__(self):
        self.published: list[DomainEvent] = []

    def publish(self, event: DomainEvent) -> None:
        self.published.append(event)

    def published_of_type(self, event_type: type) -> list:
        return [e for e in self.published if isinstance(e, event_type)]

# Test
def test_order_placed_event_published_on_checkout():
    event_bus = FakeEventBus()
    handler = CheckoutHandler(repo=FakeOrderRepo(), event_bus=event_bus)

    handler.handle(CheckoutCommand(cart_id="cart-1", user_id="user-1"))

    events = event_bus.published_of_type(OrderPlaced)
    assert len(events) == 1
    assert events[0].cart_id == "cart-1"
```

---

## 8. Legacy Code — The Seam Model

Michael Feathers (*Working Effectively with Legacy Code*) defines a **seam** as "a place where you can alter behavior in your program without editing in that place."

**Process for testing legacy code:**

1. **Identify the seam** — find a parameter, a virtual method, a global variable you can replace
2. **Characterize the current behavior** — write a test that passes with the current (possibly buggy) behavior. This is your safety net.
3. **Extract and inject the dependency** — refactor the code to accept the dependency via parameter
4. **Now write your real test** — inject a fake and assert on the correct behavior

```python
# Legacy — untestable
class OrderProcessor:
    def process(self, order_id: str):
        db = DatabaseConnection()         # hard-coded dependency
        order = db.query(f"SELECT * FROM orders WHERE id = '{order_id}'")
        EmailClient().send(order.email, "Processed")

# Step 1 — Extract and inject
class OrderProcessor:
    def __init__(self, db: OrderRepository, email: EmailService):
        self._db = db
        self._email = email

    def process(self, order_id: str):
        order = self._db.find_by_id(order_id)
        self._email.send(order.email, "Processed")

# Now testable
def test_order_processor_sends_confirmation_email():
    order = make_order(email="alice@example.com")
    repo = FakeOrderRepo(orders=[order])
    email = FakeEmailService()
    processor = OrderProcessor(db=repo, email=email)

    processor.process(order.id)

    assert email.outbox[0].to == "alice@example.com"
```

**Feathers' legacy TDD rules:**
1. Do not write new features without a test
2. When you touch a class to add a feature, add tests for the existing behavior first
3. Never refactor without a safety net of characterization tests
