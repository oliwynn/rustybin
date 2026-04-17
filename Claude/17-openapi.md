# Prompt 17 — OpenAPI Specification

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand ALL endpoints that have been implemented across every module. You need to document every single route.

## Goal

Create a comprehensive OpenAPI 3.0.3 specification covering every endpoint in Rustybin, and serve it from the application. This spec will be used by Kong's Dev Portal, service discovery, and OAS validation plugins.

## What to build

### File: `src/openapi.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/openapi.json` | GET | Return the full OpenAPI 3.0.3 spec as JSON |
| `/openapi.yaml` | GET | Return the same spec as YAML |
| `/docs` | GET | Serve a Swagger UI or Scalar HTML page that loads the spec from `/openapi.json` |

### OpenAPI Spec

Build the spec as a Rust struct (or a large JSON/YAML const) that covers:

#### Info
```yaml
openapi: "3.0.3"
info:
  title: Rustybin
  description: >
    A high-performance, all-in-one HTTP stub service for API gateway testing.
    Built in Rust with axum. Designed to exercise every category of Kong Gateway plugin.
  version: "{from Cargo.toml}"
  contact:
    name: Rustybin
servers:
  - url: "http://localhost"
    description: HTTP
  - url: "https://localhost"
    description: HTTPS / mTLS
```

#### Tags

Organise endpoints into these tags:
- **Echo** — `/echo`, `/anything`
- **Auth** — All auth endpoints (basic, api-key, jwt, mtls, OIDC)
- **Status** — `/status/{code}`
- **Response Shaping** — `/delay`, `/bytes`, `/stream`, `/drip`, `/cache`, `/response-headers`
- **Redirects & Cookies** — `/redirect`, `/cookies`
- **Info** — `/ip`, `/date`, `/time`
- **Random** — `/uuid`, `/guuid`, `/random`
- **AI Gateway** — `/ai/v1/*`
- **GraphQL** — `/graphql`
- **Orchestration** — `/orchestration/*`
- **SOAP** — `/soap`
- **Utility** — `/health`, `/identity`, `/image`, `/flaky`

#### Paths

Document EVERY endpoint with:
- Summary and description
- All supported HTTP methods
- Path parameters with types and validation
- Query parameters with defaults
- Request body schemas where applicable
- Response schemas for success and error cases
- Security requirements where applicable
- Example values

#### Components

Define reusable schemas in `components/schemas`:
- `EchoResponse`, `EchoBody`
- `AuthResponse`, `JwtExchangeResponse`
- `ErrorResponse`
- `ChatCompletionRequest`, `ChatCompletionResponse`
- etc.

Define security schemes:
- `basicAuth` (HTTP Basic)
- `bearerAuth` (JWT Bearer)
- `apiKeyAuth` (API Key header)

### Implementation approach

**Option A (recommended):** Use the `utoipa` crate with `utoipa-axum` to generate the OpenAPI spec from the existing code using derive macros. This keeps the spec in sync with the code.

**Option B:** Build the spec as a `serde_json::Value` manually. Less elegant but avoids adding derive macros to every existing module.

Choose whichever approach results in a complete and accurate spec. If using `utoipa`, you'll need to add `#[utoipa::path(...)]` annotations to existing handlers — that's acceptable.

### Docs UI

For `/docs`, serve a minimal HTML page that loads Scalar (modern OpenAPI docs UI) from CDN:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Rustybin API Docs</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
</head>
<body>
    <script id="api-reference" data-url="/openapi.json"></script>
    <script src="https://cdn.jsdelivr.net/npm/@scalar/api-reference"></script>
</body>
</html>
```

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/openapi.json` → valid OpenAPI 3.0.3 JSON
3. `curl http://localhost/openapi.yaml` → valid YAML version
4. Verify the spec covers ALL endpoints (count them — should be 40+ paths)
5. Paste the JSON into https://editor.swagger.io — should parse without errors
6. `curl http://localhost/docs` → returns HTML page
7. Spot-check: verify `/ai/v1/chat/completions` has request/response schemas, `/auth/basic-auth` has security requirement, `/delay/{ms}` has parameter validation
