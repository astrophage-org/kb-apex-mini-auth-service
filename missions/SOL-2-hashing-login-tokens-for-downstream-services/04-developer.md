---
mission: SOL-2
title: 'Hashing login tokens for downstream services'
role: developer
status: ai_drafted
version: 1
author: Sol
ai_drafted: true
---

# Build spec: Downstream Login Token Hashing

## Files and services touched

### mini-auth-service
- `src/auth.py`: Add `hash_token(token: str) -> str` utility using standard library `hashlib.sha256` ([[kb:mini-auth-service/entities/auth-domain]]).
- `src/main.py`: Update login route handlers (`/api/v1/auth/login` and `/auth/login`) to hash access tokens before passing to downstream dispatch channels while preserving the raw JWT client response ([[kb:mini-auth-service/entities/presentation-main]]).
- `tests/test_auth.py`: Add unit tests for `hash_token` hashing logic, empty/invalid input safety, and token verification rejection of hash strings.
- `tests/test_api.py`: Add integration tests for login endpoint responses, downstream token payload verification, and route guard replay rejection on `GET /api/v1/auth/me`.

## What to reuse

- `hashlib.sha256` from Python standard library for deterministic cryptographic hashing.
- `create_access_token(data: dict, expires_delta: timedelta = None)` in `src/auth.py` ([[kb:mini-auth-service/entities/auth-domain]]) for initial JWT generation.
- `verify_token(token: str)` in `src/auth.py` ([[kb:mini-auth-service/entities/auth-domain]]) for dependency guard validation in protected routes.
- `settings` in `src/config.py` ([[kb:mini-auth-service/entities/configuration-settings]]) for algorithm and secret configurations.
- `OAuth2PasswordRequestForm` and `HTTPException` handling in `src/main.py` ([[kb:mini-auth-service/entities/presentation-main]]).

## Tasks

1. **Add `hash_token` utility to Auth Domain (`src/auth.py`)**
   - Implement `hash_token(token: str) -> str` in `src/auth.py` ([[kb:mini-auth-service/entities/auth-domain]]).
   - Encode `token` to UTF-8 bytes and compute `hashlib.sha256(token.encode("utf-8")).hexdigest()`.
   - Ensure `hash_token` safely raises `ValueError` or handles empty/non-string token inputs without emitting raw credentials.

2. **Integrate downstream token hashing in Presentation Layer (`src/main.py`)**
   - Import `hash_token` into `src/main.py` ([[kb:mini-auth-service/entities/presentation-main]]).
   - In `login` handler (`POST /api/v1/auth/login`) and alias `POST /auth/login` ([[kb:mini-auth-service/summaries/api-reference]]), generate the raw JWT via `create_access_token(data={"sub": user["username"]})`.
   - Compute `hashed_token = hash_token(token)`.
   - Route downstream event/message payloads (`event_topic Header.Payload.Signature`) to use `hashed_token` alongside non-sensitive user metadata (`user["username"]`), ensuring raw JWT strings are never passed downstream.
   - Return the unmodified response body `{"access_token": token, "token_type": "bearer"}` to the HTTP caller.

3. **Implement unit tests (`tests/test_auth.py`)**
   - Test deterministic SHA-256 hex digest generation for valid JWT strings.
   - Test input boundary and error handling for empty or malformed token strings.
   - Test that `verify_token` in `src/auth.py` ([[kb:mini-auth-service/decisions/jwt-hs256-signing]]) rejects hashed token strings and returns `None`.

4. **Implement integration and contract tests (`tests/test_api.py`)**
   - Test `POST /api/v1/auth/login` and `POST /auth/login` return `200 OK` with valid raw bearer token in response payload (`access_token`, `token_type`).
   - Verify downstream emitted events/payloads contain the 64-character SHA-256 hex string and 0 raw JWT strings.
   - Test replay prevention: verify `GET /api/v1/auth/me` and `GET /auth/me` with header `Authorization: Bearer <hashed_token>` return `401 Unauthorized` ([[kb:mini-auth-service/decisions/fastapi-dependency-injection]]).
   - Test consecutive logins for the same user produce distinct hashed tokens downstream due to unique issuance timestamps.

## Test plan

### Unit Tests
- **File**: `tests/test_auth.py`
  - `test_hash_token_deterministic`: Proves SHA-256 output matches expected hex digest for a known input string. Covers AC-1.
  - `test_hash_token_invalid_input`: Proves empty/null strings raise `ValueError` and do not emit invalid state downstream. Covers Edge Cases.
  - `test_verify_token_rejects_hashed_token`: Proves `verify_token(hashed_token)` returns `None` due to missing three-part base64 JWT structure. Covers AC-3, Replay Risk.

### Integration Tests
- **File**: `tests/test_api.py`
  - `test_login_returns_raw_jwt_to_client`: Proves `POST /api/v1/auth/login` returns valid client credentials matching schema `{"access_token": str, "token_type": "bearer"}`. Covers AC-3, `POST /api/v1/auth/login`.
  - `test_downstream_dispatch_receives_hashed_token`: Proves downstream dispatch receives the 64-character SHA-256 hash and not the raw JWT. Covers AC-1, AC-2, `event_topic Header.Payload.Signature`.
  - `test_replaying_hashed_token_fails`: Proves sending `Authorization: Bearer <hashed_token>` to `GET /api/v1/auth/me` returns `401 Unauthorized`. Covers AC-3, `GET /api/v1/auth/me`.
  - `test_consecutive_logins_unique_hashes`: Proves successive logins produce unique timestamps and distinct downstream hashes. Covers Edge Cases.

### Commands to run
```bash
# Run unit tests
pytest tests/test_auth.py -v

# Run integration tests
pytest tests/test_api.py -v

# Run complete test suite
pytest
```

## Rollout

- **Migrations**: None. `mini-auth-service` operates statelessly with no database migrations required.
- **Flags & Configuration**: No configuration changes required in `src/config.py` ([[kb:mini-auth-service/entities/configuration-settings]]).
- **Deploy steps**:
  1. Deploy updated `mini-auth-service` container image.
  2. Verify downstream consumers handle opaque 64-character hex strings for session tracking without attempting `jwt.decode`.
  3. Verify application health check and smoke-test `POST /api/v1/auth/login` and `GET /api/v1/auth/me`.

## Verification checklist

- [ ] Task 1 done: `hash_token(token: str)` implemented in `src/auth.py` ([[kb:mini-auth-service/entities/auth-domain]]).
- [ ] Task 2 done: `src/main.py` updated to send hashed tokens to downstream dispatch while returning raw JWTs to client.
- [ ] Task 3 done: Unit tests written and passing in `tests/test_auth.py`.
- [ ] Task 4 done: Integration tests written and passing in `tests/test_api.py`.
- [ ] Reused components (`hashlib.sha256`, `create_access_token`, `verify_token`, `settings`) not duplicated.
- [ ] Contracts marked unchanged (`POST /api/v1/auth/login`, `POST /auth/login`, `GET /api/v1/auth/me`, `GET /auth/me`) are untouched in client schema.
- [ ] Deploy and rollout steps completed and verified with 0 raw JWT exposure downstream.
