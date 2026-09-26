---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: engineering
status: ai_drafted
version: 3
author: Sol
ai_drafted: true
---

# Engineering design: Account Lockout on Repeated Failed Logins

## Applications changing (and why each)
- **`mini-auth-service`**: Implement account lockout tracking, threshold evaluation, and lockout duration validation within the authentication domain ([[kb:mini-auth-service/entities/auth-domain|src/auth.py]]) and presentation endpoint handlers ([[kb:mini-auth-service/entities/presentation-main|src/main.py]]). Configuration constants for lockout limits and durations will be introduced in [[kb:mini-auth-service/entities/configuration-settings|src/config.py]].

## Approach

### 1. In-Memory Lockout State Management
In alignment with the current architecture of `mini-auth-service`, attempt state will be managed in-memory within [[kb:mini-auth-service/entities/auth-domain|src/auth.py]]. A thread-safe state container will track failed attempts and lockout timestamps keyed by normalized (lowercased) username:

```python
# State schema per username
{
    "failed_attempts": int,
    "locked_until": Optional[datetime]
}
```

### 2. Configuration Parameters
Add lockout settings to [[kb:mini-auth-service/entities/configuration-settings|src/config.py]]:
- `MAX_FAILED_LOGIN_ATTEMPTS: int = 5`
- `ACCOUNT_LOCKOUT_DURATION_MINUTES: int = 15`

### 3. Execution Flow in `authenticate_user` / `POST /api/v1/auth/login`
When a client submits credentials to `POST /api/v1/auth/login` ([[kb:mini-auth-service/concepts/jwt-authentication-flow]]):
1. **Username Normalization & Duplicate Identity Resolution**: Normalize `form_data.username.strip().lower()` to canonical form to ensure duplicate representations (e.g., `Admin` vs `admin`) map to the exact same identity key.
2. **Lockout Verification**: Check whether the user currently has an active lockout (`locked_until > datetime.utcnow()`).
   - If active: Immediately abort authentication and raise an `HTTPException(status_code=401, detail="Account is temporarily locked. Try again later.")`. The lockout expiration remains unchanged.
   - If `locked_until` exists but has elapsed (`locked_until <= datetime.utcnow()`): Reset `failed_attempts = 0` and clear `locked_until`.
3. **Credential Evaluation**: Execute standard password verification against the canonical user identity.
4. **On Authentication Success**:
   - Reset `failed_attempts = 0` and clear `locked_until`.
   - Issue signed HS256 JWT per [[kb:mini-auth-service/decisions/jwt-hs256-signing]].
5. **On Authentication Failure**:
   - Increment `failed_attempts` by 1.
   - If `failed_attempts >= MAX_FAILED_LOGIN_ATTEMPTS`:
     - Set `locked_until = datetime.utcnow() + timedelta(minutes=ACCOUNT_LOCKOUT_DURATION_MINUTES)`.
     - Raise `HTTPException(status_code=401, detail="Account locked due to 5 consecutive failed login attempts. Try again in 15 minutes.")`.
   - Otherwise, raise standard `HTTPException(status_code=401, detail="Incorrect username or password")`.

### 4. Memory Bounds & Non-existent User Handling
To mitigate memory exhaustion from automated brute-force attacks cycling randomized usernames:
- Failed attempts against unknown usernames are tracked in the same bounded cache with a TTL eviction policy or max-size LRU eviction mechanism.

### 5. Duplicate Account Check & Identity Resolution
To enforce account integrity and prevent split-state lockout bypasses:
- Account lookup and lockout tracking enforce strict uniqueness on normalized username keys.
- Credential evaluations check for ambiguous or duplicate account identities in the user store and resolve to a single canonical account record, ensuring that concurrent attempts across case variants cannot bypass lockout thresholds.

## Contracts affected

| Contract | Owner app | Consumers | Unchanged / Additive / Breaking |
| :--- | :--- | :--- | :--- |
| `POST /api/v1/auth/login` | `mini-auth-service` | External API clients, web clients | Additive (Returns HTTP 401 with lockout-specific error message when threshold is reached or lockout is active; existing request form schema and 200 OK JWT payload remain untouched) |
| `GET /api/v1/auth/me` | `mini-auth-service` | Authenticated clients, microservices | Unchanged |

## Must not break
- **Standard Authentication Flow**: Users with valid credentials on an unlocked account must continue to receive valid signed JWT tokens with standard 15-minute expiry per [[kb:mini-auth-service/concepts/jwt-authentication-flow]].
- **Token Inspection Contract**: Valid tokens must continue to decode successfully on `GET /api/v1/auth/me` as defined in [[kb:mini-auth-service/summaries/api-reference]].
- **OAuth2 Form Compliance**: Ingestion of `OAuth2PasswordRequestForm` (`application/x-www-form-urlencoded`) in [[kb:mini-auth-service/entities/presentation-main|src/main.py]] must remain compliant.

## Guardrails and ADRs in play
- **[[kb:mini-auth-service/decisions/jwt-hs256-signing|JWT HS256 Signing]]**: Token issuance and signing invariants must remain intact.
- **FastAPI HTTP Exception Model**: Error responses must adhere to standard FastAPI error structures (`{"detail": "..."}`) with appropriate HTTP 401 Unauthorized status codes.
- **Stateless Microservice Roadmap**: In-memory state serves as the baseline implementation; interfaces should encapsulate lockout checks so that future migrations to Redis or database persistence (noted in [[kb:mini-auth-service/summaries/roadmap-and-improvements]]) can occur without altering route contracts.

## Test strategy

| Level | What it proves | AC or contract covered |
| :--- | :--- | :--- |
| **Unit** | Lockout tracking logic in `src/auth.py`: failed attempt incrementing, threshold triggering, lockout expiration arithmetic, and counter reset on success | AC-1, AC-2, AC-4, AC-5, AC-6 |
| **Unit** | Username casing normalization, duplicate account check/identity resolution, and bounded memory eviction on non-existent usernames | Edge cases: username casing, non-existent usernames, duplicate account check |
| **Integration** | `POST /api/v1/auth/login` endpoint returns HTTP 401 with lockout message after 5 failed attempts, blocks subsequent valid/invalid attempts during lockout window, and unlocks after expiration | AC-1, AC-2, AC-3, AC-5 |
| **Contract** | Request form schema and HTTP response structures for `POST /api/v1/auth/login` remain backward-compatible; existing token verification on `GET /api/v1/auth/me` remains unchanged | `POST /api/v1/auth/login`, `GET /api/v1/auth/me` |

## Rollout and rollback

- **Deployment Ordering**: Standalone single-service deployment for `mini-auth-service`. No dependent cross-service deployment ordering required.
- **Feature Flags & Config**: Lockout thresholds (`MAX_FAILED_LOGIN_ATTEMPTS`, `ACCOUNT_LOCKOUT_DURATION_MINUTES`) configured in `src/config.py` with safe defaults (5 attempts, 15 minutes).
- **Data Migration**: None. Attempt tracking is managed in-memory with zero database schema migrations or backfills required.
- **Rollback Procedure**: Revert `mini-auth-service` release to the previous commit/container image. In-memory lockout state will be cleared on process restart with immediate return to un-throttled login behavior.

## Risks
- **Account Lockout Denial-of-Service (DoS)**: An attacker can deliberately lock a known user's account by sending 5 invalid password attempts. *(Mitigation: Lockout is temporary at 15 minutes; IP rate limiting is planned in future roadmap phases per [[kb:mini-auth-service/summaries/roadmap-and-improvements]])*.
- **In-Memory Volatility**: Restarting or horizontally scaling the `mini-auth-service` process clears or splits the lockout state. *(Mitigation: Acceptable for the current single-instance architecture prior to distributed cache introduction)*.

## Verification checklist

- [ ] Consecutive failed login attempts increment by 1 on `POST /api/v1/auth/login`.
- [ ] An account is placed in locked status upon the 5th consecutive failed attempt.
- [ ] Login requests for a locked account are rejected during the 15-minute lockout period, even when the correct password is submitted.
- [ ] A successful login before reaching 5 failures resets the failure counter to 0.
- [ ] After 15 minutes elapse from the lockout time, valid credentials successfully authenticate and reset the counter.
- [ ] Duplicate account check and username normalization ensure attempts across case variants map to the same canonical account identity.
- [ ] The request/response schema for `POST /api/v1/auth/login` remains additive and does not break existing clients.
- [ ] The `GET /api/v1/auth/me` endpoint contract is unchanged.
- [ ] JWT token generation continues to follow [[kb:mini-auth-service/decisions/jwt-hs256-signing]].
