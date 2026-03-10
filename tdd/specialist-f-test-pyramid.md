# Specialist F: Test Pyramid & Strategy

## Scope
The test pyramid, what belongs at each level, ratios, integration test strategy, contract tests, and the cost model for each layer.

---

## 1. The Test Pyramid

```
         /\
        /  \
       / E2E \          Few — slow, fragile, expensive
      /--------\
     /Integration\      Some — medium speed, real I/O
    /--------------\
   /   Unit Tests   \   Many — fast, isolated, cheap
  /------------------\
```

**The pyramid shape is the target.** An inverted pyramid (lots of E2E, few unit tests) is a warning sign.

---

## 2. Unit Tests

**Definition:** Tests that exercise a single unit of behavior in complete isolation from I/O and slow collaborators.

**Characteristics:**
- Speed: < 10ms each (ideally < 1ms)
- No database, no network, no filesystem
- All dependencies are test doubles or real in-memory objects
- Deterministic — same result every run

**What belongs here:**
- Domain logic (calculations, rules, transformations, invariants)
- Application orchestration logic (with fakes for I/O)
- Pure functions
- Error handling paths
- Boundary conditions

**Ratio target:** 70–80% of your total tests

```python
# Unit test — no I/O, no framework, pure logic
def test_cart_discount_applied_for_bulk_order():
    cart = Cart()
    cart.add_item(Product(price=10.00), quantity=10)
    assert cart.total() == 90.00  # 10% bulk discount
```

---

## 3. Integration Tests

**Definition:** Tests that exercise the behavior of a component with its real dependencies — real database, real filesystem, real HTTP client.

**Characteristics:**
- Speed: 50ms–2s each
- Uses real external systems (test database, test container, local server)
- Does NOT use test doubles for the layer being integrated
- May use test doubles for OTHER layers not under test

**What belongs here:**
- Repository implementations (does `save` write to the DB correctly? Does `find` return the right row?)
- ORM queries and migrations
- HTTP client adapters (does the client parse the response correctly?)
- File parsers/writers
- Cache adapters
- Message queue producers/consumers

**What does NOT belong here:**
- Business logic — that's in unit tests
- Full user journeys — that's E2E

**Ratio target:** 15–20% of your total tests

```python
# Integration test — real database
@pytest.mark.django_db
def test_order_repository_saves_and_retrieves_order():
    repo = DjangoOrderRepository()
    order = make_order(status="pending", items=2)

    repo.save(order)
    retrieved = repo.find_by_id(order.id)

    assert retrieved is not None
    assert retrieved.id == order.id
    assert retrieved.status == "pending"
    assert len(retrieved.items) == 2
```

---

## 4. End-to-End Tests

**Definition:** Tests that drive the system from the user-facing entry point through to the infrastructure and back.

**Characteristics:**
- Speed: seconds to minutes each
- Exercises the full system (no test doubles)
- Brittle — any layer can break the test
- High maintenance cost

**What belongs here:**
- Critical user journeys ONLY: registration, login, core purchase flow, payment
- Smoke tests — is the system up and responding?
- The 5–10 paths that, if broken, would constitute a P0 incident

**What does NOT belong here:**
- Error cases and edge cases (that's unit tests)
- Every feature (over-investment in E2E destroys velocity)

**Ratio target:** 5–10% of total tests, < 20 total E2E tests for most systems

```python
# E2E test — drives the full stack via the HTTP API
def test_user_can_place_an_order(api_client, db):
    # Register + login
    api_client.post("/users", json={"email": "alice@example.com", "password": "secret"})
    token = api_client.post("/auth/login", json={"email": "alice@example.com", "password": "secret"}).json()["token"]

    # Place order
    response = api_client.post(
        "/orders",
        json={"items": [{"sku": "SKU-001", "quantity": 1}]},
        headers={"Authorization": f"Bearer {token}"},
    )

    assert response.status_code == 201
    assert response.json()["status"] == "confirmed"
```

---

## 5. The Cost Model

Every test has three costs:

| Cost | Unit | Integration | E2E |
|------|------|-------------|-----|
| **Write time** | Low | Medium | High |
| **Run time** | ~1ms | ~500ms | ~5s |
| **Maintenance** | Low | Medium | High |
| **Feedback speed** | Instant | Seconds | Minutes |
| **Failure localization** | Exact | Approximate | "somewhere in the stack" |

**Principle:** Push tests as low in the pyramid as the behavior allows. An integration test that could be a unit test is waste.

---

## 6. Contract Tests

When two services communicate (microservices, event-driven systems), use **contract tests** to verify the interface agreement without running both services:

**Consumer-Driven Contract Testing (Pact):**
1. Consumer writes a test that records what it expects from the provider
2. This becomes the "contract" (a JSON/YAML artifact)
3. Provider runs the contract against its own code to verify it satisfies the consumer

```
Service A (consumer) → tests against a Pact mock provider
                     → generates pact contract file
Service B (provider) → runs the pact contract against itself
                     → fails CI if it no longer satisfies the contract
```

**When to use contract tests:**
- Microservice API boundaries (REST, gRPC, events)
- Event schema agreements between producers and consumers
- When integration tests between services are slow or require both services running

---

## 7. Test Environment Strategy

| Test type | Environment |
|-----------|------------|
| Unit | In-process, no external services |
| Integration (DB) | Real DB in Docker / test container |
| Integration (HTTP) | Real local server or test container |
| E2E | Dedicated staging environment |
| Contract | Local, in-process (Pact mock) |

**Rules:**
- Never run integration or E2E tests against production.
- Unit tests must run without Docker, without network, without environment setup.
- Use `pytest.mark.integration` / `@Tag("integration")` to separate slow tests from the default run.
- The default `make test` / `npm test` should run only unit tests. Integration tests run on `make test-integration` or in CI.

---

## 8. Test Suite Health Metrics

A healthy test suite has:

| Metric | Target |
|--------|--------|
| Unit test suite runtime | < 5 seconds total |
| Unit test count ratio | ≥ 70% of all tests |
| Flaky test count | 0 |
| Tests skipped | 0 (or explicitly justified) |
| Coverage (line) | ≥ 80% for business logic |
| Coverage (branch) | ≥ 70% for business logic |

**Coverage is a floor, not a goal.** 100% coverage with tests that don't assert anything is worthless. Coverage tells you what wasn't tested. It doesn't tell you tests are good.
