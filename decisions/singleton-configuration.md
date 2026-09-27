# ADR: Centralized Singleton Pattern for Application Settings and Constants

## Status
Accepted

## Context
The **Mini Auth Service** requires consistent access to core operational parameters, cryptographic secrets, signing algorithms, and infrastructure endpoints across multiple architectural boundaries:
- In the domain layer ([[entities/auth-domain]]), token creation (`create_access_token`) and verification (`verify_token`) require access to `JWT_SECRET` and `ALGORITHM` to guarantee symmetric signature validity (see [[decisions/jwt-hs256-signing]]).
- In the presentation layer ([[entities/presentation-main]]), routes and guards depend on consistent application state.
- In future persistence components, database connectors will require access to `DATABASE_URL`.

Without a centralized configuration strategy, application parameters risk being duplicated across modules or repeatedly re-parsed across HTTP request lifecycles, leading to configuration drift, security vulnerabilities, or performance degradation. The application needed a lightweight, centralized mechanism to manage these settings across [[summaries/architecture-overview]].

## Decision
We adopted a centralized **Singleton Pattern** implemented in [[entities/configuration-settings]] (`src/config.py`). 

The configuration is structured as follows:
1. Define a `Settings` class declaring all required configuration constants (`JWT_SECRET`, `ALGORITHM`, `DATABASE_URL`).
2. Instantiate a single, module-level instance `settings = Settings()` directly within `src/config.py`.
3. Export and import this `settings` singleton across dependent modules (e.g., `from .config import settings` in [[entities/auth-domain]] and [[entities/presentation-main]]).

```python
# src/config.py
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

## Consequences

### Positive
- **Single Source of Truth**: All cryptographic signing and validation routines reference identical key material and algorithm configurations, preventing signature mismatch bugs during [[concepts/jwt-authentication-flow]] and [[concepts/token-verification-guard]].
- **Zero Overhead**: Settings are instantiated once upon module load and reused throughout the process lifetime, avoiding per-request instantiation overhead.
- **Clean Dependency Graph**: Consumers simply import `settings` without needing complex factory patterns or boilerplate dependency injection for basic static parameters.

### Negative / Trade-offs
- **Static Key Management**: The initial implementation hardcodes secret strings directly into the class attributes rather than reading from runtime environment variables or an external secrets vault.
- **Testing Constraints**: Directly importing a global singleton can complicate unit testing environments where overriding configuration per test runner is required.

## Future Evolution
As outlined in [[summaries/roadmap-and-improvements]], the `Settings` class will be migrated to `pydantic-settings` (`BaseSettings`). This evolution will preserve the singleton pattern while adding dynamic environment variable ingestion (`.env`), type validation, and dynamic key rotation capabilities.

## Related Documents
- [[entities/configuration-settings]] — Configuration module implementation
- [[entities/auth-domain]] — Authentication domain consuming configuration
- [[decisions/jwt-hs256-signing]] — Architectural decision on symmetric JWT signing
- [[summaries/roadmap-and-improvements]] — Roadmap for environment-based secrets management
- [[index]] — System index and high-level overview