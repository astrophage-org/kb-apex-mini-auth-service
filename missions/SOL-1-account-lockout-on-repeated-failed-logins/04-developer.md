---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: developer
status: draft
version: 5
author: dev
ai_drafted: false
---

# Build spec: Account Lockout on Repeated Failed Logins

## Files and services touched (paths)
- **`mini-auth-service`**:
  - `src/config.py`: Add lockout configuration parameters (`MAX_FAILED_LOGIN_ATTEMPTS: int = 5`, `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15`) to [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
  - `src/auth.py`: Implement in-memory state tracking, username normalization (`username.strip().lower()`), lockout status evaluation, failure counter increments, and reset logic within [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
  - `src/main.py`: Update the `POST /api/v1/auth/login` endpoint handler in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] to integrate lockout pre-checks and map lockout exceptions to standard HTTP 401 Unauthorized responses.
  - `tests/test_auth.py`: Add comprehensive unit and integration test suites covering lockout enforcement, counter resets, lockout expiration, casing consistency, and non-existent username handling.

## What to reuse
- **Existing JWT Signing & Token Creation**: Reuse `create_access_token` and the [[kb:mini-auth-service/decisions/jwt-hs256-signing|JWT HS256 signing]] implementation in [[kb:mini-auth-service/entities/auth-domain|src/auth.py]].
- **Configuration Pattern**: Extend the existing static configuration pattern in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].
- **OAuth2 Form Parsing**: Reuse FastAPI's `OAuth2PasswordRequestForm` dependency injection in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] without changing input schema or content type (`application/x-www-form-urlencoded`).
- **HTTP Exception Handling**: Reuse FastAPI's standard `HTTPException(status_code=401, detail=...)` structure for consistent error serialization across the [[kb:mini-auth-service/concepts/jwt-authentication-flow|JWT authentication flow]].

## Tasks
1. - [ ] **Add Lockout Configuration**: In [[kb:mini-auth-service/entities/configuration-settings|src/config.py]], declare `MAX_FAILED_LOGIN_ATTEMPTS = 5` and `ACCOUNT_LOCKOUT_DURATION_MINUTES = 15` on the configuration settings object.
2. - [ ] **Implement State Tracking & Normalization**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], define an in-memory thread-safe dictionary/structure mapping normalized (`username.strip().lower()`) identifiers to state records (`{"failed_attempts": int, "locked_until": Optional[datetime]}`).
3. - [ ] **Implement Lockout Evaluation & Helper Functions**: In [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], implement helper methods to:
   - Check if an account is actively locked (`locked_until > datetime.utcnow()`).
   - Clean up / reset expired locks (`locked_until <= datetime.utcnow()`).
   - Increment failed login attempts and assign `locked_until = datetime.utcnow() + timedelta(minutes=ACCOUNT_LOCKOUT_DURATION_MINUTES)` upon reaching the 5th attempt.
   - Reset `failed_attempts` and clear `locked_until` upon successful credential authentication.
4. - [ ] **Integrate Lockout Checks into Authentication Flow**: In [[kb:mini-auth-service/entities/presentation-main|src/main.py]] and [[kb:mini-auth-service/entities/auth-domain|src/auth.py]], update the execution flow of `POST /api/v1/auth/login` ([[kb:mini-auth-service/concepts/jwt-authentication-flow]]):
   - Normalize the submitted username (`username.strip().lower()`).
   - Evaluate lockout status prior to checking passwords. If an active lock exists, immediately raise `HTTPException(status_code=401, detail="Account is temporarily locked. Try again later.")`.
   - If credentials fail authentication, increment the failure counter. If the 5th failure threshold is reached, set the 15-minute lock and raise `HTTPException(status_code=401, detail="Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes.")`. Otherwise, raise `HTTPException(status_code=401, detail="Incorrect username or password")`.
   - If credentials succeed, clear failed attempt counters and issue a signed Bearer JWT token per [[kb:mini-auth-service/decisions/jwt-hs256-signing]].
5. - [ ] **Handle Non-Existent Usernames & Bounded Cache**: Ensure failed attempts against non-existent usernames follow the same failure tracking mechanism without leaking user existence or causing unbounded memory growth.
6. - [ ] **Write Automated Unit and Integration Tests**: Create automated tests in `tests/test_auth.py` verifying failure counting, lockout activation on the 5th attempt, lock enforcement against valid passwords during lock windows, expiration recovery after 15 minutes, and counter resets on success.

## Test plan
- **Unit Tests (`tests/test_auth.py` / `src/auth.py`)**:
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
  - `test_token_inspection_endpoint`: Ensure `GET /api/v1/auth/me` functionality and token validation remain unaffected as specified in [[kb:mini-auth-service/summaries/api-reference]].

## Rollout
1. Deploy the updated `mini-auth-service` application containing changes in `src/config.py`, `src/auth.py`, and `src/main.py`.
2. Ensure service starts up with in-memory lockout tracker initialized.
3. Validate endpoint health and run smoke tests against `POST /api/v1/auth/login` and `GET /api/v1/auth/me`.

## Open questions
- Should lockout state survive a service restart? (Currently in-memory per service architecture; Redis/DB persistence is planned in [[kb:mini-auth-service/summaries/roadmap-and-improvements]]).

## Verification checklist
- [ ] Task 1: Lockout configuration constants defined in `src/config.py`.
- [ ] Task 2: In-memory state tracking and username normalization implemented in `src/auth.py`.
- [ ] Task 3: Lockout evaluation, expiration, incrementing, and reset logic implemented in `src/auth.py`.
- [ ] Task 4: Lockout checks integrated into `POST /api/v1/auth/login` in `src/main.py` and `src/auth.py`.
- [ ] Task 5: Non-existent usernames and cache bounds handled safely.
- [ ] Task 6: Unit and integration tests written and passing.
- [ ] Reused components (`create_access_token`, `OAuth2PasswordRequestForm`, JWT HS256 signing) not duplicated.
- [ ] API contracts for `POST /api/v1/auth/login` and `GET /api/v1/auth/me` remain compliant with [[kb:mini-auth-service/summaries/api-reference]].
