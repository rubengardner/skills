## SPECIALIST C: Naming Conventions

### Master Table

| Construct | Convention | Examples | Anti-examples |
|-----------|------------|---------|---------------|
| Module | `snake_case` | `user_service.py` | `UserService.py`, `userservice.py` |
| Package | `snake_case`, short | `myproject`, `utils` | `MyProject`, `my-project` |
| Class | `PascalCase` | `UserService`, `HTTPClient` | `User_service`, `Httpclient` |
| Exception | `PascalCase` + `Error` | `ValidationError`, `NotFoundError` | `UserException`, `invalid` |
| Function/Method | `snake_case` | `get_user()`, `parse_response()` | `GetUser()`, `parseResponse()` |
| Variable | `snake_case` | `user_id`, `response_body` | `userId`, `ResponseBody` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` | `maxRetries`, `max_retries` |
| Type alias | `PascalCase` | `UserId = int` | `user_id_type` |
| TypeVar | `PascalCase` T/descriptive | `T`, `KeyT`, `ValueT` | `t`, `key_type` |
| Boolean var/fn | `is_`, `has_`, `can_`, `should_` prefix | `is_active`, `has_permission` | `active`, `permission` |

### Visibility Prefixes

| Prefix | Meaning | When to use |
|--------|---------|-------------|
| `name` | Public API | Stable, documented, tested |
| `_name` | Internal | Implementation detail, avoid depending on externally |
| `__name` | Name-mangled | Only to avoid subclass collision — rare |
| `__name__` | Dunder/magic | Only for Python protocol implementations |

**Rules:**
- Acronyms in class names: capitalize the full acronym → `HTTPClient`, `URLParser`, `JSONEncoder` (not `HttpClient`)
- Factory functions: `create_`, `make_`, `build_` prefix → `create_user()`, `build_response()`
- Never single-letter names except `i`, `j`, `k` (loop counters) and `T`, `K`, `V` (TypeVars)
- Forbidden single letters: `l`, `O`, `I` — visually ambiguous (ruff E741)
- `__double_underscore` (name mangling) impairs testability. Use `_single` for "private by convention"

---
