## SPECIALIST F: Documentation

### Docstring Style: Google (default)

Use Google style for all projects unless: (a) scientific/data-heavy and functions have 5+ params → use NumPy style.

```python
def fetch_user(user_id: int, *, include_deleted: bool = False) -> User | None:
    """Retrieve a user by their primary key.

    Args:
        user_id: The primary key of the user to retrieve.
        include_deleted: If True, also return soft-deleted users.

    Returns:
        The User object if found, or None if not found.

    Raises:
        DatabaseConnectionError: If the database is unavailable.
    """
```

### What MUST have docstrings

- Every public module (top-level triple-quoted string)
- Every public class
- Every public function and method
- `__init__` if it does anything non-obvious
- Every `property` that is not self-evident from name + type

### What must NOT have docstrings

- Private helpers where name + type annotations are fully self-explanatory
- Trivial property getters returning a single attribute
- `__repr__`, `__str__` unless format is non-obvious
- Test functions — use descriptive names instead

### Inline Comments: WHY not WHAT

```python
# BAD — states what the code does
x = x + 1  # increment x by 1

# GOOD — states why
x = x + 1  # offset by one — API uses 1-based indexing

# BAD — restates condition
if not user.is_active:  # check if user is not active
    return

# GOOD — adds context
retry_count += 1  # exponential backoff starts after first failure, not immediately
```

### TODO/FIXME Format

```python
# TODO(username): Short description of what needs to be done
# FIXME(username): Something broken that must be fixed
# HACK(username): Workaround — link to issue tracker
# NOTE: Non-obvious decision context
```

---
