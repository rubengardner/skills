## SPECIALIST J: mypy Configuration

### `strict` Mode — What It Enables

`mypy --strict` is equivalent to enabling all of these flags:

| Flag | What it checks |
|------|---------------|
| `--disallow-untyped-defs` | All functions must have type annotations |
| `--disallow-incomplete-defs` | Partial annotations not allowed |
| `--disallow-any-generics` | `list` must be `list[X]`, not bare `list` |
| `--disallow-subclassing-any` | Cannot subclass `Any`-typed classes |
| `--warn-return-any` | Warn when returning `Any` from typed function |
| `--warn-unused-ignores` | Flag `# type: ignore` comments that are no longer needed |
| `--warn-redundant-casts` | Flag `cast()` calls that are no longer necessary |
| `--no-implicit-reexport` | Imported names not in `__all__` are not re-exported |
| `--strict-equality` | Warn on always-false `==` comparisons between incompatible types |
| `--extra-checks` | Additional checks for correctness |

**Rule:** Enable `strict = true` for all new projects. Roll it in incrementally on legacy code using per-module overrides. Do not add pyright — mypy is the single source of truth.

### Recommended `pyproject.toml` Configuration

```toml
[tool.mypy]
python_version = "3.11"
strict = true
files = ["src"]
exclude = ["tests/"]

# Allow tests to be less strictly typed
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
disallow_incomplete_defs = false

# Third-party libraries without stubs
[[tool.mypy.overrides]]
module = [
    "some_untyped_lib.*",
    "another_lib.*",
]
ignore_missing_imports = true
```

### Incremental Adoption Strategy

For legacy codebases, adopt mypy incrementally:

1. Start with `ignore_errors = true` globally
2. Enable per-module with `[[tool.mypy.overrides]]` as you fix each module
3. Target: `disallow_untyped_defs` first (highest value), then full strict

```toml
# Legacy module — not yet typed
[[tool.mypy.overrides]]
module = "myproject.legacy.*"
ignore_errors = true

# Partially fixed module
[[tool.mypy.overrides]]
module = "myproject.utils.*"
disallow_untyped_defs = true
```

### `# type: ignore` Policy

```python
# BAD — suppresses all errors on the line, invisible to future maintainers
result = legacy_fn()  # type: ignore

# GOOD — suppress only the specific error code, with a comment
result = legacy_fn()  # type: ignore[no-untyped-call]  # legacy_fn has no stubs

# GOOD — document why it's needed and when it can be removed
cast_value = cast(User, raw)  # type: ignore[misc]  # TODO: add stubs for external_lib
```

**Rules:**
- Always use `# type: ignore[error-code]` — never bare `# type: ignore`
- Add a comment explaining WHY and (when applicable) when it can be removed
- `warn_unused_ignores = true` (enabled in strict mode) will error when the ignore is no longer needed

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `error: Function is missing a return type annotation` | Missing `-> Type` | Add return annotation |
| `error: Returning Any from function declared to return "X"` | Returning untyped value | Narrow or cast at boundary |
| `error: Item "None" of "X \| None" has no attribute "y"` | Not narrowing Optional | Add `if x is None: return` guard |
| `error: Cannot determine type of "x"` | Variable defined in branch | Annotate at declaration: `x: int` |
| `error: Module has no attribute "x"` | Using private/missing attr | Check spelling; check stubs |
| `error: Incompatible types in assignment` | Type mismatch | Fix type or use correct type |
| `error: Need type annotation for "x"` | Empty collection | Annotate: `items: list[str] = []` |

---
