# Prompt 03 — Status Codes

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement an endpoint that returns any HTTP status code on demand. This is essential for testing gateway error handling, exit-transformer, and request-termination plugins.

## What to build

### File: `src/status.rs`

### Routes

All routes support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE.

| Route | Behaviour |
|---|---|
| `/status/{code}` | Return the given HTTP status code (100–599) |

### Behaviour

1. Parse `{code}` as a `u16`. If it's not a valid number or outside 100–599, return `400` with `{"error": "invalid_status_code", "details": "Status code must be between 100 and 599"}`.

2. **Informational (1xx)** and **No Content (204, 304)**: Return an empty body with the requested status.

3. **All other codes**: Return `{"status": <code>}` as the body with the requested status code.

4. Support content negotiation — XML response if `Accept: application/xml`.

5. For redirect codes (301, 302, 307, 308): also include a `Location` header pointing to `/echo` so you can see the redirect chain in action when testing through a gateway.

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -i http://localhost/status/200` → 200 with `{"status": 200}`
3. `curl -i http://localhost/status/418` → 418 with `{"status": 418}`
4. `curl -i http://localhost/status/204` → 204 with empty body
5. `curl -i http://localhost/status/302` → 302 with `Location: /echo`
6. `curl -i http://localhost/status/999` → 400 error
