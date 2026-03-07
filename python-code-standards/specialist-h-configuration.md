## SPECIALIST H: Configuration & Secrets

### Cardinal Rule

Never hardcode configuration or secrets in source code. This includes passwords, API keys, database URLs, feature flags, environment-specific URLs. Ruff S105/S106 flags hardcoded passwords.

### pydantic-settings Pattern

```python
# src/myproject/config.py
from functools import lru_cache
from pydantic import Field, SecretStr, PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_nested_delimiter="__",
        case_sensitive=False,
    )

    debug: bool = False
    log_level: str = "INFO"
    database_url: PostgresDsn           # required — fails loudly if missing
    api_key: SecretStr                  # SecretStr prevents logging leaks
    jwt_secret: SecretStr = Field(min_length=32)
    max_retries: int = Field(default=3, ge=1, le=10)


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

**Rules:**
- Use `SecretStr` for all credentials
- Fail loudly on startup if required config is missing — never default to insecure values
- `@lru_cache` ensures `.env` is read once per process
- In tests: inject `Settings(...)` directly, never read `.env`

### .env Files

```bash
# .env  → .gitignore (NEVER commit)
DATABASE_URL=postgresql://localhost:5432/myproject_dev
API_KEY=dev-key-not-real

# .env.example  → commit this, no real values
DATABASE_URL=postgresql://host:5432/dbname
API_KEY=your-api-key-here
JWT_SECRET=generate-with: openssl rand -hex 32
```

**Priority order (highest → lowest):** `Settings(kwarg)` > env var > `.env` file > model default.

---
