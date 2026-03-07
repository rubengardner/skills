## SPECIALIST A: Project Layout

### Rule Set

1. **Use the `src` layout** for any project that will be installed, packaged, or tested with pytest.
   ```
   myproject/
   ├── pyproject.toml
   ├── uv.lock
   ├── .python-version       # pin Python version
   ├── .env.example          # committed, no real secrets
   ├── .pre-commit-config.yaml
   ├── README.md
   ├── src/
   │   └── myproject/
   │       ├── __init__.py
   │       ├── py.typed      # PEP 561 — mandatory for typed libraries
   │       └── ...
   ├── tests/
   │   ├── conftest.py
   │   └── ...
   └── docs/
   ```

2. **Flat layout** is only acceptable for: single-file scripts, notebooks, throwaway tools not installable via pip.

3. **Package manager: `uv`** — no pip-only, no poetry for new projects.
   ```bash
   uv init --package myproject   # creates src layout
   uv add pydantic httpx         # runtime deps
   uv add --dev pytest ruff mypy # dev deps
   uv sync                       # install from lockfile
   uv run pytest                 # run inside venv
   ```

4. **`pyproject.toml` is the only config file.** Delete `setup.py`, `setup.cfg`, `.flake8`, `mypy.ini`, `.isort.cfg` if present.

5. **Commit `uv.lock`** for applications. For distributed libraries, commit it for CI reproducibility but do not publish it to PyPI.

6. **Monorepo → uv workspaces:**
   ```toml
   # root pyproject.toml
   [tool.uv.workspace]
   members = ["packages/*", "services/*"]
   ```
   Each member has its own `pyproject.toml` + `src/`. Inter-package deps use `workspace = true`.

### Minimal `pyproject.toml` Template

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.9", "mypy>=1.10"]

[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
addopts = "-ra --strict-markers"

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E","F","I","N","B","UP","C90","SIM","RUF","S","ANN","PTH"]
ignore = ["ANN101","ANN102"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101","ANN"]

[tool.ruff.lint.mccabe]
max-complexity = 10

[tool.mypy]
python_version = "3.11"
strict = true
files = ["src"]
```

---
