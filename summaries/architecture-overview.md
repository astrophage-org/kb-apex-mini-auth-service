<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Architecture Overview

`mini-auth-service` is a stateless, lightweight identity and token management service implemented with **FastAPI** and **PyJWT**. The system provides identity verification, cryptographic JSON Web Token ([[entities/jwt-token-payload|JWT]]) generation, and token validation for downstream services.

---

## 1. High-Level Architecture & Layered Design

The codebase adheres to a clean layered architecture separating HTTP routing, core authentication domain logic, and configuration management:

```
                  +-----------------------------------+
                  |           Client / API            |
                  +-----------------+-----------------+
                                    |
                        HTTP POST / GET Requests
                                    |
                                    v
+--------------------------------------------------------------------------+
|  API Layer (`src/main.py` / [[entities/api-router]])                    |
|  - REST routing & HTTP status mapping                                   |
|  - Dependency Injection (OAuth2PasswordRequestForm, verify_token)       |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+--------------------------------------------------------------------------+
|  Security & Domain Layer (`src/auth.py` / [[entities/auth-engine]])     |
|  - Credential verification & user lookup                                 |
|  - JWT issuance & HMAC-SHA256 signing                                    |
|  - Token decoding, signature verification, and expiration validation     |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+--------------------------------------------------------------------------+
|  Configuration Layer (`src/config.py` / [[entities/settings]])          |
|  - Centralized settings singleton (Secrets, Algorithm, Database URI)     |
+--------------------------------------------------------------------------+
```

### Layer Responsibilities

| Layer / Module | Entity | Responsibility | Primary Dependencies |
| :--- | :--- | :--- | :--- |
| **API Layer**<br>`src/main.py` | [[entities/api-router]] | Exposes REST endpoints defined in [[summaries/api-endpoints]], binds request data via [[concepts/dependency-injection]], and maps domain results/exceptions to HTTP status codes (`200 OK`, `401 Unauthorized`). | Calls [[entities/auth-engine]] functions; consumes [[entities/settings]]. |
| **Security & Domain Layer**<br>`src/auth.py` | [[entities/auth-engine]] | Encapsulates authentication logic, user lookup, token generation with expiration (`exp`), and cryptographic decoding. | Consumes settings from [[entities/settings]]; uses `PyJWT`. |
| **Configuration Layer**<br>`src/config.py` | [[entities/settings]] | Houses service settings, cryptographic secrets, and connection parameters as a centralized singleton. | Standard library defaults. |

---

## 2. Core Workflows and Data Flow

### A. Authentication & Token Issuance

The login workflow verifies credentials and produces a signed Bearer token:

```
Client                    API Router (main.py)           Auth Engine (auth.py)
  |                                |                               |
  |--- 1. POST /auth/login ------->|                               |
  |    (form data: user/pass)      |--- 2. authenticate_user() --->|
  |                                |<-- 3. user record / None -----|
  |                                |                               |
  |                                |--- 4. create_access_token() ->|
  |                                |<-- 5. signed JWT string ------|
  |<-- 6. 200 OK (access_token) ---|                               |
```

1. **Request Intake**: The client submits credentials using `application/x-www-form-urlencoded` format, parsed using `OAuth2PasswordRequestForm` via [[concepts/dependency-injection]].
2. **Credential Validation**: The route handler invokes `authenticate_user()` to validate the username and password against the identity store (detailed in [[concepts/credential-validation-flow]]).
3. **Token Creation**: `create_access_token()` builds the [[entities/jwt-token-payload]] with subject (`sub`) and expiration (`exp` = 15 minutes) claims, signing it using `HS256` via `settings.JWT_SECRET`.
4. **Response**: A JSON response containing `access_token` and `token_type` is returned to the client.

### B. Authenticated Request & Token Verification

The `/me` endpoint demonstrates protected route access:

1. **Request Intake**: The client calls `/api/v1/auth/me`.
2. **Dependency Resolution**: FastAPI resolves the `verify_token` dependency via `Depends(verify_token)`.
3. **Cryptographic Validation**: `verify_token()` performs signature verification and expiration checks via [[concepts/jwt-signing-verification]].
4. **Context Injection**: The decoded payload is injected into the endpoint handler to return user metadata.

---

## 3. Key Design Patterns & System Principles

* **Stateless Token-Based Authentication**: The system operates without server-side session stores, embedding identity context directly within signed tokens (see [[concepts/token-based-authentication]] and [[decisions/stateless-vs-stateful-sessions]]).
* **FastAPI Dependency Injection (DI)**: Declarative request extraction and validation are enforced using `Depends()` for request form parsing and token verification pipelines (see [[concepts/dependency-injection]]).
* **Singleton Configuration**: Runtime secrets and algorithmic parameters are managed globally via a single `Settings` instance in [[entities/settings]].

---

## 4. Production Architectural Considerations

The architecture is designed to evolve across several key dimensions:

* **Symmetric vs. Asymmetric Signing**: Transitioning from symmetric `HS256` to asymmetric algorithms (`RS256` or `EdDSA`) will allow downstream services to verify tokens using a public key without sharing the secret key (see [[decisions/symmetric-vs-asymmetric-signing]]).
* **Credential Persistence & Hashing**: Upgrading from hardcoded credentials to database persistence via `DATABASE_URL` with secure hashing algorithms such as Argon2 or bcrypt (see [[decisions/credential-storage-hashing]]).
* **Environment-Based Configuration**: Enhancing [[entities/settings]] to subclass `pydantic-settings` (`BaseSettings`) for environment variable validation (see [[decisions/environment-configuration-management]]).
* **Header-Based Token Extraction**: Updating `verify_token` to utilize `fastapi.security.OAuth2PasswordBearer` to extract tokens directly from the HTTP `Authorization: Bearer <token>` header.