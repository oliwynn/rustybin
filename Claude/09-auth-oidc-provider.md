# Prompt 09 — Auth: OIDC Identity Provider

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, the `JwtState` created in the JWT module (prompt 08), shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement a self-contained OpenID Connect Identity Provider. This eliminates the need for Auth0, Keycloak, or any external IdP during demos. It provides enough OIDC compliance to satisfy Kong's OpenID Connect plugin, including discovery, token issuance, JWKS, and userinfo.

## What to build

### File: `src/oidc.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/.well-known/openid-configuration` | GET | OIDC Discovery document |
| `/oauth/token` | POST | Token endpoint — issue signed JWTs via `client_credentials` or `password` grant |
| `/oauth/jwks` | GET | JSON Web Key Set — public keys for token verification |
| `/oauth/authorize` | GET | Authorization endpoint — minimal HTML consent page that redirects with an auth code |
| `/oauth/authorize` | POST | Process the consent form submission |
| `/oauth/userinfo` | GET | Return user profile claims from a valid Bearer token |
| `/oauth/introspect` | POST | Token introspection endpoint (RFC 7662) |

### OIDC Discovery (`/.well-known/openid-configuration`)

Return a JSON document that satisfies the OpenID Connect Discovery spec. The `issuer` must be derived from the incoming request's `Host` header (so it works behind Kong).

```json
{
    "issuer": "http://{host}",
    "authorization_endpoint": "http://{host}/oauth/authorize",
    "token_endpoint": "http://{host}/oauth/token",
    "userinfo_endpoint": "http://{host}/oauth/userinfo",
    "jwks_uri": "http://{host}/oauth/jwks",
    "introspection_endpoint": "http://{host}/oauth/introspect",
    "response_types_supported": ["code", "token", "id_token", "code id_token"],
    "grant_types_supported": ["authorization_code", "client_credentials", "password"],
    "subject_types_supported": ["public"],
    "id_token_signing_alg_values_supported": ["RS256", "HS256"],
    "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"],
    "scopes_supported": ["openid", "profile", "email"],
    "claims_supported": ["sub", "iss", "aud", "exp", "iat", "name", "email"]
}
```

### Token Endpoint (`/oauth/token`)

Support three grant types via `application/x-www-form-urlencoded` body:

**1. `client_credentials`**
- Accept `client_id` and `client_secret` via Basic Auth OR form body
- Default accepted credentials: `client_id=rustybin`, `client_secret=secret`
- Also accept ANY client_id/secret combo (for flexibility in demos — log a warning for non-default creds)
- Issue a signed JWT (RS256 using JwtState's key) with claims:
  - `iss`: derived from Host
  - `sub`: the client_id
  - `aud`: the client_id
  - `exp`: now + 3600
  - `iat`: now
  - `jti`: UUID v4
  - `scope`: from request or default `openid`
  - `token_type`: `bearer`

**2. `password`**
- Accept `username` and `password` in form body, plus client credentials
- Default accepted: any username/password (demo mode)
- Issue JWT with `sub` = username, plus `name` and `email` claims derived from username

**3. `authorization_code`**
- Accept `code`, `redirect_uri`, `client_id`, `client_secret`
- Validate the code exists in an in-memory store (populated by `/oauth/authorize`)
- Exchange for JWT

**Response format:**
```json
{
    "access_token": "eyJ...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "id_token": "eyJ...",
    "scope": "openid profile email"
}
```

### JWKS (`/oauth/jwks`)

Return the RS256 public key from `JwtState` in JWK format:

```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "alg": "RS256",
            "kid": "rustybin-1",
            "n": "...",
            "e": "AQAB"
        }
    ]
}
```

### Authorization Endpoint (`/oauth/authorize`)

**GET**: Return a minimal HTML page with a login form. Accept query params: `client_id`, `redirect_uri`, `response_type`, `scope`, `state`.

The HTML should show:
- "Rustybin OIDC Login"
- Username and password fields (pre-filled with `demo` / `demo`)
- An "Authorize" button
- Hidden fields for all the query params

**POST**: Process the form. Generate a random authorization code, store it in-memory (with associated claims, redirect_uri, and 60s expiry), and redirect to `redirect_uri?code={code}&state={state}`.

### Userinfo (`/oauth/userinfo`)

1. Require `Authorization: Bearer <token>`.
2. Decode the JWT (use RS256 verification with JwtState's key, or accept HS256 with the shared secret).
3. Return user claims:

```json
{
    "sub": "...",
    "name": "...",
    "email": "...@rustybin.local",
    "email_verified": true
}
```

### Introspection (`/oauth/introspect`)

1. Accept `token` in form body, plus client credentials (Basic Auth or form).
2. Attempt to decode/verify the token.
3. **Active token:** Return `{"active": true, "sub": "...", "scope": "...", "exp": ..., "client_id": "..."}`.
4. **Invalid/expired:** Return `{"active": false}`.

### In-memory auth code store

Use a `DashMap<String, AuthCode>` or `Mutex<HashMap<String, AuthCode>>` to store authorization codes. Each code should expire after 60 seconds. Clean up expired codes on each access.

### Router integration

Export `pub fn router(jwt_state: Arc<JwtState>) -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/.well-known/openid-configuration` → valid discovery document
3. `curl -X POST http://localhost/oauth/token -d 'grant_type=client_credentials&client_id=rustybin&client_secret=secret'` → returns access_token
4. `curl http://localhost/oauth/jwks` → returns JWK set with RS256 key
5. Decode the access_token and verify the signature against the JWKS
6. `curl -H 'Authorization: Bearer <token>' http://localhost/oauth/userinfo` → returns user claims
7. `curl -X POST http://localhost/oauth/introspect -d 'token=<token>&client_id=rustybin&client_secret=secret'` → `active: true`
8. Test the full auth code flow: GET `/oauth/authorize?...` → submit form → get code → exchange at `/oauth/token`
