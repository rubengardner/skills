## SPECIALIST B: Tooling

### The Stack (Non-Negotiable)

| Tool | Role | Replaces |
|------|------|---------|
| `uv` | Package manager, venv, Python versions | pip, pipenv, pyenv |
| `ruff` | Linter + formatter | black, isort, flake8, pyupgrade |
| `mypy` | Static type checker | — |
| `pytest` | Test runner | unittest |
| `pytest-cov` | Coverage | — |
| `pre-commit` | Git hook automation | manual enforcement |

### Ruff Rule Groups — What to Enable

| Group | Code | Enable? | Notes |
|-------|------|---------|-------|
| pycodestyle | E, W | Always | Baseline PEP 8 |
| pyflakes | F | Always | Undefined names, unused imports |
| isort | I | Always | Import order |
| pep8-naming | N | Always | Naming violations |
| bugbear | B | Always | Common mistakes, mutable defaults |
| pyupgrade | UP | Always | Modernizes syntax for target Python |
| ruff-specific | RUF | Always | High signal-to-noise |
| simplify | SIM | Always | Reduces redundant constructs |
| mccabe complexity | C90 | Always | Set max-complexity = 10 |
| annotations | ANN | Yes* | *Disable ANN101, ANN102 (self/cls) |
| security/bandit | S | Yes | Disable S101 in tests |
| use-pathlib | PTH | Yes | os.path → pathlib |
| eradicate | ERA | CI only | Commented-out code; noisy locally |
| docstrings | D | No | Too noisy; configure separately if needed |
| pylint | PL* | Selective | Overlap with other rules |

### mypy Configuration

```toml
[tool.mypy]
python_version = "3.11"
strict = true
files = ["src"]

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

**mypy vs pyright:** Use `mypy --strict` as default. Add `pyright` in `basic` mode if the team is VS Code-heavy. Do not run both in `strict` mode simultaneously — they disagree on edge cases.

### pre-commit Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: debug-statements
      - id: check-added-large-files
        args: ["--maxkb=500"]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        args: [--strict]
```

Install:
```bash
uv add --dev pre-commit
pre-commit install
pre-commit run --all-files
```

### CI Pipeline (GitHub Actions minimum)

```yaml
- uses: astral-sh/setup-uv@v3
- run: uv sync --dev
- run: uv run ruff check src tests
- run: uv run ruff format --check src tests
- run: uv run mypy src
- run: uv run pytest --cov=src --cov-report=xml
```

---
