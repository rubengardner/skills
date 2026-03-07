## SPECIALIST G: Settings

### Use `pydantic-settings` — Not `django-environ`

`pydantic-settings` provides full type validation, `SecretStr`, and fail-fast startup errors. `django-environ` lacks runtime type safety.

```toml
# pyproject.toml
dependencies = [
    "pydantic-settings>=2.0",
    "dj-database-url>=2.0",
]
```

```python
# config/settings/base.py
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict
import dj_database_url


class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # Required — no default; startup fails immediately if missing
    SECRET_KEY: SecretStr
    DATABASE_URL: str

    # Optional with safe defaults
    DEBUG: bool = False
    ALLOWED_HOSTS: list[str] = Field(default_factory=list)
    CORS_ALLOWED_ORIGINS: list[str] = Field(default_factory=list)
    REDIS_URL: str = "redis://localhost:6379/0"
    LOG_LEVEL: str = "WARNING"


_settings = AppSettings()

# Expose as module-level names Django expects
SECRET_KEY: str = _settings.SECRET_KEY.get_secret_value()
DEBUG: bool = _settings.DEBUG
ALLOWED_HOSTS: list[str] = _settings.ALLOWED_HOSTS
DATABASES: dict = {
    "default": dj_database_url.parse(
        _settings.DATABASE_URL,
        conn_max_age=600,
        conn_health_checks=True,
    )
}
```

**Rules:**
- `SecretStr` for `SECRET_KEY`, tokens, passwords — prevents accidental logging
- `DATABASE_URL` as a single 12-factor env var parsed by `dj-database-url`
- Missing required fields raise `ValidationError` at process startup — never at first use
- `.env` in `.gitignore`; `.env.example` committed with placeholder values

### Settings Split

```python
# config/settings/local.py
from .base import *  # noqa: F401, F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]
INSTALLED_APPS += ["debug_toolbar", "nplusone.ext.django"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
NPLUSONE_RAISE = True
```

```python
# config/settings/production.py
from .base import *  # noqa: F401, F403

SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

Set `DJANGO_SETTINGS_MODULE` per environment:
- Dev: `config.settings.local`
- Prod: `config.settings.production`
- CI: `config.settings.local` (or a dedicated `test.py`)

---
