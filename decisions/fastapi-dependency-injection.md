# Architectural Decision Record: FastAPI Dependency Injection for Route Guards

## Status

**Accepted**

---

## Context

In building the [[summaries/architecture-overview|Mini Auth Service]], the presentation layer in [[entities/presentation-main|src/main.py]] required a standardized, robust mechanism to enforce security controls and validate identity claims before executing protected endpoint logic. 

Without a structured interception pattern, route handlers would need to manually parse HTTP headers or request bodies, invoke cryptographic decoding routines from [[entities/auth-domain|src/auth.py]], handle signature/expiration errors, and construct identity contexts. Such manual handling causes:
1. **Code Duplication**: Repetitive token parsing and validation code across multiple protected endpoints.
2. **Tight Coupling**: Inadvertently entangling HTTP parsing with authentication domain logic and token verification mechanics.
3. **Testing Friction**: Difficulties in mocking or overriding identity contexts during integration and unit testing.

The system needed a declarative, modular mechanism to enforce route-level authentication guards while supporting the [[concepts/jwt-authentication-flow|OAuth2 password flow]] and [[concepts/stateless-session-pattern|stateless session model]].

---

## Decision

We decided to leverage **FastAPI's built-in Dependency Injection (DI) system** (`fastapi.Depends`) to manage request-level credential parsing and token verification guards.

Specifically:
1. **Credential Ingestion**: The login endpoint (`/api/v1/auth/login`) injects `OAuth2PasswordRequestForm` via `Depends()` to automatically parse standard `application/x-www-form-urlencoded` credentials (`username` and `password`).
2. **Authentication Guards**: Protected routes (such as `GET /api/v1/auth/me` documented in [[summaries/api-reference|API Reference]]) declare dependencies on domain validation functions:
   ```python
   @app.get("/api/v1/auth/me")
   async def get_current_user(token_data: dict = Depends(verify_token)):
       return {"username": token_data["sub"], "status": "active"}
   ```
3. **Execution Pipeline**: The `verify_token` function in [[entities/auth-domain|src/auth.py]] acts as the reusable [[concepts/token-verification-guard|token verification guard]]. It reads runtime configuration from [[entities/configuration-settings|src/config.py]], decodes the token using the symmetric key defined in [[decisions/jwt-hs256-signing|JWT HS256 Signing]], and provides the validated payload dictionary directly to downstream route handlers.

---

## Consequences

### Positive
* **Declarative Security**: Route signatures clearly declare their authentication requirements in the function signature, improving readability and maintainability.
* **Separation of Concerns**: Endpoint handlers in [[entities/presentation-main|src/main.py]] focus purely on business responses, delegating validation to [[entities/auth-domain|src/auth.py]].
* **Testability**: Dependencies can be easily replaced during testing using FastAPI's `app.dependency_overrides` dictionary, enabling end-to-end testing without generating valid cryptographic tokens for every test case.
* **OpenAPI Documentation**: FastAPI automatically reads dependencies to generate accurate OpenAPI schema contracts.

### Negative & Trade-offs
* **Direct Parameter Binding Limitations**: In the current implementation, `verify_token` binds directly as a generic parameter dependency rather than using FastAPI's `OAuth2PasswordBearer` security scheme.
* **Implicit Header Extraction**: As outlined in [[summaries/roadmap-and-improvements|Roadmap and Improvements]], future iterations will integrate `OAuth2PasswordBearer` to standardize HTTP `Authorization: Bearer <token>` header extraction, automated 401 error signaling, and OpenAPI security scheme declarations.

---

## Related Documents

* [[entities/presentation-main]] - Implementation of endpoints and dependency injection in `src/main.py`
* [[entities/auth-domain]] - Token verification guard implementation in `src/auth.py`
* [[concepts/token-verification-guard]] - Concept explanation for token verification and route protection
* [[decisions/jwt-hs256-signing]] - Symmetric signing decision utilized by the verification guard
* [[decisions/singleton-configuration]] - Centralized runtime configuration used during verification
* [[summaries/roadmap-and-improvements]] - Planned enhancements for `OAuth2PasswordBearer` integration