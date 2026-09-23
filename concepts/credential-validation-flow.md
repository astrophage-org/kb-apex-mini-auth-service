# Credential Validation Flow

The credential validation flow describes the end-to-end process by which a client submits user identity credentials, the service verifies those credentials, and a cryptographically signed JSON Web Token (JWT) is issued.

This flow represents the entry point for [[concepts/token-based-authentication]] within the `mini-auth-service` application, connecting the [[entities/api-router]], the [[entities/auth-engine]], and configuration parameters managed by [[entities/settings]].

---

## 1. High-Level Sequence Diagram

The following sequence details the interactions across components during the login and credential validation workflow:

```
Client                     API Router (main.py)           Auth Engine (auth.py)        Settings (config.py)
  |                                 |                               |                         |
  |-- 1. POST /api/v1/auth/login -->|                               |                         |
  |   (Form: username/password)     |-- 2. authenticate_user() ---->|                         |
  |                                 |      (username, password)     |                         |
  |                                 |                               |                         |
  |                                 |   [ Check user credentials ]  |                         |
  |                                 |                               |                         |
  |                                 |<-- 3. user dict or None ------|                         |
  |                                 |                               |                         |
  |   [ If None: raise HTTP 401 ]   |-- 4. create_access_token() -->|                         |
  |                                 |      ({"sub": username})      |-- 5. Read Secret/Alg -->|
  |                                 |                               |<-- JWT_SECRET, ALGO ----|
  |                                 |                               |                         |
  |                                 |                               |-- [ Sign with PyJWT ]   |
  |                                 |<-- 6. Signed JWT String ------|                         |
  |                                 |                               |                         |
  |<-- 7. 200 OK (access_token) ----|                               |                         |
```

---

## 2. Step-by-Step Flow Breakdown

### Step 1: HTTP Request Ingestion and Form Parsing
* **Endpoint**: `POST /api/v1/auth/login` (documented in [[summaries/api-endpoints]]).
* **Mechanism**: Handled in [[entities/api-router]] (`src/main.py`) via FastAPI's [[concepts/dependency-injection]] system.
* The request payload must be sent as `application/x-www-form-urlencoded` conforming to the OAuth2 specification. FastAPI binds the incoming parameters into an `OAuth2PasswordRequestForm` object:
  ```python
  @app.post("/api/v1/auth/login")
  async def login(form_data: OAuth2PasswordRequestForm = Depends()):
      ...
  ```
* If the payload is missing required fields (`username` or `password`), FastAPI automatically responds with a `422 Unprocessable Entity` validation error before executing endpoint logic.

### Step 2: Credential Verification
* The [[entities/api-router]] calls `authenticate_user()` in [[entities/auth-engine]] (`src/auth.py`), passing `form_data.username` and `form_data.password`.
* The [[entities/auth-engine]] executes user lookup:
  ```python
  def authenticate_user(username: str, password: str):
      if username == "admin" and password == "secret123":
          return {"username": "admin", "role": "admin"}
      return None
  ```
* **Success**: Returns a dictionary representing the authenticated user (`{"username": "admin", "role": "admin"}`).
* **Failure**: Returns `None`.

> [!NOTE]
> The current system performs static in-memory equality checks. The transition to database querying and password hashing (e.g., Argon2 or bcrypt) is detailed in [[decisions/credential-storage-hashing]].

### Step 3: Error Dispatch or Token Construction
* If `authenticate_user` returns `None`, the router halts execution and raises an `HTTPException`:
  ```python
  raise HTTPException(
      status_code=status.HTTP_401_UNAUTHORIZED,
      detail="Incorrect username or password",
  )
  ```
* If the user is authenticated, the router proceeds to token issuance by calling `create_access_token(data={"sub": user["username"]})`.

### Step 4: Token Generation and Cryptographic Signing
* `create_access_token()` in [[entities/auth-engine]] constructs the [[entities/jwt-token-payload]]:
  1. Copies the input data payload containing the subject claim (`sub`).
  2. Computes the expiration time (`exp`) by adding default duration (15 minutes) to `datetime.utcnow()`.
  3. Updates the payload with the calculated expiration.
* Signs the payload using `jwt.encode()` with parameters supplied by [[entities/settings]]:
  * `settings.JWT_SECRET`: The shared secret key (`"astrophage-secret-key-123"`).
  * `settings.ALGORITHM`: The symmetric hashing algorithm (`"HS256"`).
* Details regarding the cryptographic mechanism can be found in [[concepts/jwt-signing-verification]] and [[decisions/symmetric-vs-asymmetric-signing]].

### Step 5: HTTP Response Serialization
* The router encapsulates the signed JWT into a JSON response structure:
  ```json
  {
    "access_token": "<jwt_string>",
    "token_type": "bearer"
  }
  ```
* Returns HTTP status code `200 OK`. The client can now use this token to access protected routes (such as `GET /api/v1/auth/me`).

---

## 3. Failure Modes and HTTP Response Codes

| Failure Condition | Handling Layer | Resulting HTTP Status Code | Response Body |
| :--- | :--- | :--- | :--- |
| Missing `username` or `password` form field | FastAPI Request Validation / [[entities/api-router]] | `422 Unprocessable Entity` | Field validation error schema |
| Incorrect username or password | `authenticate_user()` / [[entities/api-router]] | `401 Unauthorized` | `{"detail": "Incorrect username or password"}` |
| Invalid HTTP Method (e.g., `GET /api/v1/auth/login`) | FastAPI Routing Layer | `405 Method Not Allowed` | `{"detail": "Method Not Allowed"}` |

---

## 4. Architectural Characteristics

* **Stateless Operation**: No session IDs or tokens are persisted to a database or cache upon issuance. All subsequent requests carry the full authentication state within the JWT payload. See [[decisions/stateless-vs-stateful-sessions]].
* **Separation of Concerns**: HTTP parsing and status mapping are strictly isolated in [[entities/api-router]], while cryptographic signing and user authentication logic reside in [[entities/auth-engine]].
* **Configuration Decoupling**: Cryptographic keys and algorithm choices are separated from domain logic via [[entities/settings]] and [[decisions/environment-configuration-management]].

---

## Related Documentation

* **Architecture**: [[summaries/architecture-overview]], [[summaries/api-endpoints]]
* **Concepts**: [[concepts/token-based-authentication]], [[concepts/jwt-signing-verification]], [[concepts/dependency-injection]]
* **Entities**: [[entities/auth-engine]], [[entities/api-router]], [[entities/jwt-token-payload]], [[entities/settings]]
* **Decisions**: [[decisions/credential-storage-hashing]], [[decisions/symmetric-vs-asymmetric-signing]], [[decisions/stateless-vs-stateful-sessions]]