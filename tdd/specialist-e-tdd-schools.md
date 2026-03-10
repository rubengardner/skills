# Specialist E: TDD Schools

## Scope
London School (outside-in / mockist) vs Chicago School (inside-out / classicist) — philosophy, mechanics, tradeoffs, and the decision rule for when to use each.

---

## 1. The Two Schools

**Chicago School** (also: Detroit School, Classicist)
- Test behavior through real objects
- Avoid mocks; prefer real collaborators or fakes
- Drive design from the domain inward
- Associated with: Kent Beck, Martin Fowler

**London School** (also: Mockist)
- Test interactions between objects
- Mock all collaborators; focus on communication contracts
- Drive design from the outside (user need) inward
- Associated with: Steve Freeman, Nat Pryce (*Growing Object-Oriented Software, Guided by Tests*)

---

## 2. Chicago School — Mechanics

Start from the domain. Build small units of real behavior. Wire them together. Test at the behavior level.

```python
# Chicago style — no mocks, real collaborators tested together
def test_checkout_creates_confirmed_order():
    inventory = Inventory(stock={"SKU-001": 10})
    cart = Cart()
    cart.add_item(sku="SKU-001", quantity=2)

    order = cart.checkout(inventory=inventory)

    assert order.status == "confirmed"
    assert inventory.stock_for("SKU-001") == 8
```

The test exercises `Cart`, `Inventory`, and whatever `checkout` touches — without mocking any of them. If they're all in-memory and fast, this is perfectly valid.

**When Chicago fails:**
- When real collaborators have I/O (DB, network, filesystem)
- When real collaborators are slow
- When the chain of real objects is too long to set up

In those cases, replace the I/O boundary with a **Fake** — not a mock.

---

## 3. London School — Mechanics

Start from the user need (acceptance test or behavior). Identify the top-level object. Mock its collaborators. Test each object in isolation, focusing on interactions.

```python
# London style — test Order's interaction with its collaborators
def test_place_order_notifies_warehouse(mocker):
    warehouse = mocker.Mock(spec=WarehouseService)
    payment = mocker.Mock(spec=PaymentService)
    order = Order(warehouse=warehouse, payment=payment)

    order.place(items=[OrderItem(sku="SKU-001", quantity=1)], card="4242424242424242")

    warehouse.reserve.assert_called_once_with("SKU-001", quantity=1)
    payment.charge.assert_called_once_with("4242424242424242", amount=ANY)
```

The test verifies that `Order` sends the right messages to its collaborators. It does not verify what those collaborators do with the messages.

**When London works well:**
- Testing application-layer orchestration (use cases, command handlers)
- When collaborators are external systems (email, payment, warehouse)
- When you need to specify the communication protocol between objects

**When London fails:**
- Domain logic (business rules are not about interactions, they're about transformations)
- When mocks become so numerous they obscure what's being tested
- When mock setup is longer than the actual behavior

---

## 4. The Synthesis — When to Use Each

This is not a religious war. Use the right school for the right layer:

| Layer | School | Reason |
|-------|--------|--------|
| **Domain / business logic** | Chicago | Pure functions and state; no I/O; real objects are fast and simple |
| **Application / orchestration** | London (with Fakes) | The behavior IS the interaction — what was called, with what |
| **Infrastructure** | Neither | Use real I/O; these are integration tests |
| **UI / acceptance** | London (behavior-driven) | Drive from user need, stub infrastructure |

```
Application Layer Test (London with Fake):
  - Use case receives a command
  - Calls FakeRepository.save(entity)
  - Dispatches event via FakeEventBus.publish(event)
  → Assert: FakeRepository.saved == [entity], FakeEventBus.published == [event]

Domain Layer Test (Chicago):
  - Call a domain method on a real domain object
  - Inspect the returned value or the new state
  → Assert: state is correct, returned value is correct, domain events are correct
```

---

## 5. The Outside-In Flow (London)

London TDD starts with an **acceptance test** — a failing test that describes a full user journey. This becomes the guiding test. Then you drill inward:

```
1. Write failing acceptance test (full stack behavior)
2. Identify the topmost object needed (e.g., an HTTP handler)
3. Write a unit test for that object — mock its collaborators
4. Make the unit test pass
5. Write a unit test for each mocked collaborator — mock THEIR collaborators
6. Repeat until you reach the infrastructure boundary
7. Write integration tests for the infrastructure
8. Now the acceptance test should pass
```

This produces a **walking skeleton** — the full path exists from day one, even if the logic is minimal.

---

## 6. The Inside-Out Flow (Chicago)

Chicago TDD starts from the smallest useful unit and builds outward:

```
1. Identify the core domain behavior
2. Write a unit test for the smallest piece
3. Make it pass
4. Write the next test, growing the domain object
5. Once domain objects are solid, write application-layer tests
6. Wire the application layer to real infrastructure last
7. Write integration tests for the boundaries
```

This produces a **solid domain model** before any infrastructure concern is addressed.

---

## 7. The Practical Decision Rule

Ask: **"What is this object's primary job?"**

- **Transform data / enforce rules** → Chicago. No mocks. Test the transformation.
- **Orchestrate calls to collaborators** → London (with Fakes). Test the communication.
- **Adapt between two interfaces** → Integration test with real implementations.

```
// Domain service — Chicago
OrderPricer.calculate(order, discounts) → Money
  Test: did it compute the right price for these inputs?

// Application handler — London
PlaceOrderHandler.handle(command) → void
  Test: did it call repo.save(order), eventBus.publish(OrderPlaced)?

// Infrastructure — Integration
DjangoOrderRepository.save(order) → None
  Test: does a row appear in the DB? Can it be found again?
```

---

## 8. Mocks vs Fakes — The Schools' Biggest Practical Difference

London as described in *GOOS* uses mocks heavily. In practice, the community has learned:

> **Fakes are more durable than mocks.**

A mock verifies the exact method name, argument count, and argument values. When you rename a method or add a parameter, the mock breaks — even if behavior is unchanged.

A fake has real logic. It doesn't care about method names changing. It only cares about correct behavior.

**The recommendation:**
- For I/O boundaries (DB, email, queue): **Fake**
- For verifying that an optional notification was triggered with correct data: **Spy**
- For verifying a strict protocol (idempotency, ordering, count): **Mock**

In most codebases: ~80% Fakes + Stubs, ~15% Spies, ~5% Mocks.
