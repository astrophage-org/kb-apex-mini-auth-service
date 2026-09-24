# Architectural Decision Record: Symmetric vs. Asymmetric Token Signing

## Status
**Accepted** (Current implementation: `HS256`; Architectural pathway defined for `RS256`/`EdDSA` migration).

---

## Context

In [[concepts/token-based-authentication]], JSON Web Tokens (JWTs) must be cryptographically signed by the identity provider to guarantee payload integrity and authenticity. When designing the security layer in [[entities/auth-engine]] (`src/auth.py`), a choice had to be made regarding the cryptographic signing algorithm:

1. **Symmetric Signing (e.g., HMAC-SHA256 / `HS256`)**: A single shared secret key is used for both signing (token issuance) and verification (token decoding).
2. **Asymmetric Signing (e.g., RSA-SHA256 / `RS256`, EdDSA / `Ed25519`)**: A private key is used exclusively by the auth service to sign tokens, while a mathematically linked public key is distributed to downstream resource servers and clients to verify signatures.

Currently, [[entities/settings]] in `src/config.py` defines:
```python
class Settings:
    JWT_SECRET: str = "astrophage-secret-key-123"
    ALGORITHM: str = "HS256"
```

While `HS256` satisfies the immediate needs of a self-contained auth service verifying its own tokens via [[entities/api-router]] (`GET /api/v1/auth/me`), scaling the architecture to external downstream consumers introduces trust and secret distribution challenges.

---

## Decision

We chose **Symmetric Signing (`HS256`)** for the initial implementation of `mini-auth-service`, while establishing an architectural roadmap to transition to **Asymmetric Signing (`RS256` or `EdDSA`)** as downstream microservices are introduced.

### Rationale

1. **Minimal Operational Complexity**: `HS256` requires only a single secret string (`JWT_SECRET`), simplifying early-stage configuration management and eliminating the need for PKCS#8 key pair generation, certificate management, or JWKS (JSON Web Key Set) hosting.
2. **Computational Performance**: Symmetric HMAC hashing exhibits negligible CPU overhead compared to RSA key operations, maximizing throughput for token generation in `create_access_token()` and token decoding in `verify_token()`.
3. **Unified Issuer & Verifier Scope**: In the current application topology described in [[summaries/architecture-overview]], `mini-auth-service` acts as both the token issuer (`POST /api/v1/auth/login`) and the token verifier (`GET /api/v1/auth/me`). Because the secret does not leave the boundary of this service, symmetric key exposure risks are contained.

---

## Trade-off Analysis

| Metric / Dimension | Symmetric (`HS256`) | Asymmetric (`RS256`) | Asymmetric (`EdDSA` / `Ed25519`) |
| :--- | :--- | :--- | :--- |
| **Current Adoption** | **Active** in `src/config.py` | Future Phase | Future Phase Candidate |
| **Key Material** | Single shared secret (`JWT_SECRET`) | RSA Private/Public Keypair (2048+ bit) | Curve25519 Private/Public Keypair |
| **Verification Trust Model** | **Shared Trust**: Verifiers possess signing capability | **Zero Trust Verifiers**: Public key can only verify | **Zero Trust Verifiers**: Public key can only verify |
| **Performance (Signing)** | Extremely Fast (HMAC) | Moderate (computationally intensive) | Very Fast (elliptic curve) |
| **Performance (Verification)** | Extremely Fast (HMAC) | Fast | Very Fast |
| **Token / Header Size** | Compact | Larger signature (256 bytes) | Compact signature (64 bytes) |
| **Operational Overhead** | Low (single env variable) | High (key rotation, JWKS endpoint) | Moderate (JWKS endpoint, smaller keys) |

---

## Consequences

### Positive
* **Simplicity**: No complex key-pair generation, file mounting, or PEM decoding inside [[entities/auth-engine]].
* **Portability**: The secret can easily be injected via environment variables as outlined in [[decisions/environment-configuration-management]].
* **Low Latency**: High request throughput for cryptographic operations in [[concepts/jwt-signing-verification]].

### Negative & Risks
* **Secret Distribution Risk in Microservices**: If external microservices need to independently validate [[entities/jwt-token-payload]] without routing back through `mini-auth-service`, sharing `JWT_SECRET` grants them the capability to forge valid tokens.
* **Key Revocation / Rotation Blast Radius**: Compromise of `JWT_SECRET` invalidates all currently issued tokens upon rotation and compromises the entire auth domain.

---

## Future Migration Pathway: Transitioning to Asymmetric Keys

When downstream services require independent token verification, the system will migrate to asymmetric signing:

```
+---------------------+                      +------------------------+
|  mini-auth-service  |                      | Downstream Consumer    |
|  (Private Key)      |                      | (Public Key / JWKS)    |
+----------+----------+                      +-----------+------------+
           |                                             |
           | 1. Sign JWT (RS256/EdDSA)                   |
           |-------------------------------------------->|
           |                                             | 2. Verify signature
           | 3. Expose /.well-known/jwks.json            |    using Public Key
           |<============================================|    (Cannot forge tokens)
```

1. **Configuration Update**: Update [[entities/settings]] to store `JWT_PRIVATE_KEY_PATH` and `JWT_PUBLIC_KEY_PATH` (or JWK structures).
2. **Algorithm Switch**: Change `settings.ALGORITHM` from `HS256` to `RS256` or `EdDSA`.
3. **Engine Updates**:
   * Update `create_access_token()` in `src/auth.py` to sign using the private key.
   * Update `verify_token()` in `src/auth.py` to verify using the public key.
4. **JWKS Exposure**: Expose a public endpoint (e.g., `/.well-known/jwks.json`) on [[entities/api-router]] to enable automated public key discovery and rotation for resource servers.