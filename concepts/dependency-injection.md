# Dependency Injection Pattern

In `mini-auth-service`, **FastAPI Dependency Injection (DI)** is the architectural backbone used to decouple HTTP transport logic from authentication rules, request parsing, and token verification. By utilizing FastAPI's `Depends` mechanism, the API routing layer ([[entities/api-router]]) declaratively extracts and validates request data before delegating processing to route handlers.

---

## 1. Overview of Dependency Injection in the Service

FastAPI provides a hierarchical dependency injection system that resolves dependencies prior to executing the endpoint logic. In this microservice, DI is utilized for two primary functions:

1. **Request Body Parsing**: Extracting and validating URL-encoded credentials (`OAuth2PasswordRequestForm`).
2. **Security & Claim Extraction**: Intercepting incoming requests to verify JSON Web Tokens and supplying the decoded payload (`verify_token`) to protected endpoints.

```
                    Incoming HTTP Request
                              |
                              v
                +----------------------------+
                |  FastAPI DI Engine         |
                |  (Depends resolution)      |
                +--------------+-------------+
                               |
            +------------------+------------------+
            |                                     |
            v                                     v
   [ Form Parsing DI ]                  [ Token Verification DI ]
OAuth2PasswordRequestForm                     verify_token()
  (POST /api/v1/auth/login)              (GET /api/v1/auth/me)
            |                                     |
            v                                     v
+-----------------------+             +-----------------------+
|  login() Handler      |             |  get_current_user()   |
|  - authenticate_user  |             |  Handler              |
|  - create_access_token|             |  - Read payload claims|
+-----------------------+             +-----------------------+
```

---

## 2. Implemented Dependencies

### A. Credential Parsing via `OAuth2PasswordRequestForm`

For user login, standard OAuth2 specifications mandate that credentials arrive as form-encoded data (`application/x-www-form-urlencoded`). The service injects FastAPI's built-in `OAuth2PasswordRequestForm` directly into the handler:

```python
# src/main.py
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

* **Mechanism**: When a request reaches `POST /api/v1/auth/login` (see [[summaries/api-endpoints]]), FastAPI automatically parses the form body and constructs an `OAuth2PasswordRequestForm` instance containing `form_data.username` and `form_data.password`.
* **Data Flow**: The parsed fields are forwarded to the domain validation layer via `authenticate_user()` (see [[concepts/credential-validation-flow]]).

---

### B. Route Protection via `verify_token`

For protected routes like `GET /api/v1/auth/me`, identity verification is achieved by providing a callable dependency (`verify_token`) to `Depends`:

```python
# src/main.py
@app.get("/api/v1/auth/me")
async def get_current_user(token_data: dict = Depends(verify_token)):
    return {"username": token_data["sub"], "status": "active"}
```

* **Resolution Chain**:
  1. The client sends a request to `/api/v1/auth/me`.
  2. FastAPI inspects `verify_token(token: str)` defined in `src/auth.py` (see [[entities/auth-engine]]).
  3. FastAPI parses the incoming `token` argument from the request parameters.
  4. `verify_token` performs cryptographic validation using `settings.JWT_SECRET` and `settings.ALGORITHM` (see [[concepts/jwt-signing-verification]]).
  5. The returned dictionary (conforming to [[entities/jwt-token-payload]]) is injected as `token_data` into `get_current_user()`.

```python
# src/auth.py
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

---

## 3. Architectural Benefits

| Benefit | Architectural Impact in `mini-auth-service` |
| :--- | :--- |
| **Separation of Concerns** | Routing handlers in [[entities/api-router]] focus on HTTP response formatting, while cryptographic verification is encapsulated in [[entities/auth-engine]]. |
| **Declarative Route Guards** | Any route can be guarded by adding `Depends(verify_token)` without modifying endpoint logic. |
| **Stateless Scalability** | Integrates seamlessly with [[decisions/stateless-vs-stateful-sessions]], ensuring no session state is maintained across requests. |
| **Testability & Mocking** | Dependencies can be overridden during testing using FastAPI's `app.dependency_overrides` dictionary, enabling unit tests to bypass token decoding or simulate failed authentications. |

---

## 4. Production Evolution & Considerations

While the current dependency injection pattern fulfills core requirements, several enhancements are planned for production readiness:

### 1. Header Extraction via `OAuth2PasswordBearer`
Currently, `verify_token(token: str)` requires FastAPI to infer `token` as a parameter. In production, this dependency should incorporate `OAuth2PasswordBearer` to extract tokens directly from standard HTTP `Authorization: Bearer <token>` headers:

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

def get_current_user_token(token: str = Depends(oauth2_scheme)) -> dict:
    payload = verify_token(token)
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return payload
```

### 2. Dependency-Level Exception Handling
In the current implementation, `verify_token()` returns `None` on failure, which causes an unhandled `TypeError` if `get_current_user` attempts to index `token_data["sub"]`. Raising an `HTTPException(401)` directly within the dependency ensures invalid tokens are rejected before reaching business handlers.

### 3. Settings Injection
Currently, [[entities/settings]] is imported as a global module singleton. Moving to a dependency-injected settings provider (`Depends(get_settings)`) aligns with best practices for [[decisions/environment-configuration-management]].

---

## 5. Related Architecture Links

* **System Design**: [[summaries/architecture-overview]], [[index]]
* **API Specifications**: [[summaries/api-endpoints]]
* **Authentication Concepts**: [[concepts/token-based-authentication]], [[concepts/jwt-signing-verification]], [[concepts/credential-validation-flow]]
* **Code Components**: [[entities/api-router]], [[entities/auth-engine]], [[entities/settings]]