# Architectural Decision Record: Environment Configuration Management and Migration to `pydantic-settings`

## Status

**Accepted** (Targeted for Next Major Release)

---

## Context

The configuration management in `mini-auth-service` is currently implemented via a basic Python class in `src/config.py` (`[[entities/settings]]`):

```python
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

While this singleton abstraction decouples configuration access across [[entities/auth-engine]] and [[entities/api-router]], it has critical operational and security shortcomings:

1. **Security Vulnerability**: Sensitive secrets (such as `JWT_SECRET`) and infrastructure connection strings (`DATABASE_URL`) are hardcoded directly in the source code.
2. **Lack of Environment Injection**: The current class does not automatically read or override values from operating system environment variables or `.env` files, violating Principle III ("Config") of the **Twelve-Factor App** methodology.
3. **No Type or Value Validation**: The settings object does not validate incoming configuration types, URLs, or cryptographic algorithm compatibility at application startup.
4. **Impending Architecture Needs**:
   - The roadmap for [[decisions/credential-storage-hashing]] requires strict, validated database connection strings (`PostgresDsn`).
   - The evaluation in [[decisions/symmetric-vs-asymmetric-signing]] will require dynamic configuration of private/public key pairs, key paths, and algorithm strings (`RS256` / `EdDSA`).

---

## Decision

We will migrate the configuration layer from the vanilla Python class to **`pydantic-settings`** (built on Pydantic v2).

```
+-------------------------------------------------------------------------------+
|                             Operating Environment                             |
|  (Process Env Vars: JWT_SECRET, DATABASE_URL, ALGORITHM) / (.env local file) |
+---------------------------------------+---------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
|                   Pydantic BaseSettings Validation Layer                     |
|  - Type coercion (str -> PostgresDsn / SecretStr / int)                       |
|  - Validation rules & default fallbacks                                       |
|  - Fail-fast exception handling on startup                                   |
+---------------------------------------+---------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
|                       Settings Singleton (`settings`)                        |
|  - Immutable, strongly-typed configuration instance                           |
|  - Injected across [[entities/auth-engine]] & [[entities/api-router]]         |
+-------------------------------------------------------------------------------+
```

### Key Architectural Guidelines

1. **Inheritance from `BaseSettings`**:
   The `Settings` model will extend `pydantic_settings.BaseSettings`, automatically resolving values from OS environment variables (case-insensitive by default).

2. **Secure Types & Validation**:
   - Secrets must use `pydantic.SecretStr` to prevent accidental logging or exposure in tracebacks.
   - URIs (such as `DATABASE_URL`) will use Pydantic's specialized URL types (e.g., `PostgresDsn` or `AnyUrl`).
   - Algorithm choices will be constrained using `Literal` types or Enums (e.g., `Literal["HS256", "RS256", "EdDSA"]`).

3. **Fail-Fast Initialization**:
   Missing required secrets in production (such as a missing `JWT_SECRET`) will raise a `ValidationError` at startup, immediately halting service execution rather than failing during request handling.

4. **Preserve Singleton Module Interface**:
   To minimize refactoring across [[entities/auth-engine]] and [[entities/api-router]], `src/config.py` will continue to export an initialized instance `settings = Settings()`.

---

## Proposed Implementation

```python
# src/config.py (Target State)
from typing import Literal
from pydantic import PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # Cryptographic Configuration
    JWT_SECRET: SecretStr
    ALGORITHM: Literal["HS256", "RS256", "EdDSA"] = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15

    # Storage Configuration (Roadmap)
    DATABASE_URL: PostgresDsn = "postgresql://localhost:5432/auth_db"

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

settings = Settings()
```

### Impact on Downstream Modules

When accessing `JWT_SECRET` in `src/auth.py` (`[[concepts/jwt-signing-verification]]`), secrets wrapped in `SecretStr` will be resolved explicitly:

```python
# Updated token signing
encoded_jwt = jwt.encode(
    to_encode, 
    settings.JWT_SECRET.get_secret_value(), 
    algorithm=settings.ALGORITHM
)
```

---

## Consequences

### Positive
- **Environment Parity**: Seamless transition between local development (`.env`), staging, and production environments via container environment variables.
- **Fail-Fast Reliability**: Malformed URLs or missing cryptographic keys trigger validation failures at startup rather than during live HTTP authentication requests.
- **Enhanced Security**: Elimination of checked-in default secrets in version control; `SecretStr` masks values in string representations.
- **Readiness for Future ADRs**: Sets up reliable configuration structures for [[decisions/credential-storage-hashing]] (database pooling) and [[decisions/symmetric-vs-asymmetric-signing]] (asymmetric key paths).

### Negative / Trade-offs
- **New External Dependency**: Introduces `pydantic-settings` to project requirements.
- **Minor Code Adjustments**: Consuming components in [[entities/auth-engine]] must call `.get_secret_value()` when passing secrets to `PyJWT`.
- **Environment Setup Requirement**: Local test and CI pipelines must explicitly supply configuration variables or a `.env` fixture.

---

## Related References

- [[index]] - Microservice system overview
- [[summaries/architecture-overview]] - Architectural layering and configuration dependencies
- [[entities/settings]] - Configuration entity specification
- [[concepts/jwt-signing-verification]] - Token signing algorithms and secret consumption
- [[decisions/credential-storage-hashing]] - Database configuration requirements
- [[decisions/symmetric-vs-asymmetric-signing]] - Asymmetric key configuration roadmap