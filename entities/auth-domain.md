<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# Auth Domain (`src/auth.py`)

The **Auth Domain** module encapsulates the core authentication business logic and cryptographic JSON Web Token (JWT) lifecycle operations for the application. It provides pure domain functions for verifying credentials, issuing signed access tokens, and validating token integrity and expiration.

For high-level context on how this module fits into the service, see the [[summaries/architecture-overview]] and the [[index]].

---

## Responsibilities

The primary responsibilities of `src/auth.py` are:

- **Credential Authentication**: Validating incoming username and password combinations against the identity store.
- **Token Construction and Signing**: Packaging identity claims (such as `sub`) with time-to-live (`exp`) timestamps and signing them using symmetric cryptography as outlined in [[decisions/jwt-hs256-signing]].
- **Token Verification & Decoding**: Cryptographically validating incoming JWTs, checking signature validity, enforcing expiration times, and returning claims payloads.
- **Error Handling during Verification**: Shielding the caller from raw decoding exceptions by returning safe sentinel values (`None`) upon validation failure.

---

## Dependencies

### Internal Dependencies
- [[entities/configuration-settings]] (`src/config.py`): Imports the `settings` singleton to access `settings.JWT_SECRET` and `settings.ALGORITHM`.

### External Dependencies
- `jwt` (`PyJWT`): Provides symmetric encoding and decoding functionality for standard JWTs.
- `datetime` (`datetime`, `timedelta`): Handles UTC-based expiration calculations for access tokens.

### Downstream Consumers
- [[entities/presentation-main]] (`src/main.py`): Imports `authenticate_user`, `create_access_token`, and `verify_token` to power API endpoints and route security guards.

---

## Core Functions

```
+-------------------------------------------------------------------------+
|                               src/auth.py                               |
+-------------------------------------------------------------------------+
|  + authenticate_user(username: str, password: str) -> dict | None       |
|  + create_access_token(data: dict, expires_delta: timedelta) -> str     |
|  + verify_token(token: str) -> dict | None                              |
+-------------------------------------------------------------------------+
```

### `authenticate_user(username: str, password: str)`
Evaluates user credentials against identity records.

```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```
- **Inputs**: Plaintext `username` and `password`.
- **Output**: Returns a user dictionary (`{"username": "admin", "role": "admin"}`) on match; returns `None` on invalid credentials.
- **Notes**: In its current iteration, credential matching uses static strings. Upgrading to database-backed persistent storage and secure password hashing (e.g., `bcrypt` or `argon2`) is planned in [[summaries/roadmap-and-improvements]].

### `create_access_token(data: dict, expires_delta: timedelta = None)`
Generates an encoded, signed JWT string containing caller claims and an expiration timestamp.

```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```
- **Inputs**:
  - `data` (`dict`): The payload claims to encode (e.g., `{"sub": "admin"}`).
  - `expires_delta` (`timedelta`, optional): Custom lifespan for the token. Defaults to **15 minutes**.
- **Output**: Encoded JWT string signed with `settings.JWT_SECRET` using `settings.ALGORITHM` (`HS256`).
- **Detailed Flow**: See [[concepts/jwt-authentication-flow]] for payload formatting and lifecycle details.

### `verify_token(token: str)`
Decodes an incoming JWT, verifying both its cryptographic signature and validity window.

```python
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```
- **Inputs**: Encoded JWT string extracted from client requests.
- **Output**: Decoded payload dictionary if valid; `None` if the signature is invalid, the token has expired, or the string is malformed.
- **Integration**: Used as a dependency injection guard in [[entities/presentation-main]] to protect restricted routes like `GET /api/v1/auth/me`. See [[concepts/token-verification-guard]] and [[decisions/fastapi-dependency-injection]] for usage patterns.

---

## Architectural Considerations

- **Stateless Architecture**: By encoding identity claims and expiration directly inside the JWT, the authentication layer avoids holding state in server memory or an active session store. See [[concepts/stateless-session-pattern]].
- **Configuration Decoupling**: Cryptographic parameters are isolated in [[entities/configuration-settings]] via [[decisions/singleton-configuration]].
- **Contract Reference**: For details on how these domain functions map to API endpoints and HTTP response schemas, refer to [[summaries/api-reference]].