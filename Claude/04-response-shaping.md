# Prompt 04 — Response Shaping

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement endpoints that let you control the shape, size, timing, and cacheability of responses. These are critical for testing proxy-cache, response-ratelimiting, request-size-limiting, timeout, retry, and circuit-breaker plugins.

## What to build

### File: `src/response_shaping.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/delay/{ms}` | ALL | Wait `ms` milliseconds, then return echo response. Cap at 60000ms. Optional `?jitter=true` adds ±20% random variance. |
| `/bytes/{n}` | GET | Return exactly `n` random bytes as `application/octet-stream`. Cap at 10MB (10485760). |
| `/stream/{n}` | GET | Return `n` chunks of JSON objects as chunked transfer encoding. Each chunk is `{"id": i, "timestamp": ..., "data": "random_string"}` followed by newline. Cap at 1000 chunks. Optional `?delay=100` adds delay in ms between chunks. |
| `/drip` | GET | Slow-drip response. Query params: `bytes` (total, default 1024), `delay` (ms between drips, default 100), `chunk_size` (bytes per drip, default 10). Returns `application/octet-stream` streamed slowly. |
| `/response-headers` | GET | Returns a 200 response where every query parameter becomes a response header. E.g. `?X-Custom=hello&X-Another=world` sets those headers on the response. Body is the JSON of headers set. |
| `/cache/{ttl}` | GET | Returns a JSON response with proper cache headers: `Cache-Control: public, max-age={ttl}`, `ETag` (based on content hash), `Last-Modified` (server startup time). Supports conditional requests: `If-None-Match` → 304, `If-Modified-Since` → 304. Body is `{"cached": true, "ttl": ttl, "generated_at": "...", "etag": "..."}`. |

### Behaviour details

#### `/delay/{ms}`
- Use `tokio::time::sleep` for the delay
- Validate `ms` is 0–60000, return 400 if out of range
- If `?jitter=true`, multiply delay by random factor between 0.8 and 1.2
- After delay, return a standard echo-style response with an added `delay_ms` field showing actual delay applied
- This is the primary endpoint for testing gateway timeout and retry plugins

#### `/bytes/{n}`
- Generate `n` random bytes using `rand`
- Return as `application/octet-stream` with `Content-Length` header
- Validate `n` is 1–10485760, return 400 if out of range

#### `/stream/{n}`
- Use axum's streaming response (`Body::from_stream` or `Sse`)
- Each chunk is a JSON line (newline-delimited JSON / NDJSON)
- Set `Content-Type: application/x-ndjson`
- If `?delay=X` is set, sleep X ms between chunks
- This tests how the gateway handles streaming vs buffered responses

#### `/drip`
- Stream individual byte chunks with configurable pauses
- Total response = `bytes` param bytes, delivered in `chunk_size` increments with `delay` ms between each
- Cap total bytes at 10MB, delay at 10000ms, chunk_size minimum 1

#### `/cache/{ttl}`
- Compute a stable ETag from the response content (use a hash)
- Store the server startup time for `Last-Modified`
- Check `If-None-Match` header against ETag → return 304 if match
- Check `If-Modified-Since` against server start time → return 304 if not modified
- Set `Vary: Accept` since content changes based on Accept header
- This endpoint is purpose-built for testing the proxy-cache plugin

### Router integration

Export `pub fn router(config: Arc<Config>) -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -w "\nTime: %{time_total}s\n" http://localhost/delay/500` → ~0.5s response time
3. `curl http://localhost/bytes/1024 | wc -c` → 1024
4. `curl http://localhost/stream/5` → 5 NDJSON lines
5. `curl -i http://localhost/cache/300` → has Cache-Control, ETag, Last-Modified headers
6. `curl -i -H 'If-None-Match: <etag_from_above>' http://localhost/cache/300` → 304
7. `curl http://localhost/response-headers?X-Test=hello` → X-Test header in response
