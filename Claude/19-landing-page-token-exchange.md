# Prompt 19 — Landing Page & RFC 8693 Token Exchange

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, the shared `Config` via `Arc<Config>`, the `JwtState` for signing tokens, the OIDC module (`src/oidc.rs`), content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Two additions:

1. **Landing page** — a styled HTML dashboard at `GET /` that serves as the front door to Rustybin, listing all endpoint categories with links, quick-start examples, and live instance metadata.
2. **RFC 8693 OAuth 2.0 Token Exchange** — a new grant type on the existing `/oauth/token` endpoint, implementing the `urn:ietf:params:oauth:grant-type:token-exchange` flow for testing Kong's token exchange plugin scenarios.

---

## Part A — Landing Page

### File: `src/landing.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/` | GET | Serve a styled HTML landing page |

### Landing page requirements

The HTML page must include:

**Header section:**
- Title: "Rustybin" with tagline "High-performance HTTP stub service for API gateway testing"
- Version badge (from `CARGO_PKG_VERSION`)
- Links to `/docs` (API Explorer) and `/openapi.json`

**Endpoint directory:**
A card-based or table layout grouping all endpoints by category. Each entry should show the route, supported methods, and a one-line description. Categories:

| Category | Endpoints |
|---|---|
| Echo & Reflection | `/echo`, `/anything` |
| Status Codes | `/status/:code` |
| Response Shaping | `/delay/:ms`, `/bytes/:n`, `/stream/:n`, `/drip`, `/cache/:ttl`, `/response-headers` |
| Redirects & Cookies | `/redirect/:n`, `/absolute-redirect/:n`, `/redirect-to`, `/cookies`, `/cookies/set`, `/cookies/delete` |
| Info & Random | `/ip`, `/date`, `/time`, `/uuid`, `/guuid`, `/random/*`, `/image/*` |
| Auth: Basic & API Key | `/auth/basic-auth`, `/auth/api-key` |
| Auth: JWT | `/auth/jwt`, `/auth/jwt/exchange` |
| Auth: OIDC Provider | `/.well-known/openid-configuration`, `/oauth/token`, `/oauth/jwks`, `/oauth/authorize`, `/oauth/userinfo`, `/oauth/introspect` |
| Auth: mTLS | `/auth/mtls`, `/auth/mtls/get-client-cert`, `/auth/mtls/get-ca-cert` |
| AI Gateway | `/ai/v1/chat/completions`, `/ai/v1/completions`, `/ai/v1/embeddings`, `/ai/v1/models` |
| GraphQL | `/graphql`, `/graphql/schema` |
| Orchestration | `/orchestration/step/1-4`, `/orchestration/status` |
| SOAP / XML | `/soap`, `/soap/wsdl` |
| Reliability | `/flaky/:rate`, `/flaky/pattern/:p`, `/flaky/after/:n`, `/flaky/recover/:n` |
| Utility | `/health`, `/identity`, `/openapi.json`, `/openapi.yaml`, `/docs` |

**Quick-start section:**
3–4 curl examples showing common operations:
```
curl http://localhost/echo
curl -X POST http://localhost/oauth/token -d 'grant_type=client_credentials&client_id=rustybin&client_secret=secret'
curl http://localhost/status/418
curl http://localhost/ai/v1/chat/completions -d '{"model":"rustybin","messages":[{"role":"user","content":"hello"}]}'
```

**Footer:**
- Instance ID, hostname, and version
- Link to the project repository (or just "Rustybin" branding)

### Styling requirements

- Self-contained — all CSS inline in the HTML (no external CDN dependencies)
- Dark-mode friendly colour scheme (dark background, light text, accent colour for links)
- Responsive layout that works on mobile
- Monospace font for endpoint paths and code examples
- Clean, professional look — no emojis, no icons

### Implementation notes

- Use `axum::response::Html` with a `const &str` or `format!()` to inject dynamic values (version, instance_id)
- The handler needs access to `Arc<Config>` via axum `State` to read `instance_id`
- Register in `main.rs` via `.merge(landing::router())`

---

## Part B — RFC 8693 Token Exchange

### Overview

Implement [RFC 8693 OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693) as a new grant type on the existing `/oauth/token` endpoint. This allows a client to present an existing token (the "subject token") and receive a new token in return — used for impersonation, delegation, and cross-service token exchange scenarios.

### Modifications to: `src/oidc.rs`

### Grant type

Add support for `grant_type=urn:ietf:params:oauth:grant-type:token-exchange` on the existing `POST /oauth/token` endpoint.

### Request parameters (form-encoded)

| Parameter | Required | Description |
|---|---|---|
| `grant_type` | Yes | Must be `urn:ietf:params:oauth:grant-type:token-exchange` |
| `subject_token` | Yes | The existing token to exchange (a JWT string) |
| `subject_token_type` | Yes | Token type URI (see below) |
| `actor_token` | No | Optional secondary token representing the acting party |
| `actor_token_type` | Conditional | Required if `actor_token` is present |
| `audience` | No | Intended audience for the new token |
| `scope` | No | Requested scopes for the new token |
| `resource` | No | Target service URI |
| `requested_token_type` | No | Desired token type URI for the new token |

### Supported token type URIs

| URI | Meaning |
|---|---|
| `urn:ietf:params:oauth:token-type:access_token` | OAuth 2.0 access token |
| `urn:ietf:params:oauth:token-type:id_token` | OIDC ID token |
| `urn:ietf:params:oauth:token-type:jwt` | Generic JWT |
| `urn:ietf:params:oauth:token-type:refresh_token` | Refresh token |

### Behaviour

1. **Validate required fields**: Return `400` with `{"error": "invalid_request", "error_description": "..."}` if `subject_token` or `subject_token_type` is missing.

2. **Validate `subject_token_type`**: Must be one of the four supported URIs above. Return `400` with `invalid_request` if unrecognised.

3. **Validate `actor_token_type`** if `actor_token` is present: same validation. Return `400` if `actor_token` is present but `actor_token_type` is missing.

4. **Decode the subject token**: Attempt to verify as a JWT using `JwtState` (try RS256 then HS256, matching the existing `verify_token()` helper). If verification fails, fall back to structural decode only (split on `.`, base64url-decode header + payload) — this allows exchanging third-party tokens.

5. **Decode the actor token** (if present): same logic.

6. **Build the new token claims**:
   - `iss`: derived from Host header (same as other OIDC endpoints)
   - `sub`: from subject token's `sub` claim, or `"unknown"` if missing
   - `aud`: from the `audience` request parameter, or subject token's `aud`, or the client_id
   - `exp`: `now + 3600`
   - `iat`: now
   - `jti`: UUID v4
   - `scope`: from request `scope` parameter, or subject token's `scope`, or `"openid"`
   - `act`: if actor_token was provided, include `{"sub": "<actor_sub>"}` (RFC 8693 §4.1 delegation semantics)
   - `original_claims`: include the full decoded subject token claims for traceability (Rustybin-specific, useful for debugging)

7. **Sign the new token**: RS256 using `JwtState`, matching existing OIDC token issuance.

8. **Return the response**:
```json
{
    "access_token": "eyJ...",
    "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "openid"
}
```

If `requested_token_type` was `urn:ietf:params:oauth:token-type:id_token`, set `issued_token_type` accordingly and return the same JWT as both `access_token` and `id_token` fields.

### Discovery update

Update `/.well-known/openid-configuration` to include:
- Add `"urn:ietf:params:oauth:grant-type:token-exchange"` to `grant_types_supported`

### Error handling

All errors use the existing `error_response()` helper with OAuth-standard error codes:
- `invalid_request` — missing required parameters
- `invalid_target` — if `resource` URI is malformed (basic URL validation only)
- `unsupported_grant_type` — only if the grant_type string is completely unrecognised (should not happen for token-exchange since we handle it)

### TokenRequest struct update

Add these optional fields to the existing `TokenRequest` deserialize struct:
```rust
#[serde(default)]
subject_token: Option<String>,
#[serde(default)]
subject_token_type: Option<String>,
#[serde(default)]
actor_token: Option<String>,
#[serde(default)]
actor_token_type: Option<String>,
#[serde(default)]
audience: Option<String>,
#[serde(default)]
resource: Option<String>,
#[serde(default)]
requested_token_type: Option<String>,
```

### OpenAPI spec update

Update `src/openapi.rs` to document the token exchange grant type on the `/oauth/token` endpoint. Add:
- Description mentioning RFC 8693 support
- Request body schema showing the token-exchange parameters
- Response schema for the token exchange response format

---

## Verification

1. `cargo build` — compiles cleanly
2. `cargo clippy -- -D warnings` — no warnings
3. `cargo test` — all existing + new tests pass
4. `curl http://localhost/` — returns styled HTML landing page with all endpoint categories
5. Landing page shows correct version and instance_id
6. Token exchange flow:
```bash
# Get an initial token
TOKEN=$(curl -s -X POST http://localhost/oauth/token \
  -d 'grant_type=client_credentials&client_id=rustybin&client_secret=secret' | jq -r .access_token)

# Exchange it
curl -X POST http://localhost/oauth/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange&subject_token=$TOKEN&subject_token_type=urn:ietf:params:oauth:token-type:access_token&audience=my-service"
```
7. Verify discovery doc includes `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported`
8. Verify error responses for missing `subject_token`, invalid `subject_token_type`
9. Test with actor_token delegation:
```bash
ACTOR=$(curl -s -X POST http://localhost/oauth/token \
  -d 'grant_type=password&username=admin&password=admin&client_id=rustybin&client_secret=secret' | jq -r .access_token)

curl -X POST http://localhost/oauth/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange&subject_token=$TOKEN&subject_token_type=urn:ietf:params:oauth:token-type:access_token&actor_token=$ACTOR&actor_token_type=urn:ietf:params:oauth:token-type:access_token"
```
10. Decoded exchanged token should include `act.sub` claim when actor_token was provided

## Tests

### Landing page tests (`src/landing.rs`)
- `GET /` returns 200 with `text/html` content-type
- Response body contains "Rustybin"
- Response body contains the version string
- Response body contains `/echo`, `/docs`, `/oauth/token` (spot-check key endpoints)

### Token exchange tests (in `src/oidc.rs`)
- Token exchange with valid subject_token returns 200 with `issued_token_type`
- Missing `subject_token` returns 400 with `invalid_request`
- Missing `subject_token_type` returns 400 with `invalid_request`
- Invalid `subject_token_type` URI returns 400 with `invalid_request`
- Actor token with missing `actor_token_type` returns 400 with `invalid_request`
- Actor token delegation includes `act` claim in response token
- `requested_token_type` of `id_token` returns `issued_token_type` of `urn:ietf:params:oauth:token-type:id_token`
- Exchange with third-party (unverifiable) JWT still succeeds via structural decode
- Discovery document includes `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported`
