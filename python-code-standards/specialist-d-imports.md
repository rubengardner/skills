## SPECIALIST D: Imports

### The Three Groups (enforced by ruff `I`)

```python
# 1. Standard library
import os
import sys
from pathlib import Path
from typing import TYPE_CHECKING

# 2. Third-party
import httpx
from pydantic import BaseModel

# 3. Local / first-party
from myproject.core import UserService
from myproject.models import User
```

One blank line between groups. Ruff enforces this automatically with `known-first-party` configured.

### Absolute vs Relative

- **Default: absolute imports.** Unambiguous regardless of invocation context.
- **Relative imports acceptable** only within a tightly-coupled subpackage to avoid repetition.
- **Never** use implicit relative imports (Python 2 legacy — `SyntaxError` in Python 3).
- **Never** use `from module import *` in production code.

### TYPE_CHECKING Guard

```python
from __future__ import annotations  # required in 3.10/3.11; enables deferred evaluation
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence
    from myproject.heavy_module import ExpensiveClass
```

**Rules:**
- Always pair `TYPE_CHECKING` with `from __future__ import annotations`
- Do not put imports in `TYPE_CHECKING` if needed at runtime (e.g., for `isinstance`)
- In Python 3.12+, use `type` statement for aliases instead of `TypeAlias`
- All `TYPE_CHECKING` imports must still be real, resolvable modules — type checkers validate them

### Define `__all__` in Public Modules

```python
__all__ = ["UserService", "UserNotFoundError"]
```

Required when the module is part of a public API. Enables `--no-implicit-reexport` in mypy.

### Anti-patterns (all flagged by ruff)

```python
from mymodule import *           # never — wildcard import
def my_func():
    import os                    # avoid — function-scope import except for genuine circular deps
import numpy as numpy            # pointless alias
list = [1, 2, 3]                 # shadows builtin
```

---
