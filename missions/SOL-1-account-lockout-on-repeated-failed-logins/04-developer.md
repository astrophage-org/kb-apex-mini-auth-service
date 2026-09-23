---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: developer
status: draft
version: 8
author: dev
ai_drafted: false
---

# Build spec: Account Lockout on Repeated Failed Logins

## Files and services touched (paths)
- **`mini-auth-service`**:
  - `src/config.py`: Add lockout configuration constants (`MAX_FAILED_LOGIN_ATTEMPTS: int = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15`) to the settings class in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
  - `src/auth.py`: Implement in-memory thread-safe state tracking, username normalization (`username.strip().lower()`), lockout status evaluation, failure counter increments, lock timestamp management, and counter reset operations within [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
  - `src/main.py`: Update the `POST /api/v1/auth/login` presentation route in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] to enforce lockout checks during credential ingestion and map lockout domain exceptions to standard HTTP 401 Unauthorized responses.
  - `tests/test_auth.py`: Add comprehensive unit, integration, and regression test suites covering failed attempt tracking, lockout engagement on the 5th attempt, access rejection during active lockouts (with both correct and incorrect passwords), automatic unlock upon lockout expiration, counter reset on successful login, case-insensitive username normalization, concurrent attempt handling, and non-existent user handling.

## What to reuse
- **Existing JWT Signing & Token Creation**: Reuse `create_access_token` and the [[kb:mini-auth-service/decisions/jwt-hs256-signing|JWT HS256 signing]] implementation in [[kb:mini-auth-service/entities/auth-domain|src/auth.py]] to issue signed Bearer tokens on successful authentication.
- **Configuration Pattern**: Extend the existing configuration singleton pattern in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
- **OAuth2 Form Parsing**: Reuse FastAPI's `OAuth2PasswordRequestForm` dependency injection in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] for standard `application/x-www-form-urlencoded` credential ingestion.
- **HTTP Exception Model**: Reuse FastAPI's standard `HTTPException(status_code=401, detail=...)` pattern to ensure uniform error payload structures (`{"detail": "..."}`) across the [[kb:mini-auth-service/concepts/jwt-authentication-flow|JWT authentication flow]].
- **Claims & Token Inspection**: Retain existing `GET /api/v1/auth/me` token inspection flow and `verify_token` guard as documented in [[kb:mini-auth-service/summaries/api-reference]].

## Tasks
1. - [ ] **Add Lockout Configuration Constants**: In [[kb:mini-auth-service/entities/configuration-settings|src/config.py]], add `MAX_FAILED_LOGIN_ATTEMPTS: int = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15` to the settings class.
2. - [ ] **Implement State Tracking Structure & Normalization**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], define an in-memory thread-safe dictionary mapping normalized lowercased usernames (`username.strip().lower()`) to lockout metadata: `{"failed_attempts": int, "locked_until": Optional[datetime]}`.
3. - [ ] **Implement Lockout Evaluation & Mutator Functions**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], implement domain helper functions:
   - `is_account_locked(username: str) -> bool`: Checks if `locked_until` exists and is strictly greater than `datetime.utcnow()`. Automatically clears expired locks (`locked_until <= datetime.utcnow()`) and resets `failed_attempts` to 0 when an expired lock is detected.
   - `record_failed_attempt(username: str) -> bool`: Increments `failed_attempts` by 1. If `failed_attempts >= MAX_FAILED_LOGIN_ATTEMPTS`, sets `locked_until = datetime.utcnow() + timedelta(minutes=ACCOUNT_LOCKOUT_DURATION_MINUTES)` and returns `True` (locked), otherwise `False`.
   - `reset_failed_attempts(username: str) -> None`: Clears state for the username, resetting `failed_attempts = 0` and `locked_until = None`.
4. - [ ] **Integrate Lockout Flow into `authenticate_user` / Presentation Handler**: In [[kb:mini-auth-service/entities/presentation-main|src/main.py]] and [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], wire the [[kb:mini-auth-service/concepts/jwt-authentication-flow|JWT authentication flow]] to:
   - Normalize the submitted username (`username.strip().lower()`).
   - Check `is_account_locked(username)` prior to password evaluation. If locked, immediately raise `HTTPException(status_code=401, detail="Account is temporarily locked. Try again later.")`.
   - If password validation fails (including for non-existent users), invoke `record_failed_attempt(username)`. If the lockout threshold is reached (`failed_attempts >= MAX_FAILED_LOGIN_ATTEMPTS`), raise `HTTPException(status_code=401, detail="Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes.")`; otherwise raise `HTTPException(status_code=401, detail="Incorrect username or password")`.
   - If credentials authenticate successfully, call `reset_failed_attempts(username)` and issue the signed JWT access token per [[kb:mini-auth-service/decisions/jwt-hs256-signing]].
5. - [ ] **Implement Bounded Memory / Non-Existent User Safeguards**: Ensure failed attempts against non-existent or randomized usernames are recorded in a bounded cache (with max-size or LRU eviction) to prevent memory exhaustion without leaking account existence via differentiated responses or timings.
6. - [ ] **Implement Automated Unit and Integration Test Suite**: In `tests/test_auth.py`, implement test cases covering threshold locking, active lockout rejection, lockout expiration, counter resets on success, case insensitivity, concurrent attempts, and non-existent username handling.

## Test plan
- **Unit Tests (`tests/test_auth.py`)**:
  - `test_username_normalization`: Verify leading/trailing whitespace and mixed casing are normalized (e.g., `" Admin "` -> `"admin"`).
  - `test_increment_failed_attempts`: Verify `failed_attempts` increments sequentially on failed attempts.
  - `test_lockout_trigger_at_five`: Verify that exactly 5 consecutive failures triggers `locked_until` set to 15 minutes in the future.
  - `test_reset_on_successful_authentication`: Verify that calling `reset_failed_attempts` sets `failed_attempts` to 0 and clears `locked_until`.
  - `test_lockout_expiration_unlock`: Mock or advance time past 15 minutes and verify `is_account_locked` returns `False` and clears expired state.
- **Integration Tests (`POST /api/v1/auth/login`)**:
  - `test_login_success_unlocked`: Submit valid credentials (`admin` / `secret123`) and assert HTTP 200 with Bearer JWT token per [[kb:mini-auth-service/summaries/api-reference]].
  - `test_five_failed_attempts_locks_account`: Submit 5 consecutive incorrect passwords; assert the 5th attempt returns HTTP 401 with lockout message `"Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes."`.
  - `test_rejection_during_active_lockout_with_valid_credentials`: Submit valid credentials while the account is locked; assert HTTP 401 `"Account is temporarily locked. Try again later."` and verify access token is not issued.
  - `test_rejection_during_active_lockout_with_invalid_credentials`: Submit invalid credentials while the account is locked; assert HTTP 401 `"Account is temporarily locked. Try again later."` and verify lockout expiration timestamp does not reset.
  - `test_successful_login_resets_failure_counter`: Submit 4 incorrect attempts, followed by 1 valid login (assert HTTP 200); submit 1 incorrect attempt afterwards and assert failure counter is 1 rather than triggering lockout.
  - `test_lockout_expiration_allows_login`: Lock account after 5 attempts, freeze/advance time by 15 minutes + 1 second, submit valid credentials, assert HTTP 200 and access token issuance.
  - `test_case_insensitive_username_lockout`: Submit alternating cases (e.g., `admin`, `ADMIN`, `Admin`) with wrong passwords and verify lockout triggers on the 5th attempt across casing variations.
  - `test_non_existent_username_failure_handling`: Submit 5 failed attempts for a non-existent username; verify consistent HTTP 401 responses and lockout triggering without runtime errors or account existence leakage.
  - `test_concurrent_failed_logins`: Simulate concurrent failed login requests to ensure atomic incrementing and thread safety under load.
- **Regression Tests**:
  - `test_me_endpoint_unaffected`: Verify valid tokens issued after counter reset continue to decode successfully on `GET /api/v1/auth/me` per [[kb:mini-auth-service/summaries/api-reference]].

## Rollout
1. Deploy the updated `mini-auth-service` application with changes to `src/config.py`, `src/auth.py`, and `src/main.py`.
2. Verify service startup and configuration loading of `MAX_FAILED_LOGIN_ATTEMPTS` and `ACCOUNT_LOCKOUT_DURATION_MINUTES`.
3. Execute post-deployment smoke tests against `POST /api/v1/auth/login` and `GET /api/v1/auth/me`.

## Open questions
- Should lockout state persist across process restarts? (State is currently maintained in-memory consistent with existing architecture; Redis/DB persistence is documented on the roadmap in [[kb:mini-auth-service/summaries/roadmap-and-improvements]]).

## Monitoring
- Alert when lockouts spike above normal.

## Verification checklist
- [ ] Task 1: Lockout configuration constants (`MAX_FAILED_LOGIN_ATTEMPTS = 5`, `ACCOUNT_LOCKOUT_DURATION_MINUTES = 15`) added to `src/config.py`.
- [ ] Task 2: In-memory state tracking dictionary and username normalization implemented in `src/auth.py`.
- [ ] Task 3: Lockout evaluation (`is_account_locked`), failure recording (`record_failed_attempt`), and counter reset (`reset_failed_attempts`) implemented in `src/auth.py`.
- [ ] Task 4: Lockout checks integrated into `POST /api/v1/auth/login` in `src/main.py` and `src/auth.py` with standard HTTP 401 exceptions.
- [ ] Task 5: Non-existent usernames and bounded memory handling implemented without account enumeration leaks.
- [ ] Task 6: Unit, integration, and regression test suites in `tests/test_auth.py` pass.
- [ ] Reused components (`create_access_token`, `OAuth2PasswordRequestForm`, JWT HS256 signing) are not duplicated.
- [ ] API contracts for `POST /api/v1/auth/login` and `GET /api/v1/auth/me` remain compliant with [[kb:mini-auth-service/summaries/api-reference]].
