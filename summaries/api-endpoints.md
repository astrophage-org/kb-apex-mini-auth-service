<!-- anchor: src/main.py:L1-L100 sha:HEAD -->

# REST API Endpoints Specification

The `mini-auth-service` exposes an HTTP REST interface under the `/api/v1/auth` prefix, implemented in the [[entities/api-router]] (`src/main.py`). The API provides stateless user authentication, cryptographic token generation, and token validation.

For the higher-level design, see [[summaries/architecture-overview]].

---

## Endpoint Summary

| Method | Endpoint | Description | Auth Required | Request Format |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/login` | Authenticates user credentials and issues a signed JWT access token | No | `application/x-www-form-urlencoded` |
| `GET` | `/api/v1/auth/me` | Validates caller token and retrieves identity details | Yes (Bearer/Token) | Query / Dependency Injection |

---

## 1. Token Issuance / Login

Authenticates user credentials and returns a signed JSON Web Token (JWT).

* **HTTP Method**: `POST`
* **Route**: `/api/v1/auth/login`
* **Content-Type**: `application/x-www-form-urlencoded`
* **Underlying Logic**: [[concepts/credential-validation-flow]] via `authenticate_user()` and `create_access_token()` in [[entities/auth-engine]].

### Request Parameters

Request parameters are parsed via FastAPI's `OAuth2PasswordRequestForm` using [[concepts/dependency-injection]].

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `username` | `string` | Yes | Identity identifier (e.g., `"admin"`). |
| `password` | `string` | Yes | Plaintext password (e.g., `"secret123"`). |
| `grant_type` | `string` | No | OAuth2 grant type (defaults to `"password"`). |
| `scope` | `string` | No | OAuth2 optional scopes (default empty). |
| `client_id` | `string` | No | Optional OAuth2 client ID. |
| `client_secret` | `string` | No | Optional OAuth2 client secret. |

#### Example Request

```http
POST /api/v1/auth/login HTTP/1.1
Host: localhost:8000
Content-Type: application/x-www-form-urlencoded

username=admin&password=secret123
```

### Responses

#### `200 OK`
Authentication succeeded. Returns a signed [[entities/jwt-token-payload]] using `HS256` ([[decisions/symmetric-vs-asymmetric-signing]]) with a default expiration of 15 minutes.

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

#### `401 Unauthorized`
Credential verification failed.

```json
{
  "detail": "Incorrect username or password"
}
```

---

## 2. Get Current User Profile

Retrieves the identity details encoded within a validated access token.

* **HTTP Method**: `GET`
* **Route**: `/api/v1/auth/me`
* **Underlying Logic**: [[concepts/jwt-signing-verification]] via `verify_token()` in [[entities/auth-engine]].

### Request Parameters

Handled through FastAPI's [[concepts/dependency-injection]] binding `token_data: dict = Depends(verify_token)`.

| Parameter | Type | In | Required | Description |
| :--- | :--- | :--- | :--- | :--- |
| `token` | `string` | Query / Parameter | Yes | JWT access token string to decode and verify. |

#### Example Request

```http
GET /api/v1/auth/me?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... HTTP/1.1
Host: localhost:8000
```

### Responses

#### `200 OK`
Token verification succeeded. Returns the subject claim (`sub`) and user status.

```json
{
  "username": "admin",
  "status": "active"
}
```

#### `422 Unprocessable Entity` / Validation Failure
Occurs if the `token` parameter is missing or invalid according to FastAPI parameter requirements.

---

## Architectural & Security Notes

* **Stateless Session Design**: Endpoints do not persist or query session tokens in memory or database storage; all claims are verified statelessly via signature integrity ([[decisions/stateless-vs-stateful-sessions]], [[concepts/token-based-authentication]]).
* **Cryptographic Configuration**: Token signing algorithm (`HS256`) and secret keys are resolved from the [[entities/settings]] singleton (`src/config.py`).
* **Transport Modernization**: In the current implementation, `/api/v1/auth/me` uses direct dependency parameter resolution. Production upgrades involve integrating `OAuth2PasswordBearer` to automatically extract Bearer tokens from standard `Authorization: Bearer <token>` HTTP headers.
* **Credential Persistence**: Default static verification (`admin`/`secret123`) is designed to be upgraded to database lookups with secure hashing ([[decisions/credential-storage-hashing]]).