<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Configuration Settings

The `src/config.py` module defines the centralized runtime configuration and environment parameters for the service. It encapsulates cryptographic keys, signature algorithms, and persistent data store connection strings, exposing them as a singleton object for application-wide consumption.

For broader architectural context on how configuration integrates across the microservice, see the [[summaries/architecture-overview]] and the root [[index]].

---

## Responsibilities

1. **Centralized Configuration State**: Defines the static and runtime parameters required by the service in a unified schema (`Settings`).
2. **Cryptographic Key & Algorithm Provisioning**: Supplies the symmetric signing secret (`JWT_SECRET`) and hashing algorithm (`ALGORITHM`) consumed during token creation and validation in [[entities/auth-domain]].
3. **Infrastructure Connection Management**: Declares connection URIs such as `DATABASE_URL` for future persistent storage integration.
4. **Singleton Export**: Instantiates and exposes a single global configuration instance (`settings`) to ensure consistency across the application, adhering to the [[decisions/singleton-configuration]] architectural decision.

---

## Dependencies

### Upstream Dependencies (Internal & External)
- **Standard Python Library**: Does not depend on external third-party libraries; relies purely on standard Python object definitions.

### Downstream Consumers
- **[[entities/auth-domain]] (`src/auth.py`)**: Consumes `settings.JWT_SECRET` and `settings.ALGORITHM` in `create_access_token` and `verify_token` to perform symmetric HS256 encoding and decoding.
- **[[entities/presentation-main]] (`src/main.py`)**: Imports `settings` to coordinate application-level parameters and initialize FastAPI lifecycle dependencies.

```
       +--------------------+
       |   src/config.py    |
       |  (Settings class)  |
       +---------+----------+
                 |
        +--------+--------+
        |                 |
        v                 v
+---------------+ +---------------+
|  src/auth.py  | |  src/main.py  |
+---------------+ +---------------+
```

---

## Configuration Schema & Variables

The settings are encapsulated within the `Settings` class:

```python
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

### Attribute Details

| Attribute | Type | Default Value | Description | Relevant Patterns / Decisions |
| :--- | :--- | :--- | :--- | :--- |
| `JWT_SECRET` | `str` | `"astrophage-secret-key-123"` | Symmetric cryptographic secret key used to sign and verify JWT signatures. | [[decisions/jwt-hs256-signing]], [[concepts/jwt-authentication-flow]] |
| `ALGORITHM` | `str` | `"HS256"` | Cryptographic algorithm applied for token signing and decoding verification. | [[decisions/jwt-hs256-signing]], [[concepts/token-verification-guard]] |
| `DATABASE_URL` | `str` | `"postgresql://localhost:5432/auth_db"` | Relational database connection string for persistent user store. | [[summaries/roadmap-and-improvements]] |

---

## Architectural Role & Usage

### 1. Token Cryptography
The values defined in `Settings` are central to the [[concepts/stateless-session-pattern]]. When a client logs in via the API defined in [[entities/presentation-main]], [[entities/auth-domain]] uses `settings.JWT_SECRET` and `settings.ALGORITHM` to sign the payload:

```python
# src/auth.py
return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

During subsequent requests protected by [[concepts/token-verification-guard]], the same configuration ensures accurate signature validation.

### 2. Singleton Initialization Pattern
Rather than instantiating configuration objects within request lifecycles, `src/config.py` initializes a single instance (`settings = Settings()`). This ensures deterministic cryptographic operations and low-overhead access across all FastAPI routes. See [[decisions/singleton-configuration]] for architectural trade-offs.

---

## Evolution & Production Hardening

As outlined in [[summaries/roadmap-and-improvements]], the current hardcoded configuration pattern is targeted for modernization:
- **Dynamic Secret Loading**: Transitioning from static class attributes to `pydantic-settings` to dynamically parse environment variables and `.env` files.
- **Key Vault Integration**: Integrating external secret management systems (e.g., AWS Secrets Manager or HashiCorp Vault) to rotate `JWT_SECRET` in high-security production deployments.