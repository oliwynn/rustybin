# Prompt 02 — Echo & Anything Endpoints

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement the core echo endpoints that reflect back every detail of the incoming request. These are the most-used endpoints for gateway testing — they let you see exactly what headers, body, and query params arrive at the upstream after gateway plugins have transformed the request.

## What to build

### File: `src/echo.rs`

### Echo Response Schema

```rust
#[derive(Serialize, Deserialize)]
pub struct EchoResponse {
    pub method: String,
    pub path: String,
    pub path_info: Vec<String>,          // path segments split by /
    pub query_string: String,            // raw query string
    pub query_params: serde_json::Value, // parsed query params (supports repeated keys as arrays)
    pub headers: HashMap<String, Vec<String>>,  // all headers, multi-value aware
    pub host: String,
    pub port: u16,
    pub scheme: String,                  // "http" or "https"
    pub remote_ip: String,              // peer IP (or X-Forwarded-For if trust_forward is on)
    pub body: EchoBody,
    pub timestamp_unix_ms: u64,
}

#[derive(Serialize, Deserialize)]
pub struct EchoBody {
    pub present: bool,                   // was there a body at all?
    pub included: bool,                  // was it included in the response? (false if too large or binary)
    pub body: Option<String>,            // the body text if included
    pub bytes: usize,                    // raw byte count
    pub truncated: bool,                 // was it truncated?
    pub utf8: Option<bool>,             // was it valid UTF-8?
    pub reason: Option<String>,          // why body was excluded (e.g. "binary", "too_large")
}
```

### Routes

All of these routes must support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.

| Route | Behaviour |
|---|---|
| `/echo` | Return full EchoResponse |
| `/echo/*path` | Same, but captures sub-path in `path` and `path_info` |
| `/anything` | Alias for `/echo` — identical behaviour |
| `/anything/*path` | Alias for `/echo/*path` |

### Behaviour details

1. **Headers**: Capture ALL request headers. Represent them as `header_name → [value1, value2]` to handle multi-value headers correctly.

2. **Query params**: Parse the raw query string. If a key appears multiple times (e.g. `?a=1&a=2`), represent it as an array. Single values are strings.

3. **Body handling**:
   - If no body → `present: false`, everything else null/zero
   - If body is present and valid UTF-8 and under `RUSTYBIN_BODY_LIMIT` → include it
   - If body exceeds limit → truncate and set `truncated: true`
   - If body is not valid UTF-8 → set `utf8: false`, `reason: "binary"`, don't include
   - Never panic on malformed input

4. **Scheme detection**: Check `X-Forwarded-Proto` header first (if `trust_forward` is enabled), then fall back to whether the connection is TLS.

5. **Remote IP**: Use `X-Forwarded-For` first value if `trust_forward` is enabled, otherwise use the peer socket address.

6. **Content negotiation**: Use the shared content negotiation helper to return JSON or XML based on the `Accept` header.

7. **Config access**: Use axum's `State` or `Extension` to access the shared config (for `trust_forward`, `body_limit`).

### Router integration

Export a `pub fn router(config: Arc<Config>) -> Router` function.

In `main.rs`, merge this router into the app router.

## Verification

1. `cargo build` — compiles cleanly
2. `cargo clippy` — no warnings
3. Test with curl:
   - `curl http://localhost/echo` → returns method, path, headers
   - `curl -X POST http://localhost/echo/foo/bar -d '{"test":true}' -H 'Content-Type: application/json'` → body included, path_info = ["echo","foo","bar"]
   - `curl http://localhost/anything?a=1&a=2&b=3` → query_params has `a` as array
   - `curl -H 'Accept: application/xml' http://localhost/echo` → returns XML
