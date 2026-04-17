# Prompt 08 — Auth: JWT Validation & Exchange

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types (`AuthResponse` in `types.rs`), content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement JWT validation and token exchange endpoints. These are essential for testing Kong's JWT plugin, the OpenID Connect plugin's token introspection, and DataKit flows that involve token exchange between services.

## What to build

### File: `src/auth_jwt.rs`

### Shared JWT Secret

Generate an HS256 secret at startup (or use a fixed well-known one for demo reproducibility). Store it as shared state. Suggested: use a fixed secret `rustybin-demo-secret-do-not-use-in-production` for HS256 so demo JWTs are reproducible and debuggable.

Also generate an RS256 key pair at startup using `ring` or `jsonwebtoken`'s RSA support, and store it in shared state. This will be used by the OIDC provider in the next prompt.

Create a shared `JwtState` struct:

```rust
pub struct JwtState {
    pub hs256_secret: String,
    pub rs256_encoding_key: EncodingKey,   // private key for signing
    pub rs256_decoding_key: DecodingKey,   // public key for verification
    pub rs256_jwk: serde_json::Value,      // JWK representation of public key
}
```

Store this in app state so it's accessible by both this module and the OIDC provider.

### Routes

All routes support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE.

| Route | Behaviour |
|---|---|
| `/auth/jwt` | Validate a Bearer JWT from the Authorization header. Decode and return header + claims. Does NOT verify signature — structure-only validation. |
| `/auth/jwt/exchange` | Accept a structurally valid Bearer JWT, then issue a NEW HS256-signed JWT with inherited claims plus `iss: rustybin`, `iat`, `jti`. Return the new signed token. |

### Behaviour — JWT Validation (`/auth/jwt`)

1. Require `Authorization: Bearer <token>` header.
2. Split the token on `.` — must have exactly 3 parts (header.payload.signature).
3. Base64-decode the header and payload parts.
4. Parse both as JSON.
5. Do NOT verify the signature — this endpoint validates structure only, which is useful for testing Kong's JWT plugin (which does the actual verification at the gateway layer).
6. **Success (200):** Return `AuthResponse` with `authenticated: true`, `auth_type: "jwt"`, `claims` (decoded payload), `jwt_header` (decoded header).
7. **Failure (401):** Return `{"authenticated": false, "error": "unauthorized"}` if:
   - No Authorization header
   - Doesn't start with `Bearer `
   - Token doesn't have 3 parts
   - Header or payload aren't valid base64/JSON

### Behaviour — JWT Exchange (`/auth/jwt/exchange`)

1. Same initial validation as `/auth/jwt` (structure check).
2. Take the decoded claims from the incoming token.
3. Create a new claims object that:
   - Inherits all claims from the incoming token
   - Overrides/adds `iss: "rustybin"`
   - Adds `iat` (current unix timestamp)
   - Adds `jti` (new UUID v4)
   - Adds `exp` (iat + 3600 = 1 hour)
4. Sign with HS256 using the shared secret.
5. **Success (200):**

```json
{
    "authenticated": true,
    "auth_type": "jwt",
    "claims": { /* claims in the new token */ },
    "exchanged_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

6. Use a dedicated response struct for this (e.g. `JwtExchangeResponse`) — it extends beyond `AuthResponse`.

### Router integration

Export `pub fn router(jwt_state: Arc<JwtState>) -> Router`. Merge into app router in `main.rs`.

Ensure the `JwtState` is created once in `main.rs` and shared via `Arc`.

## Verification

1. `cargo build` — compiles cleanly
2. Generate a test JWT: `echo -n '{"alg":"HS256","typ":"JWT"}' | base64 | tr -d '=' | tr '/+' '_-'` → header, `echo -n '{"sub":"1234","name":"Test User"}' | base64 | tr -d '=' | tr '/+' '_-'` → payload, concatenate with `.fakesig`
3. `curl -H 'Authorization: Bearer <test_jwt>' http://localhost/auth/jwt` → 200, shows decoded claims
4. `curl http://localhost/auth/jwt` → 401
5. `curl -H 'Authorization: Bearer <test_jwt>' http://localhost/auth/jwt/exchange` → 200, returns new signed token
6. Decode the exchanged token and verify it has `iss: rustybin`, `iat`, `jti`, `exp`
