<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Auth Engine

The **Auth Engine** (`src/auth.py`) is the core security and domain layer of the `mini-auth-service`. It encapsulates the business logic for credential authentication, JSON Web Token ([[concepts/token-based-authentication|JWT]]) creation, cryptographic signing, and token decoding/verification. 

It serves as the domain core consumed by the [[entities/api-router]] and relies on configuration provided by [[entities/settings]].

---

## Responsibilities

The Auth Engine isolates security and identity operations from the HTTP transport layer. Its primary responsibilities include:

1. **User Authentication & Identity Lookup (`authenticate_user`)**:
   - Validates user-supplied credentials against stored identity records.
   - Returns a sanitized user profile dictionary on success, or `None` if verification fails.
   - Participates as the primary validation step in the [[concepts/credential-validation-flow]].

2. **Access Token Generation & Cryptographic Signing (`create_access_token`)**:
   - Constructs the token claims set defined in [[entities/jwt-token-payload]].
   - Computes token expiration timestamps (`exp`) based on default TTLs (15 minutes) or custom `timedelta` arguments.
   - Cryptographically signs the payload with HMAC-SHA256 using the secret key from [[entities/settings]].

3. **Token Decoding & Verification (`verify_token`)**:
   - Parses incoming raw JWT strings and executes [[concepts/jwt-signing-verification]].
   - Validates cryptographic signatures against the configured secret key.
   - Evaluates token lifetime to ensure the token has not expired.
   - Acts as a callable dependency for FastAPI's [[concepts/dependency-injection]] pipeline in protected endpoints.

---

## Dependencies

- **[[entities/settings]] (`src/config.py`)**: Provides runtime configuration parameters, notably `settings.JWT_SECRET` (the symmetric secret key) and `settings.ALGORITHM` (the cryptographic algorithm identifier).
- **`PyJWT` (`jwt`)**: The underlying cryptographic library used to encode, sign, decode, and validate JSON Web Tokens.
- **`datetime` / `timedelta`**: Standard library modules used to calculate UTC-based token expiration boundaries (`exp` claim).
- **Downstream Consumers**: Invoked directly by route handlers and dependency resolvers in the [[entities/api-router]] (`src/main.py`).

---

## Component Interfaces & Implementation Details

```
+-------------------------------------------------------------+
|                          Auth Engine                        |
|                        (`src/auth.py`)                      |
+-------------------------------------------------------------+
| + authenticate_user(username: str, password: str) -> dict   |
| + create_access_token(data: dict, expires_delta) -> str     |
| + verify_token(token: str) -> dict | None                   |
+-------------------------------------------------------------+
          |                                       |
          v                                       v
+--------------------+                  +--------------------+
|  PyJWT (`jwt`)     |                  |  [[entities/settings]]  |
+--------------------+                  +--------------------+
```

### 1. `authenticate_user(username: str, password: str)`

```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```

- **Behavior**: Evaluates plaintext credentials against the current static user record (`admin` / `secret123`).
- **Return Value**: Returns a dictionary representing the user record (`{"username": "admin", "role": "admin"}`) on success, or `None` on invalid credentials.
- **Architectural Evolution**: In the current implementation, identity checks are hardcoded in memory. Future iterations will route lookups through database-backed models with Argon2/bcrypt password hashing as outlined in [[decisions/credential-storage-hashing]].

---

### 2. `create_access_token(data: dict, expires_delta: timedelta = None)`

```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

- **Behavior**: Copies the input dictionary, appends an `exp` expiration timestamp (defaulting to 15 minutes from creation in UTC), and encodes the data into a compact JWT string.
- **Signing Mechanism**: Uses `settings.JWT_SECRET` and `settings.ALGORITHM` (default `HS256`). For details on signing considerations, see [[concepts/jwt-signing-verification]] and [[decisions/symmetric-vs-asymmetric-signing]].
- **Payload Schema**: Injects the claims according to [[entities/jwt-token-payload]], ensuring standard claims like `sub` and `exp` are present.

---

### 3. `verify_token(token: str)`

```python
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

- **Behavior**: Decodes the compact JWT string, verifies that the cryptographic signature matches `settings.JWT_SECRET`, and ensures the current timestamp is prior to the `exp` claim.
- **Exception Handling**: Catches exceptions (e.g., `jwt.ExpiredSignatureError`, `jwt.InvalidTokenError`, malformed strings) and safely returns `None`.
- **Integration**: Injected into routes such as `GET /api/v1/auth/me` via [[concepts/dependency-injection]] within [[entities/api-router]].

---

## Architectural Context

- **Stateless Operation**: The Auth Engine is completely stateless. It does not persist issued tokens in server memory or data stores, supporting the [[decisions/stateless-vs-stateful-sessions]] model.
- **Symmetric Cryptography**: Currently configured for HMAC-SHA256 (`HS256`). While suitable for single-service deployments, distributed microservice architectures may require transitioning to asymmetric public-key cryptography as detailed in [[decisions/symmetric-vs-asymmetric-signing]].