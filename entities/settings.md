<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Settings

The `Settings` entity represents the centralized configuration layer of `mini-auth-service`, implemented in `src/config.py`. It encapsulates service-level parameters, cryptographic keys, algorithm choices, and infrastructure connection strings into a singleton instance.

```
+--------------------------------------------------------------------------+
|  Configuration Layer (`src/config.py`)                                   |
|  - Settings Singleton                                                    |
|    * JWT_SECRET: "astrophage-secret-key-123"                             |
|    * ALGORITHM: "HS256"                                                  |
|    * DATABASE_URL: "postgresql://localhost:5432/auth_db"                 |
+-------------------+----------------------------------+-------------------+
                    |                                  |
                    v                                  v
+------------------------------------+   +---------------------------------+
| Security Layer (`src/auth.py`)     |   | API Layer (`src/main.py`)       |
| - [[entities/auth-engine]]         |   | - [[entities/api-router]]       |
+------------------------------------+   +---------------------------------+
```

---

## Responsibilities

The primary responsibilities of the `Settings` class and the exported `settings` singleton are:

1. **Cryptographic Parameter Management**:
   * Holds the symmetric signing key (`JWT_SECRET`) required by [[concepts/jwt-signing-verification]] for HMAC-SHA256 signing and verification.
   * Defines the active token algorithm (`ALGORITHM = "HS256"`).
2. **Infrastructure Endpoints**:
   * Specifies data persistence endpoints such as `DATABASE_URL` for downstream components.
3. **Single Source of Truth**:
   * Provides a unified, immutable configuration singleton (`settings`) imported across both the security/domain layer ([[entities/auth-engine]]) and the API router layer ([[entities/api-router]]).

---

## Configuration Properties

| Attribute | Type | Current Value | Description |
| :--- | :--- | :--- | :--- |
| `JWT_SECRET` | `str` | `"astrophage-secret-key-123"` | Symmetric secret key passed to PyJWT for encoding and decoding [[entities/jwt-token-payload]] instances. |
| `ALGORITHM` | `str` | `"HS256"` | Specifies the symmetric algorithm used for [[concepts/jwt-signing-verification]]. |
| `DATABASE_URL` | `str` | `"postgresql://localhost:5432/auth_db"` | Connection string reserved for database-backed user identity storage and hashing. |

---

## Implementation Details

The configuration is declared as a plain Python class and instantiated as a singleton at module import time in `src/config.py`:

```python
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
    DATABASE_URL: str = "postgresql://localhost:5432/auth_db"

settings = Settings()
```

---

## Dependencies

### Inbound / Consumed By
* **[[entities/auth-engine]] (`src/auth.py`)**: Consumes `settings.JWT_SECRET` and `settings.ALGORITHM` in `create_access_token()` to sign JWTs and in `verify_token()` to validate incoming bearer tokens.
* **[[entities/api-router]] (`src/main.py`)**: Imports `settings` during application initialization and routing context setup.

### Outbound / Upstream
* Standard Python runtime (no external package dependencies in the minimal implementation).

---

## Architectural Evolution & Future Considerations

* **Environment Variable Binding**: The current static class model requires manual source edits to change parameters. The roadmap outlined in [[decisions/environment-configuration-management]] details transitioning to `pydantic-settings` (`BaseSettings`) for 12-factor application compliance.
* **Key Separation for Asymmetric Signatures**: If migrating from symmetric HS256 to asymmetric RS256/EdDSA (see [[decisions/symmetric-vs-asymmetric-signing]]), `Settings` will be expanded to manage distinct `JWT_PRIVATE_KEY` and `JWT_PUBLIC_KEY` values.
* **Database Activation**: `DATABASE_URL` is currently provisioned as a static placeholder. As described in [[decisions/credential-storage-hashing]], this will be integrated with an ORM (e.g., SQLAlchemy/Tortoise) to replace in-memory credential checks.