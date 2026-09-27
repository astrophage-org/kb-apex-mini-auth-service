# Architectural Decision Record: Stateless JWT Tokens vs. Stateful Database-Backed Sessions

## Status
Accepted

## Context
When architecting the authentication and session management subsystem for `mini-auth-service` (detailed in [[summaries/architecture-overview]]), the primary requirement was to provide identity verification and authorization for REST endpoints (`/api/v1/auth/login` and `/api/v1/auth/me`, specified in [[summaries/api-endpoints]]).

We evaluated two foundational session management strategies:

1. **Stateful Database-Backed Sessions**:
   - The server creates a session identifier upon authentication, stores the session data (user ID, permissions, expiration) in a centralized datastore (such as Redis or a relational database via `DATABASE_URL` in [[entities/settings]]), and sets a session cookie or token on the client.
   - Every incoming authenticated request requires a database or cache query to validate the session state, check revocation status, and retrieve user attributes.

2. **Stateless Token-Based Authentication**:
   - The server issues a cryptographically signed JSON Web Token (JWT) encapsulating identity claims (`sub`) and validity timestamps (`exp`) in an [[entities/jwt-token-payload]].
   - Downstream verification relies entirely on local cryptographic validation of the token signature and expiration claims without querying a centralized session store.

Given the goal of building a lightweight, highly scalable identity layer suitable for distributed microservice architectures, the storage, scaling, and operational characteristics of both approaches were analyzed.

## Decision
We decided to adopt **Stateless Token-Based Authentication** using signed JSON Web Tokens (JWTs) as the primary session mechanism across `mini-auth-service`.

Key aspects of this implementation include:
- **Self-Contained Claims**: Authentication state is encapsulated directly inside the token payload via `create_access_token()` in the [[entities/auth-engine]] layer.
- **Cryptographic Signature Verification**: Token integrity and validity are verified mathematically in `verify_token()` using HMAC-SHA256 (`HS256`) as detailed in [[concepts/jwt-signing-verification]], eliminating database reads during route verification.
- **Short Token Lifetime**: To mitigate the security risks of statelessness (inability to instantly revoke active tokens), access tokens are issued with a default short lifespan of 15 minutes (`timedelta(minutes=15)`).
- **Decoupled Route Handling**: Protected endpoints in [[entities/api-router]] resolve identity context purely through FastAPI dependency injection ([[concepts/dependency-injection]]) by decoding and validating the inbound Bearer token.

```
Client                        API Router (main.py)              Auth Engine (auth.py)          Database
  |                                    |                                  |                        |
  |--- 1. POST /login (credentials) -->|                                  |                        |
  |                                    |--- 2. authenticate_user() ------>|                        |
  |                                    |                                  |-- Validate User ------>|
  |                                    |<-- 3. Return user record --------|<-- User verified ------|
  |                                    |                                  |
  |                                    |--- 4. create_access_token() ---->|  (Generates signed JWT)
  |                                    |<-- 5. Return signed JWT ---------|
  |<-- 6. 200 OK (access_token) -------|
  |
  |=== (Subsequent Authenticated Requests: GET /me) ===============================================|
  |
  |--- 7. GET /me (Bearer token) ----->|                                  |  (No DB Query Required!)
  |                                    |--- 8. verify_token(token) ------>|
  |                                    |<-- 9. Return valid payload ------|
  |<-- 10. 200 OK (user payload) ------|
```

## Consequences

### Positive Consequences
- **Horizontal Scalability & Performance**: Because protected endpoints do not query a database or cache to validate sessions, request latency is minimized and API instances can scale out horizontally without adding load to a shared database.
- **Microservice Portability**: Downstream services can verify identity claims autonomously by sharing the signing secret or public key (see [[decisions/symmetric-vs-asymmetric-signing]]), decoupling internal services from synchronous authentication queries.
- **Low Operational Overhead**: No distributed cache infrastructure (e.g., Redis cluster) is required to maintain session state across deployments or restarts.

### Negative Consequences & Trade-offs
- **Revocation Complexity**: Because state is not maintained server-side, a compromised token remains valid until its `exp` claim expires (up to 15 minutes). Instant revocation requires maintaining a token blocklist or revocation list, which reintroduces state.
- **Payload Overhead**: JWTs are larger in byte size than standard session cookies due to Base64URL-encoded headers and claims, resulting in increased HTTP request header overhead.
- **Token Invalidation on Password Change**: Changing a user's password in the database (see [[decisions/credential-storage-hashing]]) does not automatically invalidate currently active JWTs unless explicit token versioning or short TTLs are enforced.

## Related Documentation
- [[concepts/token-based-authentication]] - Detailed conceptual model for token issuance and lifecycle.
- [[concepts/jwt-signing-verification]] - Cryptographic mechanics of signing and decoding tokens.
- [[concepts/credential-validation-flow]] - End-to-end credential verification and token creation flow.
- [[decisions/symmetric-vs-asymmetric-signing]] - Evaluation of symmetric (`HS256`) vs. asymmetric (`RS256`) signing schemes.
- [[decisions/credential-storage-hashing]] - Strategy for persistent user storage and password hashing.