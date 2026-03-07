## SPECIALIST I: Testing

### Framework: pytest only

`unittest` is acceptable input (runs under pytest), but never write new tests using `unittest.TestCase` directly.

### File Structure

```
tests/
├── conftest.py                   # shared fixtures
├── test_user_service.py          # mirrors src/myproject/user_service.py
├── test_http_client.py
└── integration/
    ├── conftest.py               # integration-specific fixtures
    └── test_api_endpoints.py
```

**Naming:**
- File: `test_<module_name>.py`
- Function: `test_<what>_<scenario>` or `test_<what>_when_<condition>_then_<outcome>`
- Class: `Test<ClassName>` (only when grouping tests for a single class under test)
- Fixture: descriptive noun — `authenticated_client`, `test_database`, `sample_user`

### Fixture Patterns

```python
# tests/conftest.py
import pytest
from myproject.config import Settings

@pytest.fixture(scope="session")
def test_settings() -> Settings:
    return Settings(
        database_url="postgresql://localhost:5432/test_db",
        api_key="test-key",
        jwt_secret="test-secret-minimum-32-characters-xx",
    )

@pytest.fixture(autouse=True)
def override_settings(test_settings, monkeypatch):
    monkeypatch.setattr("myproject.config.get_settings", lambda: test_settings)
```

**Fixture scopes:**
- `function` (default): stateful, recreate every test
- `module`: expensive read-only setup shared across the file
- `session`: DB connections, Docker containers
- `class`: use sparingly

### AAA Structure

```python
def test_fetch_user_returns_none_when_not_found(user_repository):
    # Arrange
    nonexistent_id = 99999

    # Act
    result = user_repository.fetch(nonexistent_id)

    # Assert
    assert result is None

def test_fetch_user_raises_on_invalid_id(user_repository):
    with pytest.raises(ValueError, match="User ID must be positive"):
        user_repository.fetch(-1)
```

### What to Test / Not Test

| Test | Skip |
|------|------|
| Public API surface | Third-party library behavior |
| Edge cases (None, 0, empty, max) | Standard library internals |
| Error paths + exception messages | Pure delegation with no logic |
| Business logic invariants | Private implementation details |
| Boundary conditions | Trivial property accessors |

**Coverage target:** 80% minimum, 90%+ for critical paths. Never chase 100% — it signals wasted effort on untestable error paths.

### Parametrize

```python
@pytest.mark.parametrize(
    ("input_value", "expected"),
    [(0, "zero"), (1, "positive"), (-1, "negative"), (None, "unknown")],
    ids=["zero", "positive", "negative", "none"],
)
def test_classify_number(input_value, expected):
    assert classify_number(input_value) == expected
```

---
