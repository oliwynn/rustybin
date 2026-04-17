# Prompt 06 — Info & Random Generators

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement informational endpoints (IP, date, time) and random value generators. These provide useful utility for demos and test payloads.

## What to build

### File: `src/info.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/ip` | GET | Returns caller's IPv4 and IPv6 as `{"ipv4": "...", "ipv6": null}`. Respects `RUSTYBIN_TRUST_FORWARD` (reads `X-Forwarded-For`). |
| `/ip/v4` | GET | IPv4 only. |
| `/ip/v6` | GET | IPv6 only. |
| `/date` | GET | Current date in UTC: `{"date": "2026-04-14", "timezone": "UTC"}` |
| `/date/{timezone}` | GET | Current date in given IANA timezone. Return 404 for unknown timezone. Use `chrono-tz` crate. |
| `/time` | GET | Current time in UTC: `{"time": "2026-04-14T15:30:00Z", "timezone": "UTC"}` |
| `/time/{timezone}` | GET | Current time in given IANA timezone. Return 404 for unknown timezone. |

### File: `src/random.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/uuid` | GET | Random UUID v4: `{"uuid": "550e8400-..."}` |
| `/guuid` | GET | Random GUID (UUID v4 in curly braces): `{"guuid": "{550e8400-...}"}` |
| `/random` | GET | Sample of all types: `{"int": ..., "uint": ..., "uuid": "...", "guuid": "...", "lorem_ipsum": "..."}` |
| `/random/int` | GET | Random signed integer in [-32000, 32000]: `{"value": 12345}` |
| `/random/int/{lower}/{upper}` | GET | Random integer in custom range. Return 400 if lower >= upper. |
| `/random/uint` | GET | Random unsigned integer in [0, 65535]: `{"value": 42000}` |
| `/random/lorem-ipsum` | GET | One paragraph of Lorem Ipsum: `{"paragraphs": ["Lorem ipsum..."]}` |
| `/random/lorem-ipsum/{count}` | GET | Up to 32 paragraphs. Return 400 if count is 0 or > 32. |

### File: `src/image.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/image/png` | GET | Return a small demo PNG image (generate a simple coloured rectangle at startup using pure Rust, or embed a minimal PNG as a `const` byte array). Set `Content-Type: image/png`. |
| `/image/jpeg` | GET | Same but JPEG. |
| `/image/gif` | GET | Same but GIF. |

### Behaviour details

#### IP detection
- Parse the peer socket address from `ConnectInfo`
- If `trust_forward` is true, prefer the first IP in `X-Forwarded-For`
- Classify IPs into v4/v6 fields
- Handle the case where the peer is `::ffff:127.0.0.1` (IPv4-mapped IPv6) — extract the v4

#### Lorem Ipsum
- Embed 5-8 standard Lorem Ipsum paragraphs as constants
- When asked for more paragraphs than available, cycle through them
- Each paragraph should be 50-100 words

#### Images
- The simplest approach: embed minimal valid PNG, JPEG, and GIF byte arrays as `const` in the source
- These can be tiny (e.g. 8x8 pixel single-colour images)
- The point is to have a valid image response for testing image proxy/transformation

### Dependencies to add

- `chrono-tz` — for timezone support in date/time endpoints

### Router integration

Export `pub fn info_router(config: Arc<Config>) -> Router`, `pub fn random_router() -> Router`, and `pub fn image_router() -> Router`. Merge all into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/ip` → shows IP
3. `curl http://localhost/date/America/New_York` → date in EST/EDT
4. `curl http://localhost/time` → current UTC time
5. `curl http://localhost/uuid` → valid UUID
6. `curl http://localhost/random` → all random types
7. `curl http://localhost/random/lorem-ipsum/3` → 3 paragraphs
8. `curl -o test.png http://localhost/image/png` → valid PNG file
9. `curl -H 'Accept: application/xml' http://localhost/date` → XML response
