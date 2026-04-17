# Prompt 05 — Redirects & Cookies

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement redirect chains and cookie management endpoints. These test how the gateway handles redirect following, header propagation, and the session plugin lifecycle.

## What to build

### File: `src/redirects.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/redirect/{n}` | GET | Redirect `n` times (302) then land on `/echo`. Max 20. `/redirect/3` → 302 to `/redirect/2` → 302 to `/redirect/1` → 302 to `/echo`. |
| `/redirect-to` | GET | Redirect to arbitrary URL. Query params: `url` (required), `status` (default 302). Validates status is a redirect code (301, 302, 303, 307, 308). |
| `/absolute-redirect/{n}` | GET | Same as `/redirect/{n}` but uses absolute URLs in the `Location` header (uses `Host` header to construct). |

### File: `src/cookies.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/cookies` | GET | Returns all cookies received in the request as JSON: `{"cookies": {"name": "value", ...}}` |
| `/cookies/set` | GET | Sets cookies from query params. `?name=value&foo=bar` sets two cookies. Returns 302 redirect to `/cookies` so you immediately see the result. Supports optional params: `?_path=/`, `?_domain=example.com`, `?_secure=true`, `?_httponly=true`, `?_samesite=Lax`, `?_maxage=3600` (prefixed with `_` to distinguish from cookie names). |
| `/cookies/set/{name}/{value}` | GET | Set a single cookie. Returns 302 redirect to `/cookies`. |
| `/cookies/delete` | GET | Deletes cookies listed in query params. `?name&foo` deletes those cookies by setting them with `Max-Age=0`. Redirects to `/cookies`. |

### Behaviour details

#### Redirects
- Each redirect decrements the counter in the path
- Preserve query parameters through the chain
- Set `X-Redirect-Count` header showing remaining hops
- If `n` > 20, return 400 with error about max redirects
- If `n` = 0, return 400 — use `/echo` directly

#### Cookies
- Parse cookies from the `Cookie` header using standard parsing
- When setting cookies, build proper `Set-Cookie` headers with all attributes
- The `_` prefixed query params control cookie attributes and apply to ALL cookies being set in that request
- Content negotiation on `/cookies` response (JSON/XML)

### Router integration

Export `pub fn redirects_router() -> Router` and `pub fn cookies_router() -> Router`. Merge both into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -L -v http://localhost/redirect/3` → follow 3 redirects, end at `/echo`
3. `curl -i http://localhost/redirect-to?url=http://example.com&status=307` → 307 with Location
4. `curl -c cookies.txt http://localhost/cookies/set?session=abc123&theme=dark` → sets cookies, redirects
5. `curl -b cookies.txt http://localhost/cookies` → shows `{"cookies": {"session": "abc123", "theme": "dark"}}`
6. `curl -b cookies.txt -c cookies.txt http://localhost/cookies/delete?session` → deletes session cookie
