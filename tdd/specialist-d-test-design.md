# Specialist D: Test Design

## Scope
What to test, behavior vs implementation, the test as first client of the API, test oracles, boundary conditions, granularity, and when NOT to test.

---

## 1. Test Behavior, Not Implementation

The most important principle in test design.

**Behavior** = what the system does from the outside (outputs, state changes, observable side effects)
**Implementation** = how the system does it (algorithms, internal structures, private methods)

```python
# BAD — testing implementation
def test_internal_cache_is_populated():
    service = ProductService(repo=FakeProductRepo())
    service.get_product("SKU-001")
    assert "SKU-001" in service._cache  # private attribute

# GOOD — testing behavior
def test_product_is_returned_by_sku():
    repo = FakeProductRepo(products=[Product(sku="SKU-001", name="Widget")])
    service = ProductService(repo=repo)
    product = service.get_product("SKU-001")
    assert product.name == "Widget"
```

The cache test is fragile — renaming `_cache` breaks it without changing behavior. The behavior test survives any internal refactoring.

---

## 2. The Test as First Client

The test is the **first user** of your API. How it feels to write determines the quality of the design.

**If a test is hard to write, the design is wrong.** Common signals:

| Test difficulty | Design problem |
|----------------|----------------|
| Long, complex setup | Class does too much (SRP violation) |
| Hard to isolate from dependencies | Over-coupling; missing abstractions |
| Need to mock many things | Too many indirect dependencies |
| Can't test without the database | Business logic in the wrong layer |
| Test is testing a private method | Wrong decomposition; missing collaborator |
| Need to control global state | Hidden dependency on singletons |

When you hit a wall, stop and redesign the production code. TDD is a design tool.

---

## 3. What to Test — The Test Oracle

A **test oracle** is the mechanism by which you determine whether the output is correct. Three oracle types:

### State Verification
Assert on the state of the system after the action:
```python
account.deposit(100)
assert account.balance == 600  # was 500
```

### Return Value Verification
Assert on the return value of a function:
```python
result = calculator.divide(10, 2)
assert result == 5.0
```

### Interaction Verification
Assert that a specific interaction with a collaborator occurred (use sparingly):
```python
assert email_spy.sent_to == "alice@example.com"
```

**Priority:** Prefer state or return value verification over interaction verification. Interaction verification couples your test to the implementation's communication patterns.

---

## 4. What to Test — Coverage Targets

Test the **public contract** of each unit:

**Always test:**
- The happy path (nominal case)
- Each distinct failure case (each exception, each error state)
- Boundary conditions (see section 5)
- State transitions (each valid and invalid transition)

**Never test:**
- Private methods (test via the public method that uses them)
- Third-party libraries (they have their own tests)
- Simple getters/setters with no logic
- Framework boilerplate

**Decide case by case:**
- Pure data objects (often over-tested)
- Configuration/wiring (prefer integration tests)

---

## 5. Boundary Conditions

Every non-trivial input has boundaries. Test them explicitly.

| Input type | Boundaries to test |
|------------|-------------------|
| Number | 0, negative, max, min, just above/below threshold |
| String | Empty string, single char, max length, whitespace only, special chars |
| Collection | Empty, single element, many elements, duplicates, max size |
| Nullable | `null` / `None` / `undefined` |
| Date/time | Epoch, far future, DST transitions, leap year |
| Enum/state | Every valid value, invalid value |

```python
# Boundary tests for a discount function
def test_no_discount_for_zero_quantity():
    assert calculate_discount(quantity=0) == 0.0

def test_no_discount_below_threshold():
    assert calculate_discount(quantity=9) == 0.0

def test_discount_applied_at_threshold():
    assert calculate_discount(quantity=10) == 0.10

def test_max_discount_applied_above_ceiling():
    assert calculate_discount(quantity=100) == 0.20

def test_discount_rejects_negative_quantity():
    with pytest.raises(ValueError, match="Quantity cannot be negative"):
        calculate_discount(quantity=-1)
```

---

## 6. Test Granularity

How much should one test exercise?

| Test type | Scope | Speed | Isolation |
|-----------|-------|-------|-----------|
| Unit | One class/function | < 1ms | Complete |
| Narrow integration | A few real collaborators | < 50ms | Partial |
| Broad integration | A full subsystem | < 500ms | None |
| End-to-end | Full stack | Seconds | None |

**Rules:**
- Unit tests are your primary feedback loop. They must be instant.
- A "unit" is a **behavior**, not a class. If testing a behavior requires 3 cooperating classes, test them together as a unit.
- Do not mock everything to achieve one-class-per-test isolation. That's over-isolation. See Specialist E (TDD Schools).

---

## 7. Equivalence Partitioning — Reducing Test Count

You don't need to test every possible input. Partition inputs into **equivalence classes** where behavior is identical for all values in the class:

```
Quantity discount:
  Class 1: quantity < 0     → error (test with: -1)
  Class 2: 0 ≤ quantity < 10 → 0% discount (test with: 0, 5, 9)
  Class 3: 10 ≤ quantity < 50 → 10% discount (test with: 10, 25, 49)
  Class 4: quantity ≥ 50    → 20% discount (test with: 50, 100)
```

Test **one or two values per class** plus the **exact boundary**. You do not need to test every value.

---

## 8. What NOT to Test

| Don't test | Reason |
|------------|--------|
| Private methods directly | Test the public behavior that exercises them |
| Third-party library behavior | They have their own test suites |
| The test double itself | Fakes should have their own unit tests if logic is complex |
| Framework routing/wiring | Framework integration tests cover this |
| Obvious getters: `def get_name(): return self.name` | No logic, no risk |
| Trivial data classes / DTOs | Nothing to test |
| Generated code | Don't test the generator's output |

**The cost of a test:**
Every test has a maintenance cost. An unnecessary test is pure cost. Only write tests for code where:
1. A bug would be possible
2. A bug would be consequential
3. The test would actually catch it

---

## 9. Test-First vs Test-After

|  | Test-First (TDD) | Test-After |
|--|---------|----------|
| Design feedback | Immediate | None |
| Coverage | 100% by construction | Gaps common |
| Overfitting | Prevented (you only write what tests require) | Common |
| Test quality | Tests specification behavior | Tests may mirror implementation |
| Motivation | Built-in (tests drive progress) | Low (tests feel like paperwork) |

**Rule:** Test-after is better than no tests. TDD is better than test-after. There is no argument against TDD for new code. For legacy code, see Specialist G (Testing Hard Things).
