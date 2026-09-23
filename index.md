# Architectural Summary: Mini AuthService (`mini-auth-service`)

`mini-auth-service` is a stateless, lightweight identity and token management microservice built using **FastAPI** and **PyJWT**. The service provides identity verification, cryptographic JSON Web Token (JWT) issuance, and token validation for downstream consumers.

---

## 1. High-Level Architecture & Layered Structure

The service follows a layered architectural pattern separating network transport/routing, core cryptographic business logic, and configuration management.

```
                  +-----------------------------------+
                  |           Client / API            |
                  +-----------------+-----------------+
                                    |
                        HTTP POST / GET Requests
                                    |
                                    v
+--------------------------------------------------------------------------+
|  API Layer (`src/main.py`)                                               |
|  - REST routing & HTTP error handling                                    |
|  - Dependency Injection (OAuth2PasswordRequestForm, Token resolution)    |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+--------------------------------------------------------------------------+
|  Security & Domain Layer (`src/auth.py`)                                 |
|  - Credential verification & user lookup                                 |
|  - JWT generation & symmetric signing (HMAC-SHA256)                      |
|  - Token decoding, signature verification, and expiration checks         |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
+--------------------------------------------------------------------------+
|  Configuration Layer (`src/config.py`)                                   |
|  - Centralized settings singleton (Secrets, Algorithm, Database URI)     |
+--------------------------------------------------------------------------+
```

### Component Breakdown

| Layer / File | Responsibility | Key Interactions |
| :--- | :--- | :--- |
| **API Layer**<br>`src/main.py` | Exposes REST endpoints under `/api/v1/auth/*`, binds request parameters via Dependency Injection, maps domain results/exceptions to HTTP status codes (e.g., `401 Unauthorized`). | Invokes `src/auth.py` functions via endpoint handlers and FastAPI dependencies. |
| **Security & Domain Layer**<br>`src/auth.py` | Encapsulates authentication logic, user verification, JWT signing with expiration claims (`exp`), and token payload extraction. | Consumes configuration parameters from `src/config.py`. |
| **Configuration Layer**<br>`src/config.py` | Manages runtime configuration, cryptographic parameters, and secrets via an instantiated `Settings` singleton. | Injected into `src/auth.py` and across the application lifecycle. |

---

## 2. Core Workflows & Data Flows

### A. Authentication & Token Issuance (`POST /api/v1/auth/login`)

The authentication lifecycle handles user credentials and produces a signed Bearer token:

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

1. **Request Reception**: The client sends credentials via `application/x-www-form-urlencoded` format, parsed using FastAPI’s `OAuth2PasswordRequestForm`.
2. **Credential Validation**: `login()` delegates to `authenticate_user()`.
3. **Token Creation**: Upon successful authentication, `create_access_token()` constructs a JWT payload with subject (`sub`) and expiration (`exp` = current time + 15 minutes), signing it with `HS256` using `settings.JWT_SECRET`.
4. **Response**: Returns a JSON Bearer token payload: `{"access_token": "<jwt>", "token_type": "bearer"}`.

---

### B. Authenticated Route & Token Verification (`GET /api/v1/auth/me`)

Token verification validates identity claims on protected routes:

1. **Request Reception**: The client calls `/api/v1/auth/me`.
2. **Dependency Resolution**: FastAPI resolves the `verify_token` dependency via `Depends()`.
3. **Cryptographic Validation**: `verify_token()` verifies signature integrity and checks expiration against `settings.JWT_SECRET` and `settings.ALGORITHM`.
4. **Context Injection**: The verified payload is passed to the route handler, which returns user context: `{"username": payload["sub"], "status": "active"}`.

---

## 3. Key Design Patterns & Abstractions

* **Stateless Token-Based Authentication**: Authentication state is encapsulated entirely within self-contained JWTs containing subject (`sub`) and expiration (`exp`) claims, eliminating server-side session persistence.
* **FastAPI Dependency Injection (DI)**: Declarative request extraction and validation are enforced through FastAPI's `Depends` system for both form payloads (`OAuth2PasswordRequestForm`) and authentication validation pipelines (`verify_token`).
* **Singleton Configuration**: Static application settings and cryptographic parameters are centralized in a single `Settings` instance in `src/config.py`.

---

## 4. Production Readiness & Architectural Considerations

1. **Token Transport via Header**: `verify_token` currently receives tokens directly as a parameter. In production, this should integrate `fastapi.security.OAuth2PasswordBearer` to automatically extract and validate Bearer tokens from the HTTP `Authorization: Bearer <token>` header.
2. **Credential Storage & Password Hashing**: The current implementation uses in-memory credentials. Production deployment requires database-backed identity persistence using `DATABASE_URL` combined with secure hashing (e.g., Argon2, bcrypt) via Passlib.
3. **Asymmetric Cryptography for Microservices**: The service currently uses symmetric signing (`HS256`), requiring token-verifying services to share the secret. Migrating to asymmetric cryptography (`RS256` or `EdDSA`) allows `mini-auth-service` to sign tokens using a private key while external services verify tokens using a public key.
4. **Environment-Based Configuration**: The `Settings` model should leverage `pydantic-settings` (`BaseSettings`) to support dynamic environment variable overrides rather than static in-code defaults.