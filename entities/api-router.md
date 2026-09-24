<!-- anchor: src/main.py:L1-L100 sha:HEAD -->

The **API Router** (`src/main.py`) serves as the HTTP transport and routing layer of the `mini-auth-service`. Built using FastAPI, it defines the external REST interface, coordinates request parsing via [[concepts/dependency-injection]], interacts with the [[entities/auth-engine]], and maps domain layer outcomes to standardized HTTP response structures and status codes.

---

## Responsibilities

* **HTTP Route Declaration & Transport Handling**: Exposes API endpoints under the `/api/v1/auth/` prefix for client authentication and identity lookup (detailed in [[summaries/api-endpoints]]).
* **Request Dependency Binding**: Uses FastAPI's declarative [[concepts/dependency-injection]] system to bind incoming request parameters, specifically:
  * Form-encoded credential extraction using `OAuth2PasswordRequestForm`.
  * Contextual token decoding and verification using the `verify_token` dependency.
* **Domain Layer Orchestration**: Acts as the interface between HTTP clients and domain logic in [[entities/auth-engine]], calling `authenticate_user`, `create_access_token`, and `verify_token`.
* **Error Translation & HTTP Status Mapping**: Translates missing/invalid credentials and domain-level authorization failures into standard HTTP errors (such as `401 Unauthorized` with `HTTPException`).
* **Response Serialization**: Constructs JSON response payloads conforming to OAuth2 Bearer token specs and [[entities/jwt-token-payload]] schemas.

---

## Dependencies

* **[[entities/auth-engine]] (`src/auth.py`)**: Consumes core cryptographic and authentication functions (`authenticate_user`, `create_access_token`, and `verify_token`).
* **[[entities/settings]] (`src/config.py`)**: Imports centralized application runtime settings (`settings`).
* **`fastapi` / `fastapi.security`**: Provides routing framework classes (`FastAPI`, `HTTPException`, `status`), dependency injection primitives (`Depends`), and request parsing utilities (`OAuth2PasswordRequestForm`).

---

## Endpoints Implementation

### 1. Token Issuance (`POST /api/v1/auth/login`)

Handles user authentication and JWT generation following the [[concepts/credential-validation-flow]].

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

* **Transport Parsing**: `form_data` is extracted from `application/x-www-form-urlencoded` payloads.
* **Authentication**: Delegates credential verification to `authenticate_user` in [[entities/auth-engine]].
* **Failure Handling**: If the user is invalid, returns HTTP `401 Unauthorized`.
* **Token Creation**: On success, calls `create_access_token` to sign a JWT with the `sub` claim set to the username, returning a standard Bearer response.

---

### 2. Current User Identification (`GET /api/v1/auth/me`)

Provides authenticated user identity validation via stateless JWT verification.

```python
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    return {"username": token_data["sub"], "status": "active"}
```

* **Dependency Resolution**: Leverages `Depends(verify_token)` to decode and validate the cryptographic token (see [[concepts/jwt-signing-verification]]).
* **Identity Context**: Extracts the subject (`sub`) claim from the decoded token payload and returns the active user status.

---

## Architectural Considerations

* **Stateless Operation**: The routing layer maintains zero in-memory session state, adhering to [[decisions/stateless-vs-stateful-sessions]].
* **Separation of Concerns**: Transport mechanics (status codes, headers, HTTP forms) are isolated within `main.py`, while cryptographic verification resides in [[entities/auth-engine]].
* **Production Improvements**:
  * **Header Extraction via OAuth2 Scheme**: Currently, `verify_token` receives the raw token directly as a dependency parameter. Integrating `fastapi.security.OAuth2PasswordBearer` will allow automatic extraction from `Authorization: Bearer <token>` headers.
  * **Validation Guards**: Adding an explicit `HTTPException(401)` check within `get_current_user` or the `verify_token` dependency if the payload evaluates to `None`.