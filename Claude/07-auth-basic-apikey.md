# Prompt 07 — Auth: Basic Auth & API Key

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement Basic Auth and API Key authentication endpoints. These let you test Kong's basic-auth, key-auth, and credential-based plugins by providing a backend that actually enforces auth and reflects what it received.

## What to build

### File: `src/auth_basic.rs`

### Routes

All routes support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE.

| Route | Behaviour |
|---|---|
| `/auth/basic-auth` | Enforce Basic Auth with default credentials: username `basic`, password `password`. |
| `/auth/basic-auth/{username}/{password}` | Enforce Basic Auth with custom credentials from the path. |

### Auth Response Schema

```rust
#[derive(Serialize, Deserialize)]
pub struct AuthResponse {
    pub authenticated: bool,
    pub auth_type: String,                    // "basic-auth", "api-key", "jwt", "mtls"
    #[serde(skip_serializing_if = "Option::is_none")]
    pub username: Option<String>,             // for basic-auth
    #[serde(skip_serializing_if = "Option::is_none")]
    pub header: Option<String>,               // for api-key: which header was used
    #[serde(skip_serializing_if = "Option::is_none")]
    pub claims: Option<serde_json::Value>,    // for jwt
    #[serde(skip_serializing_if = "Option::is_none")]
    pub jwt_header: Option<serde_json::Value>,// for jwt
    #[serde(skip_serializing_if = "Option::is_none")]
    pub client_dn: Option<String>,            // for mtls
    #[serde(skip_serializing_if = "Option::is_none")]
    pub client_ca: Option<String>,            // for mtls
}
```

Put this in `src/types.rs` so all auth modules share it.

### Behaviour — Basic Auth

1. Check for `Authorization` header starting with `Basic `.
2. Base64-decode the value after `Basic `.
3. Split on `:` to get username and password.
4. Compare against expected credentials (default or path-specified).
5. **Success (200):** Return `AuthResponse` with `authenticated: true`, `auth_type: "basic-auth"`, and the `username`.
6. **Failure (401):** Return `{"authenticated": false, "error": "unauthorized"}` with `WWW-Authenticate: Basic realm="rustybin"` header.
7. Support content negotiation.

### File: `src/auth_apikey.rs`

### Routes

All routes support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE.

| Route | Behaviour |
|---|---|
| `/auth/api-key` | Enforce API key with default header `apikey` and value `my-key`. |
| `/auth/api-key/{header_name}/{key_value}` | Enforce API key with custom header name and value. |

### Behaviour — API Key

1. Read the header specified by `header_name` (default: `apikey`).
2. Compare the value against `key_value` (default: `my-key`).
3. Header name matching should be **case-insensitive**.
4. **Success (200):** Return `AuthResponse` with `authenticated: true`, `auth_type: "api-key"`, and `header` showing which header name was used.
5. **Failure (401):** Return `{"authenticated": false, "error": "unauthorized"}`.
6. Support content negotiation.

### Router integration

Export `pub fn basic_auth_router() -> Router` and `pub fn apikey_router() -> Router`. Merge both into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -u basic:password http://localhost/auth/basic-auth` → 200, authenticated
3. `curl http://localhost/auth/basic-auth` → 401 with WWW-Authenticate header
4. `curl -u alice:secret http://localhost/auth/basic-auth/alice/secret` → 200
5. `curl -u alice:wrong http://localhost/auth/basic-auth/alice/secret` → 401
6. `curl -H 'apikey: my-key' http://localhost/auth/api-key` → 200
7. `curl http://localhost/auth/api-key` → 401
8. `curl -H 'X-Api-Key: supersecret' http://localhost/auth/api-key/X-Api-Key/supersecret` → 200
