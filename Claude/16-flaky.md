# Prompt 16 — Flaky / Unreliable Endpoint

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement endpoints that simulate unreliable upstream behaviour. These are essential for demonstrating Kong's circuit breaker, health checks (active and passive), retry logic, and request-termination plugins.

## What to build

### File: `src/flaky.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/flaky/{fail_rate}` | ALL | Fail `fail_rate`% of the time with 503. `fail_rate` is 0–100. |
| `/flaky/pattern/{pattern}` | ALL | Follow a deterministic success/fail pattern. E.g. `SSFSS` = success, success, fail, success, success, then repeat. |
| `/flaky/after/{n}` | ALL | Succeed for the first `n` requests, then fail forever (until reset). Simulates an upstream that goes down. |
| `/flaky/recover/{n}` | ALL | Fail for the first `n` requests, then succeed forever. Simulates an upstream recovering. |
| `/flaky/reset` | POST | Reset all counters for `/flaky/after` and `/flaky/recover`. |
| `/flaky/status` | GET | Return current state of all flaky counters and patterns. |

### Behaviour

#### `/flaky/{fail_rate}`
- Parse `fail_rate` as integer 0–100
- Generate random number 0–99
- If random < fail_rate → return 503 with `{"error": "service_unavailable", "fail_rate": N, "message": "Simulated failure (N% fail rate)"}`
- Otherwise → return 200 with `{"status": "ok", "fail_rate": N, "message": "Request succeeded", "request_number": <counter>}`
- Include `X-Rustybin-Flaky: true` and `X-Rustybin-Fail-Rate: {N}` headers on all responses
- Include `Retry-After: 1` header on 503 responses (helps test retry plugins)

#### `/flaky/pattern/{pattern}`
- `pattern` is a string of `S` (success) and `F` (fail) characters, e.g. `SSFSF`
- Maintain a global atomic counter. Each request advances the counter.
- Use `counter % pattern.len()` to determine the current position
- `S` → 200, `F` → 503
- Validate pattern contains only `S` and `F`, return 400 otherwise
- Response body includes current position and the full pattern

#### `/flaky/after/{n}`
- Maintain an atomic request counter
- Requests 1 through `n` → 200
- Requests `n+1` onwards → 503
- Response includes `{"request_number": X, "threshold": N, "will_fail_after": N}`

#### `/flaky/recover/{n}`
- Requests 1 through `n` → 503
- Requests `n+1` onwards → 200
- Response includes `{"request_number": X, "threshold": N, "will_recover_after": N}`

#### `/flaky/reset`
- Reset all counters for `after` and `recover` endpoints back to 0
- Return `{"status": "reset", "message": "All flaky counters reset"}`

#### `/flaky/status`
- Return current values of all counters and the current pattern position
- Useful for debugging during demos

### State

```rust
pub struct FlakyState {
    pub random_counters: DashMap<u8, AtomicU64>,     // fail_rate → request count
    pub pattern_counter: AtomicU64,
    pub after_counter: AtomicU64,
    pub recover_counter: AtomicU64,
}
```

Use `DashMap` or a simpler structure as appropriate.

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/flaky/50` → roughly 50% success/failure over many requests
3. `curl http://localhost/flaky/0` → always succeeds
4. `curl http://localhost/flaky/100` → always fails with 503
5. `curl http://localhost/flaky/pattern/SFS` → success, fail, success, success, fail, success...
6. Hit `/flaky/after/3` four times → first 3 succeed, 4th fails
7. `curl -X POST http://localhost/flaky/reset` → resets counters
8. `curl http://localhost/flaky/status` → shows all counters
