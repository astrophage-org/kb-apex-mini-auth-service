<!-- anchor: src/main.py:L1-L100 sha:HEAD -->

# API Reference: Mini Auth Service

This document specifies the REST API endpoints, request parameters, payload structures, status codes, and response contracts for the **Mini Auth Service**. 

The HTTP presentation layer is implemented in [[entities/presentation-main]] using **FastAPI**, delegating cryptographic and credential operations to [[entities/auth-domain]] and runtime parameters to [[entities/configuration-settings]].

For architectural details, see [[summaries/architecture-overview]]. For details on the authentication mechanism, see [[concepts/jwt-authentication-flow]].

---

## 1. API Overview

The service exposes endpoints under the `/api/v1/auth` prefix to handle user login and claim inspection.

| Method | Endpoint | Auth Required | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/login` | None | Authenticates credentials and returns a signed Bearer JWT |
| `GET` | `/api/v1/auth/me` | Bearer Token | Decodes token claims and returns current user status |

---

## 2. Endpoints

### 2.1. Obtain Access Token

Authenticates a client using standard OAuth2 Password credentials and issues a signed JSON Web Token (JWT) with a default 15-minute expiration.

- **URL:** `/api/v1/auth/login`
- **Method:** `POST`
- **Authentication:** None
- **Content-Type:** `application/x-www-form-urlencoded`
- **Handler Implementation:** `login()` in [[entities/presentation-main]]
- **Domain Flow:** [[concepts/jwt-authentication-flow]]

#### Request Parameters

Request parameters are parsed via FastAPI's `OAuth2PasswordRequestForm` dependency:

| Field | In | Type | Required | Description |
| :--- | :--- | :--- | :--- | :--- |
| `username` | Form Body | `string` | Yes | Identity identifier (e.g., `admin`) |
| `password` | Form Body | `string` | Yes | Identity plaintext password (e.g., `secret123`) |
| `grant_type` | Form Body | `string` | No | OAuth2 grant type (default: `password`) |
| `scope` | Form Body | `string` | No | Optional OAuth2 scopes (space-delimited) |
| `client_id` | Form Body | `string` | No | Optional client identifier |
| `client_secret` | Form Body | `string` | No | Optional client secret |

#### Example Request

```http
POST /api/v1/auth/login HTTP/1.1
Host: localhost:8000
Content-Type: application/x-www-form-urlencoded

username=admin&password=secret123
```

#### Responses

##### 200 OK
Returns a signed [[decisions/jwt-hs256-signing|HS256]] access token and the token type.

- **Content-Type:** `application/json`
- **Response Schema:**
  ```json
  {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer"
  }
  ```

| Field | Type | Description |
| :--- | :--- | :--- |
| `access_token` | `string` | Encoded JWT carrying user identity claims and expiration (`exp`) |
| `token_type` | `string` | Token scheme identifier; always `"bearer"` |

##### 401 Unauthorized
Returned when `authenticate_user()` fails to validate credentials.

- **Content-Type:** `application/json`
- **Response Body:**
  ```json
  {
    "detail": "Incorrect username or password"
  }
  ```

---

### 2.2. Get Current User

Inspects the caller's identity by validating and decoding the provided Bearer token.

- **URL:** `/api/v1/auth/me`
- **Method:** `GET`
- **Authentication:** Bearer Token via `verify_token` dependency guard
- **Content-Type:** None
- **Handler Implementation:** `get_current_user()` in [[entities/presentation-main]]
- **Verification Guard:** [[concepts/token-verification-guard]]

#### Request Headers

| Header | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `Authorization` | `string` | Yes | Standard Bearer token header (`Bearer <access_token>`) |

#### Example Request

```http
GET /api/v1/auth/me HTTP/1.1
Host: localhost:8000
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### Responses

##### 200 OK
Returned when the token signature is valid, non-expired, and decoded payload is passed into the route handler.

- **Content-Type:** `application/json`
- **Response Schema:**
  ```json
  {
    "username": "admin",
    "status": "active"
  }
  ```

| Field | Type | Description |
| :--- | :--- | :--- |
| `username` | `string` | Identity claim extracted from the `sub` claim of the JWT |
| `status` | `string` | Account state (currently static `"active"`) |

##### 401 Unauthorized / Token Error
If the token signature fails or the token has expired, `verify_token` returns `None` or triggers an authorization failure. For implementation improvements regarding error granularity, see [[summaries/roadmap-and-improvements]].

---

## 3. Presentation Architecture & Dependency Injection

The presentation layer utilizes [[decisions/fastapi-dependency-injection|FastAPI Dependency Injection]] to decouple protocol parsing from security validation:

```mermaid
graph TD
    Client[HTTP Client] -->|POST /api/v1/auth/login| LoginRoute[main.py: login]
    Client -->|GET /api/v1/auth/me| MeRoute[main.py: get_current_user]

    subgraph DI Guards
        OAuthForm[Depends OAuth2PasswordRequestForm] --> LoginRoute
        TokenGuard[Depends verify_token] --> MeRoute
    end

    subgraph Domain Logic
        LoginRoute --> AuthUser[auth.py: authenticate_user]
        LoginRoute --> CreateToken[auth.py: create_access_token]
        TokenGuard --> VerifyToken[auth.py: verify_token]
    end

    subgraph Configuration
        CreateToken -.-> Settings[config.py: settings]
        VerifyToken -.-> Settings
    end
```

By relying on stateless JWT payloads, these endpoints require no server-side session stores, enabling horizontal scalability across instances as detailed in [[concepts/stateless-session-pattern]].

---

## 4. Summary of Schemas

### Token Response (`POST /api/v1/auth/login`)
```typescript
interface TokenResponse {
  access_token: string;
  token_type: "bearer";
}
```

### User Profile Response (`GET /api/v1/auth/me`)
```typescript
interface UserProfileResponse {
  username: string;
  status: "active" | string;
}
```

### Error Response
```typescript
interface HTTPErrorResponse {
  detail: string;
}
```

---

## Related Documentation

- [[index]] - Main codebase entry point and component map
- [[entities/presentation-main]] - Presentation route handlers source code
- [[entities/auth-domain]] - Token creation and verification logic
- [[concepts/jwt-authentication-flow]] - Complete token issuance sequence
- [[concepts/token-verification-guard]] - Route guard evaluation logic
- [[summaries/roadmap-and-improvements]] - Future OpenAPI schema enhancements and bearer extraction updates