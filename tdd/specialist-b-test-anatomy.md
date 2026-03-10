# Specialist B: Test Anatomy

## Scope
Test structure (AAA / GWT), test naming, what a single test should cover, independence, assertion style.

---

## 1. Test Structure: Arrange-Act-Assert (AAA)

Every test has exactly three phases. They must be visually separated:

```
ARRANGE — Set up the preconditions. The world before the action.
ACT     — Execute the behavior under test. One action only.
ASSERT  — Verify the outcome. State, return value, or observable side effect.
```

```python
# Python
def test_account_balance_decreases_after_withdrawal():
    # Arrange
    account = BankAccount(balance=1000)

    # Act
    account.withdraw(250)

    # Assert
    assert account.balance == 750
```

```typescript
// TypeScript
it('decreases balance after withdrawal', () => {
  // Arrange
  const account = new BankAccount({ balance: 1000 });

  // Act
  account.withdraw(250);

  // Assert
  expect(account.balance).toBe(750);
});
```

**Rules:**
- Never intermix arrange and assert. All setup comes first.
- The Act phase is exactly one line (one method call, one function invocation).
- If your Act phase is two lines, you're testing two behaviors. Split the test.
- Comments (`# Arrange` / `// Act`) are optional when the structure is obvious. Use them when the test is non-trivial.

---

## 2. Test Structure: Given-When-Then (GWT)

GWT is AAA with domain-oriented language. Prefer it when tests describe user/domain behavior:

```
GIVEN — the system is in this state
WHEN  — this event/action occurs
THEN  — the system should be in this state / emit this output
```

```gherkin
# Conceptual (language-agnostic)
GIVEN an account with a balance of 1000
  AND the account is not frozen
WHEN the user withdraws 250
THEN the balance should be 750
  AND a WithdrawalCompleted event should be emitted
```

```python
# Python — GWT in test structure
def test_withdrawal_succeeds_on_active_account():
    # Given
    account = BankAccount(balance=1000, status="active")

    # When
    result = account.withdraw(250)

    # Then
    assert result.success is True
    assert account.balance == 750
    assert account.events[-1] == WithdrawalCompleted(amount=250)
```

---

## 3. Test Naming

Test names are **the specification of your system**. A reader must understand what the system does by reading test names alone — without reading bodies.

### Formula

```
[unit under test] [expected behavior] when/given [precondition]
```

or

```
[expected behavior] when [precondition]
```

### Examples

```
# BAD — maps to a method name
test_withdraw()
testWithdraw()
test_process_payment()

# BAD — too generic
test_account_works()
test_happy_path()

# GOOD — describes behavior
test_balance_decreases_when_withdrawal_is_made()
test_withdrawal_fails_when_balance_is_insufficient()
test_account_freezes_after_three_failed_withdrawals()
withdrawal_emits_event_when_successful()
raises_InsufficientFunds_when_withdrawal_exceeds_balance()
```

### Language conventions

| Language | Convention |
|----------|-----------|
| Python | `def test_behavior_when_condition():` |
| TypeScript/JS | `it('does X when Y', ...)` |
| Go | `func TestBehaviorWhenCondition(t *testing.T)` |
| Java/Kotlin | `@Test fun behaviorWhenCondition()` |
| C# | `[Test] public void BehaviorWhenCondition()` |

**Rule:** In JS/TS, use `it(...)` not `test(...)`. `it` completes the sentence "it does X when Y."

---

## 4. One Behavior Per Test

A test covers one **behavior**, not one **assertion**.

Multiple assertions are fine — as long as they all verify the same behavior:

```python
# GOOD — two assertions, one behavior ("withdrawal reduces balance AND emits event")
def test_withdrawal_reduces_balance_and_emits_event():
    account = BankAccount(balance=1000)
    account.withdraw(250)
    assert account.balance == 750
    assert account.events == [WithdrawalCompleted(amount=250)]

# BAD — two behaviors in one test ("withdrawal" AND "freeze on failure")
def test_withdrawal():
    # ...tests successful withdrawal...
    # ...then tests failed withdrawal leading to freeze...
    # When this fails, which behavior broke?
```

**How to know if you have one behavior:**
- You can describe the test with one sentence starting with "it..."
- If you need "and" in the sentence to describe the test, it's probably two tests

---

## 5. Test Independence

Tests must not depend on each other. Any test must be able to run:
- Alone
- In any order
- Concurrently (ideally)

**What breaks independence:**
- Shared mutable state (global variables, class-level state, module singletons)
- Tests that rely on another test having run first
- Tests that modify the database/filesystem and don't clean up

**Fixes:**
```python
# BAD — shared state between tests
counter = 0

def test_increment():
    global counter
    counter += 1
    assert counter == 1

def test_double_increment():
    global counter
    counter += 2
    assert counter == 2  # passes only if test_increment ran first

# GOOD — each test creates its own state
def test_increment():
    counter = Counter(initial=0)
    counter.increment()
    assert counter.value == 1

def test_double_increment():
    counter = Counter(initial=0)
    counter.increment()
    counter.increment()
    assert counter.value == 2
```

---

## 6. Assertion Style

**Assert on outcomes, not mechanisms:**

```python
# BAD — asserting on internal state
assert order._internal_status_flag == 1

# GOOD — asserting on the public interface
assert order.is_confirmed()
assert order.status == "confirmed"
```

**One concept per assertion:**

```python
# BAD — asserting unrelated things together
def test_order():
    order = create_order()
    assert order.total == 99.99
    assert order.customer.email == "alice@example.com"
    assert len(order.items) == 2
    assert order.created_at is not None
```

Each of these is a separate test. When one assertion fails, the others are skipped, hiding information.

**Exception — verifying a coherent structure:**

```python
# GOOD — multiple assertions that together verify one thing: the order shape
def test_order_is_created_with_correct_shape():
    order = OrderFactory.create(items=2, customer=ALICE)
    assert order.total == 99.99
    assert order.item_count == 2
    assert order.customer_id == ALICE.id
    # These three together define "correct shape"
```

---

## 7. Test as Documentation

The test suite, read sequentially, should tell the story of your system:

```
test_shopping_cart_starts_empty
test_item_can_be_added_to_cart
test_duplicate_item_increases_quantity
test_item_can_be_removed_from_cart
test_cart_total_reflects_item_prices
test_cart_applies_discount_code
test_cart_rejects_expired_discount_code
test_checkout_creates_pending_order
test_checkout_fails_when_cart_is_empty
```

If someone reads only these names, they understand what a shopping cart does. That is the goal.
