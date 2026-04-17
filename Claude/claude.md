# Rustybin — Build Orchestrator

## What is Rustybin?

Rustybin is a high-performance, all-in-one HTTP stub/echo service written in Rust using `axum`. It is designed to exercise every category of API gateway plugin (specifically Kong Gateway) from a single binary. Think httpbin, but faster, broader, and purpose-built for gateway demos and testing.

## How to use this file

This file is the orchestrator. Each prompt file in `./prompts/` builds one self-contained module of Rustybin. **Run them in order.** After each prompt completes, **clear your context** before starting the next one — each prompt is written to be self-contained and will read the existing codebase to orient itself.

## Build Order

Run the following prompts in sequence. Each one will produce working, compilable code before finishing.

### Phase 1 — Foundation
1. **`prompts/01-scaffold.md`** — Project structure, Cargo.toml, main.rs, Docker, TLS certs, health check
2. **`prompts/02-echo.md`** — `/echo`, `/anything`, content negotiation (JSON + XML), body capture, sub-paths

### Phase 2 — Utility & Shaping
3. **`prompts/03-status-codes.md`** — `/status/{code}` for any HTTP status 100–599
4. **`prompts/04-response-shaping.md`** — `/bytes/{n}`, `/delay/{ms}`, `/drip`, `/stream/{n}`, `/response-headers`, `/cache/{ttl}`
5. **`prompts/05-redirect-cookies.md`** — `/redirect/{n}`, `/redirect-to`, `/cookies/set`, `/cookies`, `/cookies/delete`
6. **`prompts/06-info-random.md`** — `/ip`, `/date`, `/time`, `/uuid`, `/guuid`, `/random/*`, `/lorem-ipsum`, `/image/{type}`

### Phase 3 — Auth
7. **`prompts/07-auth-basic-apikey.md`** — `/auth/basic-auth`, `/auth/api-key` with default and custom credential paths
8. **`prompts/08-auth-jwt.md`** — `/auth/jwt`, `/auth/jwt/exchange` with HS256 signing
9. **`prompts/09-auth-oidc-provider.md`** — Full OIDC IdP: discovery, `/oauth/token`, `/oauth/jwks`, `/oauth/authorize`, `/oauth/userinfo`
10. **`prompts/10-auth-mtls.md`** — `/auth/mtls`, `/auth/mtls/get-client-cert`, demo CA generation at startup

### Phase 4 — Advanced Use Cases
11. **`prompts/11-ai-gateway.md`** — OpenAI-compatible `/ai/v1/chat/completions` (streaming SSE), `/ai/v1/embeddings`, `/ai/v1/completions`
12. **`prompts/12-graphql.md`** — `/graphql` with users/products/orders schema, hardcoded resolvers
13. **`prompts/13-datakit-orchestration.md`** — `/orchestration/step/{n}` multi-step chained endpoints for DataKit demos
14. **`prompts/14-soap-xml.md`** — `/soap` endpoint accepting SOAP envelopes, returning SOAP responses
15. **`prompts/15-upstream-identity.md`** — `/identity` returning hostname, instance ID, port for load balancer demos
16. **`prompts/16-flaky.md`** — `/flaky/{fail_rate}` configurable failure rate for circuit breaker and health check testing

### Phase 5 — Polish
17. **`prompts/17-openapi.md`** — Auto-generate and serve `/openapi.json` covering all endpoints
18. **`prompts/18-final-review.md`** — Full compile check, `cargo clippy`, Docker build, integration smoke test

## Conventions

Every prompt must follow these rules:

- **Compile after every prompt.** Run `cargo build` before declaring done. Fix all errors.
- **No `unwrap()` in production paths.** Use proper error handling with `thiserror` or `anyhow`.
- **All routes registered in `main.rs`** via a `router()` function that merges sub-routers from each module.
- **Each module lives in `src/{module_name}.rs`** (or `src/{module_name}/mod.rs` if complex).
- **Shared types** go in `src/types.rs`.
- **Consistent JSON responses** — all error responses use `{"error": "...", "details": "..."}`.
- **Content negotiation** — every endpoint that returns JSON should also support `Accept: application/xml` using the shared serialisation helper.
- **Logging** — use `tracing` with structured fields. Log method, path, and key headers on every request via tower middleware.
- **Tests** — each module should include at least basic unit/integration tests using `axum::test`.
