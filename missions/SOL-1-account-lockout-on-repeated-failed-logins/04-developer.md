---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: developer
status: draft
version: 2
author: dev
ai_drafted: false
---

# Build spec: Account Lockout on Repeated Failed Logins

## Files and services touched (paths)
- **`mini-auth-service`**:
  - `src/config.py`: Add lockout configuration settings (`MAX_FAILED_LOGIN_ATTEMPTS`, `ACCOUNT_LOCKOUT_DURATION_MINUTES`) to [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
  - `src/auth.py`: Implement in-memory state tracking, username normalization, lockout verification, failure incrementing, and reset logic within [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
  - `src/main.py`: Update route handling for `POST /api/v1/auth/login` in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] to handle lockout-specific exceptions and status responses.
  - `tests/test_auth.py` (or test suite directory): Add unit and integration tests covering lockout triggers, reset on success, lockout expiration, and edge cases.

## What to reuse
- **Existing JWT Signing & Token Creation**: Reuse `create_access_token` and [[kb:mini-auth-service/decisions/jwt-hs256-signing|HS256 signing]] routines in [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
- **Configuration Pattern**: Extend the existing configuration structure in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
- **OAuth2 Form Parsing**: Reuse FastAPI's `OAuth2PasswordRequestForm` dependency injection in [[kb:mini-auth-service/entities/presentation-main|src/main.py]].
- **Authentication Exceptions**: Reuse standard FastAPI `HTTPException(status_code=401, detail=...)` patterns for error responses.

## Tasks
1. - [ ] **Add Lockout Configuration**: In [[kb:mini-auth-service/entities/configuration-settings|src/config.py]], declare `MAX_FAILED_LOGIN_ATTEMPTS = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES = 15`.
2. - [ ] **Implement State Tracking & Normalization**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], define an in-memory thread-safe dictionary/structure mapping normalized (`username.strip().lower()`) identifiers to `{"failed_attempts": int, "locked_until": Optional[datetime]}`.
3. - [ ] **Implement Lockout Evaluation & Expiration Functions**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], implement helper methods to:
   - Check if an account is actively locked (`locked_until > datetime.utcnow()`).
   - Clean up / reset expired locks (`locked_until <= datetime.utcnow()`).
   - Increment failed login attempts and set `locked_until` timestamp when reaching 5 attempts.
   - Reset failed attempts and lockout state upon successful authentication.
4. - [ ] **Integrate Lockout Checks into Authentication Flow**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]] and [[kb:mini-auth-service/entities/presentation-main|src/main.py]], update the `POST /api/v1/auth/login` execution flow ([[kb:mini-auth-service/concepts/jwt-authentication-flow]]):
   - Check lockout status prior to verifying credentials. If locked, reject immediately with HTTP 401 `{"detail": "Account is temporarily locked. Try again later."}`.
   - If credentials fail, increment failure counter. If the 5th failure is reached, set 15-minute lock and return HTTP 401 `{"detail": "Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes."}`. Otherwise return HTTP 401 `{"detail": "Incorrect username or password"}`.
   - If credentials succeed, clear failed attempt counters and issue signed Bearer JWT.
5. - [ ] **Handle Non-Existent Usernames & Bounded Cache**: Ensure tracking operates safely for non-existent users without unhandled exceptions or unbounded memory growth.
6. - [ ] **Write Automated Unit and Integration Tests**: Create comprehensive tests verifying failure counting, lockout activation on the 5th attempt, lock enforcement against valid passwords during lock windows, expiration recovery after 15 minutes, and counter resets on success.

## Test plan
- **Unit Tests (`src/auth.py`)**:
  - `test_normalize_username`: Verify casing and leading/trailing whitespace normalization (e.g., `" Admin "` -> `"admin"`).
  - `test_failed_attempt_increment`: Verify failure counter increments from 1 up to 5 on consecutive failed attempts.
  - `test_lockout_trigger`: Verify that the 5th failed attempt sets `locked_until` to `utcnow() + 15 minutes`.
  - `test_counter_reset_on_success`: Verify `failed_attempts` is reset to 0 upon successful credential verification.
  - `test_lockout_expiration`: Verify that after `locked_until` timestamp elapses, the lock is cleared and login succeeds with valid credentials.
- **Integration Tests (`POST /api/v1/auth/login`)**:
  - `test_login_success_unlocked`: Submit valid credentials and verify HTTP 200 with Bearer JWT per [[kb:mini-auth-service/summaries/api-reference]].
  - `test_lockout_after_five_failures`: Submit 5 consecutive incorrect passwords for an account and assert HTTP 401 with lockout message on the 5th attempt.
  - `test_rejection_during_active_lockout`: Submit valid credentials for a locked account during the active 15-minute window; assert request is rejected with HTTP 401 and lock remains active.
  - `test_reset_before_threshold`: Submit 3 incorrect attempts, followed by 1 valid attempt (verifying 200 OK); submit another incorrect attempt and verify counter started over from 1.
  - `test_non_existent_username_attempt`: Submit 5 failed attempts on an unseeded username to ensure no server errors or unhandled exceptions.
  - `test_case_insensitive_lockout`: Verify that alternating cases (`ADMIN`, `admin`) aggregate towards the same 5-attempt threshold.
- **Regression Tests**:
  - `test_token_inspection_endpoint`: Ensure `GET /api/v1/auth/me` functionality and token validation remain unaffected.

## Rollout
1. Deploy the updated `mini-auth-service` application containing changes in `src/config.py`, `src/auth.py`, and `src/main.py`.
2. Ensure service starts up with in-memory lockout tracker initialized.
3. Validate endpoint health and run smoke tests against `POST /api/v1/auth/login` and `GET /api/v1/auth/me`.

## Open questions
- Should lockout state survive a service restart? (Today it would be in-memory.)

## Verification checklist
- [ ] Task 1: Lockout configuration constants defined in `src/config.py`.
- [ ] Task 2: In-memory state tracking and username normalization implemented in `src/auth.py`.
- [ ] Task 3: Lockout evaluation, expiration, incrementing, and reset logic implemented in `src/auth.py`.
- [ ] Task 4: Lockout checks integrated into `POST /api/v1/auth/login` in `src/main.py` and `src/auth.py`.
- [ ] Task 5: Non-existent usernames and cache bounds handled safely.
- [ ] Task 6: Unit and integration tests written and passing.
- [ ] Reused components (`create_access_token`, `OAuth2PasswordRequestForm`, JWT HS256 signing) not duplicated.
- [ ] API contracts for `POST /api/v1/auth/login` and `GET /api/v1/auth/me` remain compliant with [[kb:mini-auth-service/summaries/api-reference]].
