---
mission: SOL-1
title: 'Account Lockout on Repeated Failed Logins'
role: product
status: approved
version: 1
author: unknown
ai_drafted: false
approved_at: 2026-09-24T10:11:50Z
---

# Product spec: Account Lockout on Repeated Failed Logins

## Goal
Protect user accounts against brute-force and credential-stuffing attacks on the authentication endpoint (`POST /api/v1/auth/login`) by temporarily locking an account for 15 minutes after 5 consecutive failed password attempts.

## User stories
- **As a registered user**, I want my account protected from automated brute-force attacks so that unauthorized parties cannot compromise my credentials by repeatedly guessing passwords.
- **As a registered user who mistyped my password**, I want clear feedback when my account is locked and automatic restoration after 15 minutes so that I can regain access without requiring manual support intervention.
- **As a security engineer**, I want the authentication service to enforce a strict lockout threshold across the [[kb:mini-auth-service/concepts/jwt-authentication-flow|JWT authentication flow]] to mitigate automated credential discovery and server abuse.

## Acceptance criteria

### AC-1: Failed attempt tracking
- **Given** an account that is not currently locked,
- **When** a client submits invalid credentials for a username to `POST /api/v1/auth/login` as documented in [[kb:mini-auth-service/summaries/api-reference]],
- **Then** the service increments the consecutive failed login attempt counter for that username by 1 and returns an HTTP 401 Unauthorized response.

### AC-2: Lockout trigger on 5th consecutive failure
- **Given** an account with 4 consecutive failed login attempts,
- **When** a 5th consecutive failed login attempt is submitted to `POST /api/v1/auth/login`,
- **Then** the account is placed in a locked state for a duration of 15 minutes and the login attempt is rejected.

### AC-3: Request rejection during active lockout
- **Given** an account that is currently locked due to 5 consecutive failed attempts,
- **When** a login request is submitted for that username to `POST /api/v1/auth/login` before the 15-minute lockout period has elapsed (regardless of whether the submitted password is valid or invalid),
- **Then** the service rejects the authentication request and denies access token generation.

### AC-4: Failure counter reset on successful login
- **Given** an account with between 1 and 4 consecutive failed login attempts,
- **When** the user submits valid credentials to `POST /api/v1/auth/login`,
- **Then** the service returns a 200 OK response with a valid signed Bearer JWT and immediately resets the failed login attempt counter for that username to 0.

### AC-5: Automatic unlock after lockout expiration
- **Given** an account that has been locked for at least 15 minutes,
- **When** the user submits valid credentials to `POST /api/v1/auth/login`,
- **Then** the lock is treated as expired, the login succeeds with a 200 OK response and access token, and the failure counter is reset to 0.

## Edge cases
- **Non-existent usernames *(from the map)*:** When failed login attempts are made against non-existent usernames in the authentication store ([[kb:mini-auth-service/concepts/jwt-authentication-flow]]), the system must handle attempt tracking safely without unbounded memory leaks or leaking account existence.
- **Correct password submitted during active lockout:** If a user or attacker submits the correct password while the 15-minute lock window is active, the request must still be rejected and the lock duration must remain intact.
- **Rapid concurrent failed attempts:** If multiple failed login attempts arrive concurrently at `/api/v1/auth/login`, the counter must atomically increment to prevent race conditions that could allow more than 5 attempts before lockout engages.
- **Case sensitivity of usernames:** Tracking failed attempts must use normalized/consistent username casing so that attempts across varying cases (e.g., `Admin` vs `admin`) properly aggregate toward the same account threshold.

## Out of scope
- Multi-factor authentication (MFA) or CAPTCHA challenges during login.
- Administrative manual unlock or lock override endpoints.
- Email/SMS security alert notifications sent to users on account lockout.
- Permanent account deactivation or IP-level firewall blacklisting.
- Modifications to token inspection on `GET /api/v1/auth/me`.

## Success metric
100% of accounts undergoing 5 consecutive failed login attempts on `/api/v1/auth/login` are locked for 15 minutes, with zero successful logins permitted during active lockout windows.

## Priority (P1–P4) and why
**Priority: P1**
Protecting credential endpoints against brute-force attacks is a critical security vulnerability identified in [[kb:mini-auth-service/summaries/roadmap-and-improvements]]. Without lockout enforcement, the service is vulnerable to automated credential enumeration and unauthorized account compromise.

## Verification checklist
- [ ] AC-1: Consecutive failed login attempts are tracked and incremented on `POST /api/v1/auth/login`.
- [ ] AC-2: Account is placed into a 15-minute lockout upon the 5th consecutive failed login attempt.
- [ ] AC-3: Login attempts for locked accounts are rejected during the 15-minute lockout window regardless of password validity.
- [ ] AC-4: Successful login before reaching 5 failed attempts resets the failure counter to 0.
- [ ] AC-5: Account is automatically unlocked and accepts valid logins after the 15-minute lockout duration expires.
- [ ] Edge Case: Failed attempts on non-existent usernames do not cause errors or memory leaks *(from the map)*.
- [ ] Edge Case: Submitting valid credentials during an active lockout remains blocked until the 15 minutes expire.
- [ ] Edge Case: Concurrent failed requests do not bypass the 5-attempt lockout threshold due to race conditions.
- [ ] Edge Case: Username casing is handled consistently when tracking failed attempts.
- [ ] Metric: 100% of accounts with 5 consecutive failed attempts are locked for 15 minutes with zero unauthorized access bypasses.
