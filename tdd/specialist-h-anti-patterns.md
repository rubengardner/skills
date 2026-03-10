# Specialist H: Anti-Patterns

## Scope
The canonical list of TDD and test design anti-patterns, how to identify each, and how to fix it.

---

## 1. The Liar (False Green)

**What it is:** A test that always passes — even when the code is broken.

**Causes:**
- Missing assertion
- Assertion on the wrong object
- Exception swallowed silently
- Async operation not awaited

```python
# BAD — no assertion
def test_order_is_saved():
    repo = FakeOrderRepo()
    handler = PlaceOrderHandler(repo=repo)
    handler.handle(PlaceOrderCommand(items=[]))
    # no assert — always passes, even if handler throws

# BAD — asserting on wrong object
def test_order_total():
    order = Order(items=[])
    total = calculate_total(order)
    assert order is not None  # this is always True!

# FIX — assert on the actual outcome
def test_order_is_saved():
    repo = FakeOrderRepo()
    handler = PlaceOrderHandler(repo=repo)
    handler.handle(PlaceOrderCommand(items=[{"sku": "A", "qty": 1}]))
    assert len(repo.saved) == 1

def test_order_total():
    order = Order(items=[Item(price=10), Item(price=20)])
    assert calculate_total(order) == 30
```

**Prevention:** Always verify the red step. A test you have never seen fail is not a test you trust.

---

## 2. The Excessive Mock

**What it is:** A test that configures so many mock expectations it's impossible to understand what's being tested.

```python
# BAD — 80% setup, 20% assertion, tests nothing meaningful
def test_checkout(mocker):
    cart = mocker.Mock()
    cart.items.return_value = [mocker.Mock(sku="A", qty=1)]
    cart.is_empty.return_value = False
    inventory = mocker.Mock()
    inventory.check_stock.return_value = True
    payment = mocker.Mock()
    payment.charge.return_value = mocker.Mock(success=True)
    notifier = mocker.Mock()
    repo = mocker.Mock()

    CheckoutHandler(cart, inventory, payment, notifier, repo).handle("user-1")

    payment.charge.assert_called_once()
    repo.save.assert_called_once()
    # What behavior was tested? Impossible to tell.
```

**Fix:** Replace mocks with fakes; assert on outcomes, not call counts.

```python
# GOOD
def test_checkout_creates_confirmed_order():
    cart = FakeCart(items=[CartItem(sku="A", qty=1)])
    inventory = FakeInventory(stock={"A": 10})
    payment = FakePayment(response="success")
    repo = FakeOrderRepo()

    CheckoutHandler(cart, inventory, payment, repo).handle(user_id="user-1")

    order = repo.saved[0]
    assert order.status == "confirmed"
    assert order.user_id == "user-1"
```

---

## 3. The Inspector (Testing Privates)

**What it is:** A test that accesses private methods, internal attributes, or non-public state.

```python
# BAD — inspects private state
def test_cache_is_warmed():
    service = ProductService(repo=FakeProductRepo())
    service.get_product("SKU-1")
    assert "SKU-1" in service._cache  # private implementation detail

# BAD — calls private method directly
def test_internal_validation():
    order = Order()
    result = order._validate_items([])  # private method
    assert result is False
```

**Fix:** Test through the public API. If a private behavior matters, expose it through a public one.

```python
# GOOD — test the observable outcome of caching
def test_product_is_returned_in_under_1ms_on_second_call():
    # Or better: test that the repo is only called once
    repo = FakeProductRepo(products=[make_product(sku="SKU-1")])
    service = ProductService(repo=repo)

    service.get_product("SKU-1")
    service.get_product("SKU-1")

    assert repo.call_count("find_by_sku") == 1  # repo hit only once
```

---

## 4. The Giant (Slow Test)

**What it is:** A unit test that takes seconds. It's loading a framework, hitting a database, or doing real I/O.

**Causes:**
- Unit test that calls the real database
- Unit test that imports the full application framework
- Unit test with a `sleep()` call

**Fix:** Every import of a database, HTTP client, or framework in a unit test is a design smell. Inject the dependency and replace it with a fake.

**Rule:** If the unit test suite takes > 10 seconds, there are integration concerns in unit tests.

---

## 5. The Fragile Test (Implementation Coupling)

**What it is:** A test that breaks when you refactor the implementation without changing behavior.

```python
# BAD — test breaks if you rename _calculate_subtotal
def test_checkout_uses_calculate_subtotal(mocker):
    mocker.patch.object(CheckoutService, '_calculate_subtotal', return_value=90.0)
    service = CheckoutService()
    result = service.checkout(items)
    assert result.subtotal == 90.0

# BAD — test breaks if you change from list to set internally
def test_unique_skus():
    order = Order(items=[Item("A"), Item("B"), Item("A")])
    assert type(order._unique_skus) == list  # implementation detail
```

**Fix:** Only test the public contract. Refactoring (changing how) must not break tests.

```python
# GOOD — tests behavior regardless of internal implementation
def test_duplicate_items_are_deduplicated():
    order = Order(items=[Item("A"), Item("B"), Item("A")])
    assert order.unique_sku_count() == 2
```

---

## 6. The Fixture Junkie (Over-Shared Setup)

**What it is:** A massive `setUp`, `beforeEach`, or `@pytest.fixture` that creates the entire world for every test. Individual tests become impossible to understand without reading the fixture.

```python
# BAD — what is the starting state for test_x? Buried in 60-line setUp
class TestCheckout(unittest.TestCase):
    def setUp(self):
        self.user = User(id="u-1", email="...", role="admin", ...)
        self.product_a = Product(sku="A", price=10, stock=100, ...)
        self.product_b = Product(sku="B", price=20, stock=50, ...)
        self.cart = Cart(user=self.user)
        self.cart.add(self.product_a, qty=2)
        self.cart.add(self.product_b, qty=1)
        self.discount = Discount(code="SAVE10", ...)
        # ... 40 more lines
```

**Fix:** Each test should create only what it needs. Use factory functions for common objects.

```python
# GOOD — each test is self-contained
def test_cart_total_with_discount():
    cart = Cart()
    cart.add(make_product(price=10), qty=2)
    cart.add(make_product(price=20), qty=1)

    cart.apply_discount(make_discount(pct=10))

    assert cart.total() == 36.00

def test_cart_total_without_discount():
    cart = Cart()
    cart.add(make_product(price=10), qty=2)

    assert cart.total() == 20.00
```

---

## 7. The Happy Path Only

**What it is:** A test suite that only covers the nominal case. No edge cases, no error paths, no boundary conditions.

```
# Test names — all happy path
test_user_can_login()
test_order_can_be_placed()
test_payment_succeeds()
```

**Fix:** For every feature, list:
- What happens when input is missing?
- What happens when the dependency fails?
- What happens at the boundary (empty collection, max value)?
- What happens when state is wrong?

```
test_login_succeeds_with_correct_credentials()
test_login_fails_with_wrong_password()
test_login_fails_with_unknown_email()
test_login_locks_account_after_five_failures()
test_login_fails_for_locked_account()
test_login_fails_when_session_store_is_unavailable()
```

---

## 8. The Sleeper (Non-Deterministic Timing)

**What it is:** A test with `time.sleep()`, `Thread.sleep()`, or `await delay()` to wait for async side effects.

```python
# BAD — how long is long enough? Test is slow and still flaky.
def test_background_job_runs():
    job_runner.trigger("send-report")
    time.sleep(2)  # hope it's done
    assert email_spy.was_called()
```

**Fix:** Control time explicitly (see Specialist G). If you can't, use an explicit wait with timeout and polling:

```python
# Acceptable fallback — explicit polling with timeout
def wait_for(condition, timeout=5.0, interval=0.1):
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if condition():
            return
        time.sleep(interval)
    raise TimeoutError("Condition not met within timeout")

def test_background_job_sends_report():
    job_runner.trigger("send-report")
    wait_for(lambda: len(email_spy.outbox) > 0, timeout=5)
    assert email_spy.outbox[0].subject == "Your Report"
```

---

## 9. The Mockery (Testing the Test Double)

**What it is:** A test where the test double (mock/stub) is so complex that it needs its own tests — and the actual behavior is untested.

**Fix:** If your fake is complex, write unit tests for the fake separately. The fake is test infrastructure, but it's also code that can be wrong.

```python
# FakeOrderRepository is complex enough to warrant its own tests
def test_fake_repository_find_by_id_returns_none_for_unknown():
    repo = FakeOrderRepository()
    assert repo.find_by_id("unknown") is None

def test_fake_repository_returns_saved_order():
    repo = FakeOrderRepository()
    order = make_order()
    repo.save(order)
    assert repo.find_by_id(order.id) == order
```

---

## 10. The Irrelevant Assertion

**What it is:** An assertion that was always going to pass regardless of the behavior being tested.

```python
# BAD — last assertion is always true for any Exception
def test_raises_on_invalid_input():
    with pytest.raises(ValueError) as exc_info:
        process(invalid_input)
    assert exc_info.value is not None  # always True for any exception!

# GOOD — assert on the specific message/type
def test_raises_on_invalid_input():
    with pytest.raises(ValueError, match="Input cannot be negative"):
        process(input=-1)
```

---

## Anti-Pattern Quick Reference

| Anti-Pattern | Signal | Fix |
|--------------|--------|-----|
| Liar | Test never fails | Verify red step; add missing assertion |
| Excessive Mock | > 3 mocks in one test | Replace with fakes; assert on outcomes |
| Inspector | Accesses `_private` | Test via public interface |
| Giant | Test takes > 100ms | Inject and fake the slow dependency |
| Fragile | Breaks on refactor | Test behavior, not implementation |
| Fixture Junkie | Large setUp required | Local factories; inline setup |
| Happy Path Only | No error/edge tests | Add failure and boundary tests |
| Sleeper | `sleep()` in test | Inject clock; use polling-with-timeout |
| Mockery | Mock setup > 10 lines | Build a proper Fake |
| Irrelevant Assertion | Always passes | Assert on specific values |
