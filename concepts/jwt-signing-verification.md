# Cryptographic Signing & Token Verification

This document details the cryptographic principles, implementations, and operational security measures used to sign and verify JSON Web Tokens (JWTs) in the `mini-auth-service`. The service relies on symmetric encryption via the **HMAC-SHA256** algorithm (`HS256`) to maintain stateless authentication across client requests.

---

## 1. Overview of the Cryptographic Model

In `mini-auth-service`, token generation and validation are governed by the [[entities/auth-engine]] (`src/auth.py`), drawing cryptographic secrets and parameters from the centralized [[entities/settings]] (`src/config.py`). 

The service operates on a **symmetric key cryptography** model:
- The **same secret key** (`settings.JWT_SECRET`) is used to both sign the token during authentication and verify the token signature during downstream API requests.
- The **hashing algorithm** (`settings.ALGORITHM = "HS256"`) creates a digital signature based on a SHA-256 hash combined with the shared secret.

For details on the stateless session model enabled by this architecture, see [[decisions/stateless-vs-stateful-sessions]] and [[concepts/token-based-authentication]].

```
+-----------------------------------------------------------------------+
|                             JWT Structure                             |
|                                                                       |
|  +--------------------+   +--------------------+   +---------------+  |
|  |       Header       | . |      Payload       | . |   Signature   |  |
|  |  {"alg": "HS256"}  |   |   {"sub": "...",   |   | HMACSHA256(   |  |
|  |                    |   |    "exp": ...}     |   |   Header +    |  |
|  +--------------------+   +--------------------+   |   Payload,    |  |
|          Base64Url               Base64Url         |    Secret)    |  |
|                                                    +---------------+  |
+-----------------------------------------------------------------------+
```

---

## 2. Token Signing Mechanism

Token creation is executed via `create_access_token()` in [[entities/auth-engine]] after user credentials have been validated (see [[concepts/credential-validation-flow]]).

### Implementation (`src/auth.py`)

```python
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)
```

### Key Steps in Token Generation

1. **Payload Construction**: 
   The function takes an input payload dictionary (typically containing the subject identity `sub`) and clones it to avoid mutating source data.
2. **Expiration Enforcement (`exp`)**: 
   A cryptographic expiration timestamp is computed using `datetime.utcnow()`. By default, tokens expire in **15 minutes** unless an explicit `expires_delta` is provided. The computed Unix epoch timestamp is injected into the payload under the standard `exp` claim. For the exact schema, see [[entities/jwt-token-payload]].
3. **Encoding & Signing**: 
   The `PyJWT` library (`jwt.encode`) performs the following:
   - Serializes the header (`{"alg": "HS256", "typ": "JWT"}`) and payload to JSON.
   - Encodes both header and payload using Base64URL encoding.
   - Computes the HMAC-SHA256 signature:
     $$\text{Signature} = \text{HMAC-SHA256}(\text{Base64Url}(\text{Header}) + "." + \text{Base64Url}(\text{Payload}), \text{JWT\_SECRET})$$
   - Concatenates the three components separated by dots (`.`): `Header.Payload.Signature`.

---

## 3. Token Verification Mechanism

When a client accesses a protected endpoint, such as `GET /api/v1/auth/me` (see [[summaries/api-endpoints]]), token verification occurs through FastAPI's [[concepts/dependency-injection]] pipeline.

### Implementation (`src/auth.py`)

```python
def verify_token(token: str):
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])
        return payload
    except Exception:
        return None
```

### Verification Checks

When `jwt.decode()` processes an incoming token, it executes three mandatory verification stages:

1. **Well-Formedness Check**:
   - Ensures the token contains exactly three dot-separated Base64URL-encoded segments.
2. **Cryptographic Signature Verification**:
   - Computes a new HMAC-SHA256 signature using the incoming token's header and payload alongside the locally configured `settings.JWT_SECRET`.
   - Compares the recalculated signature against the signature provided in the third segment using a constant-time comparison to prevent timing attacks.
3. **Claim Validation (`exp`)**:
   - Decodes the `exp` claim from the payload.
   - Compares the current UTC time (`datetime.utcnow()`) against the token's expiration timestamp. If `current_time > exp`, verification fails with an `ExpiredSignatureError`.

---

## 4. Security Considerations & Protections

### Algorithm Confusion Attacks
A common vulnerability in JWT implementations is the **algorithm confusion attack**, where an attacker alters the header to specify `"alg": "none"` or switches an asymmetric key algorithm to symmetric verification.

`mini-auth-service` mitigates this vulnerability by strictly constraining the allowed algorithms parameter:
```python
jwt.decode(..., algorithms=[settings.ALGORITHM])
```
Passing an explicit list (e.g., `["HS256"]`) forces `PyJWT` to reject tokens signed with any algorithm other than the defined whitelist, preventing arbitrary or downgraded algorithm attacks.

### Secret Key Management
The strength of HMAC-SHA256 relies entirely on the secrecy and entropy of `settings.JWT_SECRET`.
- In the current implementation, `JWT_SECRET` is statically configured in `src/config.py`.
- For production environments, this key must be injected via secure environment variables or a key management service (KMS), as described in [[decisions/environment-configuration-management]].

### Symmetric Key Architecture Trade-offs
Because `HS256` uses a symmetric secret, every microservice that needs to decode and verify incoming tokens must possess the secret key. If a downstream consumer is compromised, an attacker could forge arbitrary tokens. 

To learn more about transitioning to asymmetric key pairs (where `mini-auth-service` holds a private key to sign and consumers hold a public key to verify), refer to [[decisions/symmetric-vs-asymmetric-signing]].

---

## 5. Architectural Integration

The following diagram illustrates how cryptographic signing and verification integrate across the system layers:

```
[ Client ] 
    |
    | (1) POST /api/v1/auth/login { username, password }
    v
[ src/main.py: login() ] ----------------------------------------------+
    |                                                                  |
    | (2) authenticate_user()                                          |
    v                                                                  |
[ src/auth.py: authenticate_user() ]                                   |
    |                                                                  |
    | (3) Return user record                                           |
    v                                                                  |
[ src/auth.py: create_access_token() ]                                 |
    |                                                                  |
    | (4) Sign via HMAC-SHA256(Payload, settings.JWT_SECRET)          |
    v                                                                  |
[ Client ] <--- (5) 200 OK { "access_token": "...", ... } <------------+
    |
    | (6) GET /api/v1/auth/me (token)
    v
[ src/main.py: get_current_user() ]
    |
    | (7) Depends(verify_token)
    v
[ src/auth.py: verify_token() ]
    |
    | (8) jwt.decode(token, secret, algorithms=["HS256"])
    |     - Check signature integrity
    |     - Verify exp > now()
    v
[ src/main.py: returns user info ]
```

- **Issuance**: Handled by `login()` in [[entities/api-router]] calling `create_access_token()`.
- **Validation**: Enforced via `verify_token` dependency in [[entities/auth-engine]] injected into protected routes.
- **System Architecture**: Summarized in [[summaries/architecture-overview]] and indexed in [[index]].