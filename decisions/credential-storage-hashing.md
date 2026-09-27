# ADR: Transitioning In-Memory Credentials to Database Storage with Argon2/bcrypt

## Status
Accepted (Planned for Production Milestone)

---

## Context
In the initial prototype of `mini-auth-service`, user validation is hardcoded directly inside `authenticate_user()` in [[entities/auth-engine]]:

```python
def authenticate_user(username: str, password: str):
    if username == "admin" and password == "secret123":
        return {"username": "admin", "role": "admin"}
    return None
```

While [[entities/settings]] defines a `DATABASE_URL` parameter (`postgresql://localhost:5432/auth_db`), the service does not yet establish database connections or query persistent stores. 

This current approach has severe security and architectural limitations:
1. **Plaintext Credential Exposure**: Passwords exist in plaintext in source code and memory during runtime execution.
2. **Lack of Cryptographic Salting and Hashing**: Susceptible to credential leakage, memory inspection, and timing attacks.
3. **No Multi-User or Dynamic Identity Support**: Cannot scale to support user registration, role updates, or dynamic user lifecycle management.
4. **Tight Coupling**: Identity data is coupled directly with authentication logic in [[entities/auth-engine]] rather than decoupled via a database repository layer.

To transition to a production-ready authentication service, credentials must be stored securely in a relational database with modern, slow, salted, one-way cryptographic hash functions.

---

## Decision

We will transition the credential storage and verification mechanism from in-memory static comparison to a database-backed persistence layer utilizing **Argon2id** (with **bcrypt** as an acceptable fallback/secondary algorithm) for password hashing.

### 1. Selected Hashing Algorithm: Argon2id
* **Primary Algorithm**: **Argon2id** (RFC 9106), the winner of the Password Hashing Competition (PHC).
* **Rationale**:
  * **Memory-Hard**: Resistant to GPU/ASIC-accelerated brute-force and dictionary attacks.
  * **Side-Channel Resistant**: Argon2id combines Argon2i (resistant against side-channel attacks) and Argon2d (resistant against GPU cracking).
  * **Configurable Cost Factors**: Supports independent tuning of time cost ($t$), memory cost ($m$), and parallelism ($p$).
* **Secondary / Legacy Compatibility**: `bcrypt` (with work factor $\ge 12$) will be supported as a transitional algorithm via hashing libraries such as `passlib` or `argon2-cffi`.

### 2. Architectural Target State

```
+-------------------------------------------------------------+
|                  API Layer (src/main.py)                    |
|             [[entities/api-router]]                         |
+------------------------------+------------------------------+
                               |
              1. (form_data, db_session) via [[concepts/dependency-injection]]
                               v
+-------------------------------------------------------------+
|              Auth Engine (src/auth.py)                      |
|             [[entities/auth-engine]]                        |
|   - verify_password(plain, hashed)                          |
|   - get_password_hash(plain)                                |
+---------------+-----------------------------+---------------+
                |                             |
    2. Query user hash             3. Verify with Argon2id
                v                             v
+-------------------------------+  +--------------------------+
|      Database Layer           |  | Password Hashing Engine  |
|  (PostgreSQL + SQLAlchemy)    |  | (Passlib / Argon2-Cffi)  |
|  - Table: `users`             |  +--------------------------+
+-------------------------------+
```

---

## Implementation Roadmap

### Phase 1: Cryptographic Hashing Infrastructure
Introduce a dedicated password utility module in `src/auth.py` using `passlib.context.CryptContext` or `pwdlib`:

```python
from passlib.context import CryptContext

pwd_context = CryptContext(
    schemes=["argon2", "bcrypt"],
    deprecated="auto",
    argon2__memory_cost=65536,  # 64 MB
    argon2__time_cost=3,
    argon2__parallelism=4,
)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)
```

### Phase 2: Database Layer & Entity Modeling
1. Integrate **SQLAlchemy** (async) or **SQLModel** along with **Alembic** migrations.
2. Read the database connection string dynamically from [[entities/settings]] via `settings.DATABASE_URL` (see [[decisions/environment-configuration-management]]).
3. Define the `User` relational model:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(64) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    role VARCHAR(32) NOT NULL DEFAULT 'user',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
```

### Phase 3: Refactoring the Credential Validation Flow
Refactor `authenticate_user()` to receive a database session via [[concepts/dependency-injection]] and execute constant-time password verification:

```python
async def authenticate_user(db: AsyncSession, username: str, password: str):
    user = await get_user_by_username(db, username)
    if not user:
        # Perform dummy hash verification to prevent user-enumeration timing attacks
        pwd_context.dummy_verify()
        return None
    if not verify_password(password, user.hashed_password):
        return None
    return user
```

Update [[summaries/api-endpoints]] and [[concepts/credential-validation-flow]] to integrate the new database-backed validation sequence.

### Phase 4: Transparent Hash Upgrades
Implement automatic rehashing inside the login workflow: if a user authenticates with an outdated hashing parameter or a deprecated scheme (e.g., legacy `bcrypt` instead of `Argon2id`), `pwd_context.needs_update(user.hashed_password)` triggers an asynchronous hash recalculation and database update.

---

## Consequences

### Positive
* **State-of-the-Art Security**: Protects stored credentials against offline brute-force attacks and hardware-accelerated cracking.
* **Persistent Identity Management**: Supports dynamic user registration, credential rotation, and user state checks (e.g., active/disabled accounts).
* **Timing Attack Mitigation**: Mitigates user-enumeration attacks by utilizing dummy verification runs for non-existent users and constant-time string comparisons.
* **Separation of Concerns**: Decouples identity storage from token generation in [[concepts/token-based-authentication]] and signature management in [[concepts/jwt-signing-verification]].

### Negative / Trade-offs
* **Increased Authentication Latency**: Argon2id is intentionally CPU- and memory-intensive; `POST /api/v1/auth/login` latency will increase (typically 50–200ms depending on cost factor tuning).
* **Database Dependency**: The login endpoint becomes stateful and dependent on database availability, requiring connection pooling and read/write resilience.
* **Denial of Service (DoS) Surface**: High concurrency of login attempts can exhaust CPU resources due to hashing costs. Mitigation requires IP-based rate limiting on the `/login` route.

---

## Related Documentation
* [[summaries/architecture-overview]] - Overall service architecture and layer hierarchy
* [[concepts/credential-validation-flow]] - Current vs. planned credential validation lifecycle
* [[entities/auth-engine]] - Cryptographic and authentication logic implementation
* [[entities/settings]] - Configuration parameters including `DATABASE_URL`
* [[decisions/environment-configuration-management]] - Dynamic settings injection for database connection strings
* [[decisions/stateless-vs-stateful-sessions]] - Architectural separation between stateful user identity and stateless JWT issuance