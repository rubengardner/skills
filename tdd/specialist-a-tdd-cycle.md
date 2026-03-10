# Specialist A: The TDD Cycle

## Scope
Red-Green-Refactor mechanics, what to do at each phase, baby steps, the discipline of not writing ahead, committing to a test.

---

## 1. The Three Phases

```
RED → GREEN → REFACTOR → RED → GREEN → REFACTOR → ...
```

This is not a suggestion. It is the only order. Deviating from it — writing code before a test, or refactoring on red — breaks TDD.

---

## 2. RED — Write a Failing Test

**What you do:**
- Write a single test that describes one piece of behavior the system does not yet have.
- Run the suite. It must fail. If it passes, the test is wrong or the behavior already exists.
- A compilation error counts as a failure. Red is red.

**What makes a good Red step:**
- The test name is the specification: "places order and emits OrderPlaced event"
- The failure message is meaningful: it tells you exactly what is missing
- You can predict what the failure message will be before running it — if you can't, the test is too large

**Rules:**
- Write only one test. Resist the urge to write the next three tests you can already see.
- Do not write any production code yet. Not even the class definition (unless needed to compile).
- The failing test is your contract. Once written, do not change the test to make it easier to pass.

```python
# Example — Python
def test_places_order_and_reduces_inventory():
    inventory = Inventory(stock={"SKU-001": 10})
    order = Order(items=[OrderItem(sku="SKU-001", quantity=3)])

    inventory.place(order)

    assert inventory.stock_for("SKU-001") == 7
    # Also verifies OrderPlaced event was emitted
    assert inventory.events == [OrderPlaced(sku="SKU-001", quantity=3)]
```

At this point `Inventory`, `Order`, `OrderItem`, `OrderPlaced` may not exist yet. That's fine. Run the test. See it fail. Move to Green.

---

## 3. GREEN — Write the Minimum Code to Pass

**What you do:**
- Write the simplest code that makes the failing test pass.
- Do not write code for behavior that isn't yet tested.
- Do not generalize. Do not anticipate. Do not add parameters "just in case."

**The Minimum Code rule:**
If returning a hardcoded value makes the test pass, return a hardcoded value. The next test will force you to generalize.

```python
# Minimum green — hardcoded, obviously incomplete, and that's correct
class Inventory:
    def __init__(self, stock: dict[str, int]):
        self._stock = stock
        self.events: list = []

    def place(self, order):
        self._stock["SKU-001"] -= 3
        self.events.append(OrderPlaced(sku="SKU-001", quantity=3))

    def stock_for(self, sku: str) -> int:
        return self._stock[sku]
```

This is ugly. Good. The next test will force it to be correct. Do not fix what isn't broken yet.

**Rules:**
- Stop writing code the moment the test goes green. Not one line more.
- Run the full suite on green, not just the new test. If you've broken another test, fix it before refactoring.
- If you can't make the test pass without a massive change, the test was too large. Break it down.

---

## 4. REFACTOR — Improve the Design Without Changing Behavior

**What you do:**
- With the suite green, improve the code: remove duplication, clarify names, extract abstractions, improve structure.
- Run the suite after every refactoring step. It must stay green.
- Refactor the tests too — they are production code.

**What refactoring is NOT:**
- Adding new behavior. That requires a new test first.
- Changing behavior. Tests must still pass unchanged.
- Making the code "more future-proof." Only refactor for the present design.

**Refactoring targets (in priority order):**
1. Remove duplication (DRY within reason)
2. Clarify names (variables, functions, classes)
3. Extract small, well-named functions
4. Improve data structures
5. Apply design patterns only when the pattern solves a present problem

```python
# After refactoring the green code above
class Inventory:
    def __init__(self, stock: dict[str, int]):
        self._stock = dict(stock)
        self.events: list[DomainEvent] = []

    def place(self, order: Order) -> None:
        for item in order.items:
            if self._stock.get(item.sku, 0) < item.quantity:
                raise InsufficientStock(sku=item.sku)
            self._stock[item.sku] -= item.quantity
            self.events.append(OrderPlaced(sku=item.sku, quantity=item.quantity))

    def stock_for(self, sku: str) -> int:
        return self._stock.get(sku, 0)
```

This generalization was driven by subsequent failing tests, not by anticipation.

---

## 5. Baby Steps — The Engine of TDD

A "baby step" is the smallest possible change that moves you from red to green. The discipline of baby steps:

- Prevents you from writing ahead (speculative code)
- Gives you a specific failure message at every step
- Makes each refactor safe (small surface area of change)
- Creates a natural commit cadence: after each green + refactor, you can commit

**How small is "small"?**

When learning TDD: each test should require < 10 lines of new production code.
When proficient: each test should require < 5 lines.
When practicing kata: each test should require 1-3 lines.

If a test requires 50+ lines of production code to pass, it is not a unit test — it is a design session disguised as a test. Break it down.

---

## 6. The Commit Cadence

Each completed Red-Green-Refactor cycle is a commit-worthy unit:

```
git add .
git commit -m "feat: inventory reduces stock when order is placed"
```

Benefits:
- Every commit in history is a green, working state
- You can always `git stash` or `git reset` to the last working state
- History reads as a specification of the system

---

## 7. When You're Stuck

**Test won't go green no matter what you write:**
- The test is too large. Split it into two smaller tests.
- Your understanding of the problem is wrong. Delete the test, re-think, re-write.

**The design makes the test impossible to write:**
- This is the most valuable signal TDD gives you. The design is wrong. Change the design.
- Hard-to-test = over-coupled, over-responsible, or missing an abstraction.

**You've broken an existing test:**
- Do not comment it out. Do not change it to pass.
- Understand why it broke. Either your new code is wrong, or the existing test is testing the wrong thing (implementation detail). Fix the root cause.
