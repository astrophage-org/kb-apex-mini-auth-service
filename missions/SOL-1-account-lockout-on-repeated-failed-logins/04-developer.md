---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: developer
status: draft
version: 3
author: dev
ai_drafted: false
---

# Build spec: Account Lockout on Repeated Failed Logins

## Files and services touched (paths)
- **`mini-auth-service`**:
  - `src/config.py`: Add lockout configuration parameters `MAX_FAILED_LOGIN_ATTEMPTS: int = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15` to the configuration module ([[kb:mini-auth-service/entities/configuration-settings|src/config.py]]).
  - `src/auth.py`: Implement username normalization, thread-safe in-memory attempt and lockout tracking, threshold evaluation, failure incrementing, and state reset logic within the authentication domain ([[kb:mini-auth-service/entities/auth-domain|src/auth.py]]).
  - `src/main.py`: Update request handling for `POST /api/v1/auth/login` in the API presentation layer ([[kb:mini-auth-service/entities/presentation-main|src/main.py]]) to enforce active lockout guards and return lockout-specific HTTP 401 exceptions.
  - `tests/test_auth.py`: Add unit and integration tests covering failed attempt tracking, lockout triggers, lockout window enforcement, counter reset on success, lockout expiration, and non-existent username handling.

## What to reuse
- **Existing JWT Signing & Token Creation**: Reuse `create_access_token` and [[kb:mini-auth-service/decisions/jwt-hs256-signing|JWT HS256 signing]] routines in [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
- **Configuration Pattern**: Extend the existing configuration structure and singleton settings in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
- **OAuth2 Form Parsing**: Reuse FastAPI's `OAuth2PasswordRequestForm` dependency injection in [[kb:mini-auth-service/entities/presentation-main|src/main.py]].
- **Token Verification & Profile Inspection**: Preserve token decoding and guard workflows for `GET /api/v1/auth/me` without modification per [[kb:mini-auth-service/summaries/api-reference]].
- **FastAPI Exception Patterns**: Use standard FastAPI `HTTPException(status_code=401, detail=...)` exception models for all authentication failure responses.

## Tasks
1. - [ ] **Add Lockout Configuration Settings**: In [[kb:mini-auth-service/entities/configuration-settings|src/config.py]], define `MAX_FAILED_LOGIN_ATTEMPTS: int = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15`.
2. - [ ] **Implement State Tracking Data Structure**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], define a thread-safe in-memory state store tracking `{ "failed_attempts": int, "locked_until": Optional[datetime] }` keyed by normalized username (`username.strip().lower()`). Include bounded capacity/eviction handling for untracked identifiers to prevent memory exhaustion.
3. - [ ] **Implement Lockout Domain Helper Functions**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], implement helper methods to:
   - Check if an account is actively locked (`locked_until > datetime.utcnow()`).
   - Clear expired lockouts (`locked_until <= datetime.utcnow()`) and reset `failed_attempts = 0`.
   - Increment `failed_attempts` on invalid credential submission, and set `locked_until = datetime.utcnow() + timedelta(minutes=ACCOUNT_LOCKOUT_DURATION_MINUTES)` when `failed_attempts >= MAX_FAILED_LOGIN_ATTEMPTS`.
   - Reset `failed_attempts = 0` and clear `locked_until` upon successful authentication.
4. - [ ] **Integrate Lockout Guard into Authentication Workflow**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]] and [[kb:mini-auth-service/entities/presentation-main|src/main.py]], update `authenticate_user` and the `POST /api/v1/auth/login` endpoint handler ([[kb:mini-auth-service/concepts/jwt-authentication-flow]]):
   - Check lockout status before evaluating credentials; raise `HTTPException(status_code=401, detail="Account is temporarily locked. Try again later.")` if actively locked.
   - On password verification failure, record the failed attempt; if the 5th failure threshold is reached, raise `HTTPException(status_code=401, detail="Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes.")`; otherwise raise `HTTPException(status_code=401, detail="Incorrect username or password")`.
   - On password verification success, reset the failure counter and issue a signed Bearer JWT token.
5. - [ ] **Handle Non-Existent Usernames & Edge Cases**: Ensure attempts against non-existent users increment tracking safely without throwing unhandled exceptions or leaking account existence.
6. - [ ] **Implement Test Suite**: Create unit and integration tests in `tests/test_auth.py` covering failure tracking, lockout activation, active lock rejection, automatic expiration, counter resets, and casing consistency.

## Test plan
- **Unit Tests (`src/auth.py`)**:
  - `test_normalize_username`: Verify casing and leading/trailing whitespace normalization (e.g., `" Admin "` -> `"admin"`).
  - `test_failed_attempt_increment`: Verify failure counter increments sequentially from 1 up to 5 on consecutive failed attempts.
  - `test_lockout_trigger`: Verify that reaching the 5th failed attempt sets `locked_until` to `utcnow() + 15 minutes`.
  - `test_counter_reset_on_success`: Verify `failed_attempts` resets to 0 upon successful credential verification.
  - `test_lockout_expiration_reset`: Verify that when `locked_until` has passed, the lock is cleared and `failed_attempts` resets.
- **Integration Tests (`POST /api/v1/auth/login`)**:
  - `test_login_success_unlocked`: Submit valid credentials and assert HTTP 200 with Bearer JWT per [[kb:mini-auth-service/summaries/api-reference]].
  - `test_lockout_after_five_failures`: Submit 5 consecutive incorrect passwords for an account and assert HTTP 401 with lockout message on the 5th attempt.
  - `test_rejection_during_active_lockout`: Submit valid credentials for an actively locked account and assert HTTP 401 rejection without issuing a token.
  - `test_counter_reset_before_threshold`: Submit 3 incorrect attempts, followed by 1 valid login (assert 200 OK); submit another incorrect attempt and verify the failure counter restarted at 1.
  - `test_lockout_expiration_allows_login`: Fast-forward / mock time past 15 minutes on a locked account, submit valid credentials, and assert HTTP 200 with valid JWT.
  - `test_non_existent_username_handling`: Submit 5 failed attempts on an unseeded username and verify HTTP 401 lockout response without server errors.
  - `test_case_insensitive_lockout`: Verify alternating cases (`ADMIN`, `admin`, `Admin`) aggregate towards the same 5-attempt lockout threshold.
- **Regression Tests**:
  - `test_token_inspection_endpoint`: Verify `GET /api/v1/auth/me` continues to validate signed Bearer tokens as expected.
  - `test_jwt_hs256_signing`: Verify issued tokens strictly follow [[kb:mini-auth-service/decisions/jwt-hs256-signing]].

## Rollout
1. Deploy updated `mini-auth-service` application containing changes in `src/config.py`, `src/auth.py`, and `src/main.py`.
2. Verify service startup and configuration loading of `MAX_FAILED_LOGIN_ATTEMPTS` and `ACCOUNT_LOCKOUT_DURATION_MINUTES`.
3. Execute smoke tests against `POST /api/v1/auth/login` (valid login, 5 failed attempts lockout, locked account rejection) and `GET /api/v1/auth/me`.
4. Rollback Plan: Revert application image to the prior build if unexpected authentication regressions occur.

## Verification checklist
- [ ] Task 1: Lockout configuration parameters defined in `src/config.py`.
- [ ] Task 2: Thread-safe in-memory state tracking and username normalization implemented in `src/auth.py`.
- [ ] Task 3: Lockout evaluation, expiration, incrementing, and reset helper functions implemented in `src/auth.py`.
- [ ] Task 4: Lockout checks and HTTP 401 exception mappings integrated into `POST /api/v1/auth/login` in `src/main.py` and `src/auth.py`.
- [ ] Task 5: Non-existent usernames and cache bounds handled safely without unhandled exceptions.
- [ ] Task 6: Unit, integration, and regression tests implemented and passing.
- [ ] Reused components (`create_access_token`, `OAuth2PasswordRequestForm`, JWT HS256 signing) not duplicated.
- [ ] API contracts for `POST /api/v1/auth/login` and `GET /api/v1/auth/me` remain compliant with [[kb:mini-auth-service/summaries/api-reference]].
