# Specialist C: Test Doubles

## Scope
Precise definitions of all five test double types, decision rules for choosing between them, implementation examples, and the hierarchy of preference.

---

## 1. The Five Types — Precise Definitions

Gerard Meszaros defined these in *xUnit Test Patterns* (2007). These are not interchangeable terms.

| Type | Purpose | Has behavior? | Records calls? | Verifies calls? |
|------|---------|--------------|---------------|----------------|
| **Dummy** | Fills a required parameter — never used | No | No | No |
| **Stub** | Returns configured values — controls indirect input | Yes (fixed) | No | No |
| **Spy** | Stub that also records how it was called | Yes (fixed) | Yes | No (manually) |
| **Mock** | Pre-programmed with expectations — verifies interaction | Yes (fixed) | Yes | Yes (automatic) |
| **Fake** | Working implementation with shortcuts — no real I/O | Yes (real logic) | No | No |

---

## 2. Dummy

A dummy is an object passed to satisfy a required parameter. It is **never called or used** in the test path being tested.

```python
# Python
class NullLogger:  # Dummy — satisfies the logger parameter, never called
    def log(self, message: str) -> None:
        pass

def test_order_total_calculation():
    logger = NullLogger()   # dummy
    order = Order(logger=logger, items=[Item(price=10), Item(price=20)])
    assert order.total() == 30
```

**Rule:** Use a dummy when a dependency is required by the constructor but irrelevant to the behavior being tested. Don't use a mock framework for this — write a no-op class or pass `None` if the interface allows it.

---

## 3. Stub

A stub returns pre-configured values to control the **indirect inputs** into the system under test. It has no logic — it just answers with what you tell it to answer.

```python
# Python
class StubPricingService:
    def __init__(self, price: float):
        self._price = price

    def get_price(self, sku: str) -> float:
        return self._price  # always returns the configured price

def test_order_total_uses_pricing_service():
    pricing = StubPricingService(price=15.00)
    order = Order(pricing_service=pricing)
    order.add_item(sku="SKU-001", quantity=2)
    assert order.total() == 30.00
```

```typescript
// TypeScript
class StubUserRepository implements UserRepository {
  constructor(private readonly user: User) {}

  async findById(_id: string): Promise<User | null> {
    return this.user;  // always returns the configured user
  }

  async save(_user: User): Promise<void> {}
}
```

**Rule:** Stubs answer questions. They return values that drive the system under test down a specific code path.

---

## 4. Spy

A spy is a stub that also records how it was called. You inspect it **after** the act phase.

```python
# Python
class SpyEmailService:
    def __init__(self):
        self.sent_emails: list[dict] = []

    def send(self, to: str, subject: str, body: str) -> None:
        self.sent_emails.append({"to": to, "subject": subject, "body": body})

def test_welcome_email_sent_on_registration():
    email_service = SpyEmailService()
    registration_handler = RegistrationHandler(email_service=email_service)

    registration_handler.handle(RegisterCommand(email="alice@example.com"))

    assert len(email_service.sent_emails) == 1
    assert email_service.sent_emails[0]["to"] == "alice@example.com"
    assert "Welcome" in email_service.sent_emails[0]["subject"]
```

**Rule:** Use a spy when you need to verify what was sent to a dependency, but the interaction doesn't need to change the system's behavior. Prefer spies over mocks because assertions are explicit in the `ASSERT` phase.

---

## 5. Mock

A mock has pre-programmed **expectations** that are verified automatically — usually by a framework. Verification happens implicitly at the end of the test.

```python
# Python — with pytest-mock / unittest.mock
def test_welcome_email_sent_on_registration(mocker):
    email_service = mocker.Mock()

    registration_handler = RegistrationHandler(email_service=email_service)
    registration_handler.handle(RegisterCommand(email="alice@example.com"))

    email_service.send.assert_called_once_with(
        to="alice@example.com",
        subject=ANY,
        body=ANY,
    )
```

**When mocks are appropriate:**
- Verifying that a specific interaction **protocol** is followed (e.g., "exactly one email, to the right address")
- Verifying that something was **NOT** called
- Testing an interaction that has no observable output other than the call itself

**When mocks are NOT appropriate:**
- When the interaction affects state you can check another way (use a fake)
- When you end up with more mock setup than actual test logic
- When changing the implementation would break the test even though behavior is unchanged

---

## 6. Fake

A fake is a **working implementation** that takes shortcuts to avoid real I/O. It has real logic, not just canned responses.

```python
# Python — Fake repository with real in-memory logic
class FakeUserRepository:
    def __init__(self):
        self._store: dict[str, User] = {}
        self.saved: list[User] = []  # audit list for assertions

    def find_by_id(self, user_id: str) -> User | None:
        return self._store.get(user_id)

    def find_by_email(self, email: str) -> User | None:
        return next((u for u in self._store.values() if u.email == email), None)

    def save(self, user: User) -> None:
        self._store[user.id] = user
        self.saved.append(user)  # record for assertions

    def delete(self, user_id: str) -> None:
        self._store.pop(user_id, None)
```

```typescript
// TypeScript — Fake in-memory email outbox
class FakeEmailService implements EmailService {
  readonly outbox: Email[] = [];

  async send(email: Email): Promise<void> {
    this.outbox.push(email);
  }
}

// In test
it('sends welcome email on registration', async () => {
  const emailService = new FakeEmailService();
  const handler = new RegistrationHandler(emailService);

  await handler.handle({ email: 'alice@example.com' });

  expect(emailService.outbox).toHaveLength(1);
  expect(emailService.outbox[0].to).toBe('alice@example.com');
});
```

**Why fakes beat mocks for complex tests:**
- They survive refactoring — method renames don't break them
- They test realistic behavior (multiple calls work correctly)
- They're readable — assertions are explicit
- They can be reused across the entire test suite
- They document what the real interface does

---

## 7. Decision Rules — Which to Choose

```
Do you need the dependency to actually work (with shortcuts)?
  YES → FAKE

Does the dependency return values that drive the SUT?
  YES → STUB (or SPY if you also need to verify what was passed)

Does the dependency have no return value and you need to verify it was called?
  YES → SPY (manual assertion) or MOCK (framework assertion)
  Prefer SPY — it keeps assertions in the ASSERT phase

Is the parameter required but completely irrelevant to this test?
  YES → DUMMY
```

**The hierarchy of preference (most to least preferred):**
1. **Real object** — use the actual implementation if it's fast and has no I/O
2. **Fake** — for anything with I/O, slow setup, or shared resources
3. **Stub** — for configuring indirect inputs
4. **Spy** — for verifying side effects with explicit assertions
5. **Mock** — only when interaction protocol verification is the sole objective

---

## 8. Test Double Implementation Rules

- Test doubles **must implement the same interface** as the real dependency. If the real class is typed, the fake must satisfy the same type.
- Test doubles live in the **test code** (or a shared `test-support` package if reused across services).
- Fakes are **production-quality code** in test infrastructure. Write them carefully. They will be reused.
- Never put test doubles in production code packages.
- One fake per interface, not one fake per test.
