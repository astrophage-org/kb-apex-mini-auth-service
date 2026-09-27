<!-- anchor: src/main.py:L1-L100 sha:HEAD -->

# API Presentation Layer (`src/main.py`)

The `src/main.py` module acts as the primary HTTP entry point and presentation layer for the Mini Auth Service. Built on top of **FastAPI**, this module defines the public REST endpoints, handles request deserialization and response serialization, translates domain authentication outcomes into appropriate HTTP status codes, and applies security guards using dependency injection.

For the higher-level architectural context, see [[summaries/architecture-overview]] and the [[summaries/api-reference]].

---

## Responsibilities

1. **Application Initialization**: Instantiates and exposes the core `FastAPI` application instance (`app`) configured with metadata (title and version).
2. **HTTP Route Definition & Ingestion**: Exposes the authentication endpoints:
   - `POST /api/v1/auth/login`: Handles credential verification and token issuance.
   - `GET /api/v1/auth/me`: Exposes protected identity profile data.
3. **Request Body & Form Parsing**: Ingests standard `OAuth2PasswordRequestForm` payloads to support standard OAuth2 client login requests.
4. **HTTP Status & Exception Mapping**: Catches authentication failures and maps them to standard HTTP exceptions (e.g., raising `HTTPException(status_code=401, detail="Incorrect username or password")`).
5. **Route Guard Injection**: Leverages [[decisions/fastapi-dependency-injection]] via `Depends()` to bind the token verification workflow ([[concepts/token-verification-guard]]) directly to protected routes.

---

## Dependencies

### Internal Dependencies
- **[[entities/auth-domain]] (`src.auth`)**:
  - `authenticate_user`: Validates username and password credentials.
  - `create_access_token`: Generates signed JWT access tokens containing identity claims.
  - `verify_token`: Parses and validates incoming Bearer tokens for protected endpoints.
- **[[entities/configuration-settings]] (`src.config`)**:
  - `settings`: Singleton configuration instance providing runtime parameters.

### External Dependencies
- **`fastapi`**: Core web framework providing `FastAPI`, `Depends`, `HTTPException`, and `status`.
- **`fastapi.security`**: Provides `OAuth2PasswordRequestForm` for OAuth2-compliant form parsing.

---

## Endpoint Implementations

### 1. User Login (`POST /api/v1/auth/login`)

Implements the token issuance step of the [[concepts/jwt-authentication-flow]]. It accepts form-encoded user credentials, delegates verification to [[entities/auth-domain]], and returns a signed Bearer token.

```python
@app.post("/api/v1/auth/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
        )
    token = create_access_token(data={"sub": user["username"]})
    return {"access_token": token, "token_type": "bearer"}
```

- **Request Type**: `application/x-www-form-urlencoded` containing `username` and `password`.
- **Flow**:
  1. FastAPI parses form fields via `OAuth2PasswordRequestForm`.
  2. `authenticate_user()` evaluates credentials against the store.
  3. If invalid, raises `HTTP_401_UNAUTHORIZED`.
  4. If valid, `create_access_token()` issues a signed JWT with `sub` set to `user["username"]` using [[decisions/jwt-hs256-signing]].
- **Response Contract**: JSON object containing `access_token` and `token_type` (`bearer`).

---

### 2. User Profile (`GET /api/v1/auth/me`)

Implements a protected resource following the [[concepts/stateless-session-pattern]]. Access is regulated via the [[concepts/token-verification-guard]] passed as a FastAPI dependency.

```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    return {"username": token_data["sub"], "status": "active"}
```

- **Request Type**: HTTP `GET` with inbound token parameters.
- **Dependency Guard**: Injects `verify_token` to decode and validate the token signature and expiration against [[entities/configuration-settings]].
- **Response Contract**: Returns claims associated with the authenticated subject (`sub`).

---

## Architectural Considerations

- **Declarative Security**: By utilizing [[decisions/fastapi-dependency-injection]], the presentation layer avoids repetitive token parsing and cryptographic logic inside route handlers.
- **Stateless Operation**: Neither endpoint persists session state in memory, allowing horizontal scaling as detailed in [[concepts/stateless-session-pattern]].
- **Future Enhancements**: As outlined in [[summaries/roadmap-and-improvements]], the presentation layer is scheduled to adopt `OAuth2PasswordBearer` for standard HTTP `Authorization: Bearer <token>` header extraction and automated OpenAPI Swagger UI integration.