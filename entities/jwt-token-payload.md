<!-- anchor: src/auth.py:L1-L100 sha:HEAD -->

# JWT Token Payload

The **JWT Token Payload** represents the underlying dictionary structure and standard claim set encoded within the JSON Web Tokens issued and verified by `mini-auth-service`. It encapsulates identity context and lifecycle metadata according to the [[concepts/token-based-authentication]] model.

---

## Responsibilities

* **Identity Encapsulation**: Carry the authenticated user's unique identity (`sub`) across stateless HTTP requests.
* **Temporal Validation**: Store cryptographic expiration bounds (`exp`) to restrict token validity lifetimes without requiring database lookups.
* **Stateless Authorization Context**: Provide a verified payload to route handlers downstream of [[concepts/dependency-injection]] pipelines.

---

## Dependencies

* **[[entities/settings]]**: Supplies `JWT_SECRET` and `ALGORITHM` (HS256) used during serialization and decoding.
* **[[entities/auth-engine]]**: Constructing and signing the payload in `create_access_token()` and parsing/validating it in `verify_token()`.
* **[[entities/api-router]]**: Ingests the decoded payload dictionary within `get_current_user()` to return user profile data.
* **PyJWT (`jwt`)**: Handles JSON serialization, base64url encoding, timestamp parsing, and RFC 7519 claim validation.

---

## Claim Specification

The token payload implements standard Registered Claim Names defined in RFC 7519:

| Claim | Key | Type | Description | Generation Source |
| :--- | :--- | :--- | :--- | :--- |
| **Subject** | `sub` | `str` | Identifies the principal subject of the token (the authenticated username). | Derived from `user["username"]` during [[concepts/credential-validation-flow]]. |
| **Expiration** | `exp` | `int` / `datetime` | UTC timestamp identifying when the token expires and ceases to be valid. | Generated dynamically as `datetime.utcnow() + timedelta(minutes=15)`. |

### JSON Schema Representation

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "JwtTokenPayload",
  "type": "object",
  "properties": {
    "sub": {
      "type": "string",
      "description": "Unique username or subject identifier"
    },
    "exp": {
      "type": "integer",
      "description": "Unix timestamp representing token expiration"
    }
  },
  "required": ["sub", "exp"],
  "additionalProperties": false
}
```

### Example Decoded Payload

```json
{
  "sub": "admin",
  "exp": 1717156500
}
```

---

## Payload Lifecycle

```
[ POST /api/v1/auth/login ]
           |
           v
create_access_token(data={"sub": user["username"]})
           |
           +---> Injects {"exp": utcnow() + 15 min}
           +---> Signed with HS256 via [[concepts/jwt-signing-verification]]
           |
           v
[ Encoded Bearer Token String ]
           |
           v
[ GET /api/v1/auth/me ]
           |
           v
verify_token(token)
           |
           +---> Validates signature & 'exp' timestamp
           |
           v
Decoded Payload dict {"sub": "admin", "exp": 1717156500}
           |
           v
Injected into get_current_user(token_data)
```

1. **Construction**: During authentication in [[summaries/api-endpoints]], `create_access_token` accepts a base dictionary (`{"sub": "admin"}`) and computes the default 15-minute expiration delta.
2. **Encoding**: `jwt.encode` converts the dictionary into a three-part Base64URL-encoded signed string (`header.payload.signature`).
3. **Transmission**: The payload is transmitted to the client inside the response entity:
   ```json
   {
     "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
     "token_type": "bearer"
   }
   ```
4. **Decoding & Verification**: When presented on protected endpoints, `jwt.decode` validates signature integrity against `settings.JWT_SECRET` and ensures `datetime.utcnow() < payload["exp"]`.

---

## Design Considerations

* **Statelessness**: Because all verification data resides within the payload and signature, the service requires no database session reads, supporting the decisions outlined in [[decisions/stateless-vs-stateful-sessions]].
* **Short Expiration Horizon**: The 15-minute lifetime limits the attack window if a token is intercepted, compensating for the lack of a token revocation list.
* **Extensibility**: Future iterations may append custom claims (e.g., `roles`, `permissions`, or `iss`/`aud` claims) to support fine-grained authorization checks and migrations to [[decisions/symmetric-vs-asymmetric-signing]].