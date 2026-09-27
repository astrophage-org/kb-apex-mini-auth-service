# ADR: Adoption of Symmetric HS256 Signing for Stateless JWT Authentication

## Status
Accepted

---

## Context

The **Mini Auth Service** requires a secure, high-performance mechanism to issue identity credentials and authenticate client requests across protected API routes such as `GET /api/v1/auth/me` (see [[entities/presentation-main]] and [[summaries/api-reference]]). 

In designing the token lifecycle and session strategy (see [[concepts/stateless-session-pattern]]), the architecture had to choose between:
1. **Server-side Session Storage**: Persisting session state in a shared memory store (e.g., Redis or a database), requiring network I/O and shared infrastructure on every incoming request.
2. **Asymmetric Token Cryptography (RS256 / ES256)**: Public/private key pairs where the auth server signs with a private key and downstream microservices verify with a public key.
3. **Symmetric Token Cryptography (HS256)**: HMAC with SHA-256 using a single shared secret key for both encoding and decoding tokens.

Because the current service operates as a unified, lightweight microservice where both token issuance (`create_access_token`) and token verification (`verify_token`) are handled within the same domain (see [[entities/auth-domain]]), asymmetric key distribution was evaluated against operational complexity and performance requirements.

---

## Decision

We have decided to adopt **Symmetric HS256 (`HMAC-SHA256`)** as the standard token signing algorithm for JSON Web Tokens (JWT).

```
 +---------------------------------------------------------+
 |                   Token Issuance                        |
 | Claims (sub, exp) + JWT_SECRET ──(HS256)──> Signed JWT  |
 +---------------------------------------------------------+
                              │
                              ▼
 +---------------------------------------------------------+
 |                 Token Verification                      |
 | Inbound JWT + JWT_SECRET ────────(HS256)──> Claims      |
 +---------------------------------------------------------+
```

### Implementation Specifications

1. **Algorithm Configuration**: 
   The signing standard is explicitly defined as `HS256` within [[entities/configuration-settings]] via `Settings.ALGORITHM = "HS256"`.
2. **Domain Encapsulation**: 
   The domain logic in [[entities/auth-domain]] relies on `PyJWT` to perform symmetric encoding and decoding:
   - **Encoding**: `jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.ALGORITHM)` inside `create_access_token()`.
   - **Decoding & Guard Enforcement**: `jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.ALGORITHM])` inside `verify_token()`.
3. **Explicit Algorithm Pinning**:
   To mitigate algorithm substitution attacks (e.g., `alg: "none"` or RSA/HMAC confusion), `jwt.decode` explicitly pins allowed algorithms to `algorithms=[settings.ALGORITHM]`.
4. **Integration with DI**:
   The verification logic connects directly into route protection using FastAPI dependency injection (see [[decisions/fastapi-dependency-injection]] and [[concepts/token-verification-guard]]).

---

## Consequences

### Positive
- **Computational Efficiency**: HS256 is significantly faster to sign and verify than RSA or ECDSA asymmetric algorithms, reducing latency on protected endpoints.
- **Minimal Operational Overhead**: Does not require generating, managing, rotating, and distributing public/private key-pair files (e.g., `.pem` / `.pub`).
- **Stateless Horizontal Scaling**: Fulfills the requirements of [[concepts/stateless-session-pattern]], allowing any instance of the application with access to `JWT_SECRET` to verify tokens independently without database lookups.
- **Standardized Payload Handling**: Integrates seamlessly with standard OAuth2 password flow claims (`sub`, `exp`) described in [[concepts/jwt-authentication-flow]].

### Negative & Trade-offs
- **Shared Secret Exposure**: Any consuming service that needs to verify the token locally must have access to `JWT_SECRET`. If a distributed microservice fleet requires token verification without having the authority to issue tokens, HS256 would represent a security risk.
- **Key Storage Dependency**: The security of the entire authentication boundary depends on the entropy and confidentiality of `JWT_SECRET`. Currently, the secret is statically configured, requiring future migration to dynamic environment secrets (see [[summaries/roadmap-and-improvements]]).

---

## Related References
- [[summaries/architecture-overview]] — Service architecture and module relationships.
- [[concepts/jwt-authentication-flow]] — Token issuance mechanisms and lifecycle.
- [[concepts/token-verification-guard]] — Cryptographic validation and route guarding.
- [[decisions/singleton-configuration]] — Management of cryptographic constants via `Settings`.
- [[decisions/fastapi-dependency-injection]] — Execution of `verify_token` via route dependencies.