## SPECIALIST E: Code Complexity

### Function Length

| Length | Status |
|--------|--------|
| ≤30 lines | Preferred |
| ≤50 lines | Acceptable |
| >50 lines | Refactor required |

Exception: long `match`/`case` or data-table functions may be acceptable. Justify with a comment.

### Cyclomatic Complexity (ruff C90, max=10)

Each `if`, `elif`, `for`, `while`, `try`, `except`, `with`, `and`, `or` adds 1.

| Score | Action |
|-------|--------|
| 1–5 | Good |
| 6–10 | Acceptable |
| 11–15 | Refactor if possible |
| >15 | Must refactor |

Suppress only with `# noqa: C901` + explanatory comment.

### Nesting Limit: 3 levels maximum

```python
# BAD — 4 levels
def process(items):
    for item in items:
        if item.active:
            for sub in item.children:
                if sub.valid:
                    do_work(sub)

# GOOD — guard clause + extracted function
def process(items):
    for item in items:
        if not item.active:
            continue
        _process_active_item(item)

def _process_active_item(item):
    for child in (s for s in item.children if s.valid):
        do_work(child)
```

### Early Returns (Guard Clauses)

Prefer early returns over nested if/else. Reduces nesting, clarifies the happy path.

```python
# BAD
def get_display_name(user):
    if user is not None:
        if user.is_active:
            if user.display_name:
                return user.display_name
            else:
                return user.username
        else:
            return "inactive"
    else:
        return "anonymous"

# GOOD
def get_display_name(user):
    if user is None:
        return "anonymous"
    if not user.is_active:
        return "inactive"
    return user.display_name or user.username
```

### Boolean Complexity

Max 3 conditions per expression. Extract named booleans:

```python
# BAD
if user.role == "admin" and user.is_active and not user.is_suspended and org.is_verified:

# GOOD
is_authorized = user.role == "admin" and user.is_active and not user.is_suspended
if is_authorized and org.is_verified:
```

---
