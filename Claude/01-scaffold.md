# Prompt 01 — Project Scaffold

## Goal

Create the Rustybin project from scratch: Cargo workspace, dependencies, main server entrypoint, Docker Compose, self-signed TLS certs, and a health check endpoint.

## What to build

### 1. Project structure

```
rustybin/
├── Cargo.toml
├── Cargo.lock
├── Dockerfile
├── docker-compose.yml
├── certs/
│   └── generate-certs.sh      # script to generate self-signed TLS certs
├── src/
│   ├── main.rs                 # entrypoint, server startup, router assembly
│   ├── config.rs               # env-based configuration
│   ├── types.rs                # shared response types, error types
│   ├── content_negotiation.rs  # JSON/XML response helper
│   ├── logging.rs              # tracing setup + request logging middleware
│   └── health.rs               # GET /health
```

### 2. Cargo.toml dependencies

Use these crates (pin to latest stable versions):

- `axum` — HTTP framework (with `ws` feature for later)
- `tokio` — async runtime (full features)
- `serde` / `serde_json` — JSON serialisation
- `quick-xml` with `serde` feature — XML serialisation
- `tracing` / `tracing-subscriber` — structured logging
- `tower` / `tower-http` — middleware (cors, trace, compression)
- `thiserror` — error types
- `uuid` — UUID generation (v4)
- `chrono` — date/time
- `base64` — encoding
- `jsonwebtoken` — JWT signing/verification
- `rand` — random number generation
- `rcgen` — TLS cert generation at build/startup
- `tokio-rustls` / `rustls` / `rustls-pemfile` — TLS termination
- `axum-server` with `tls-rustls` feature — serving HTTPS

### 3. `src/config.rs`

Read configuration from environment variables with sensible defaults:

| Variable | Default | Description |
|---|---|---|
| `RUSTYBIN_HTTP_PORT` | `80` | HTTP listen port |
| `RUSTYBIN_HTTPS_PORT` | `443` | HTTPS listen port |
| `RUSTYBIN_HOST` | `0.0.0.0` | Bind address |
| `RUSTYBIN_LOG_LEVEL` | `info` | tracing log level |
| `RUSTYBIN_TRUST_FORWARD` | `false` | Trust X-Forwarded-For for IP endpoints |
| `RUSTYBIN_BODY_LIMIT` | `1048576` | Max body capture in bytes (1MB) |
| `RUSTYBIN_INSTANCE_ID` | (auto UUID) | Instance identifier for `/identity` |
| `RUSTYBIN_TLS_CERT` | `certs/server.crt` | Path to TLS cert |
| `RUSTYBIN_TLS_KEY` | `certs/server.key` | Path to TLS key |

### 4. `src/main.rs`

- Parse config
- Initialise tracing
- Build the axum router by merging sub-routers: `health::router()`
- Spawn two tasks: one for HTTP on port 80, one for HTTPS on port 443
- Both use the same router
- Log startup banner with version, ports, instance ID

### 5. `src/types.rs`

Define shared types:

```rust
// Standard error response used across all endpoints
#[derive(Serialize, Deserialize)]
pub struct ErrorResponse {
    pub error: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<String>,
}

// Wrapper for content-negotiated responses
// (will be expanded as modules are added)
```

### 6. `src/content_negotiation.rs`

Create a helper function (or extractor) that:
- Checks the `Accept` header
- If `application/xml` → serialise to XML using `quick-xml`
- Otherwise → serialise to JSON (default)
- Returns the correct `Content-Type` header
- All endpoints will use this instead of `Json()` directly

Signature something like:
```rust
pub fn negotiate<T: Serialize>(accept: &HeaderValue, data: &T) -> Response
```

### 7. `src/logging.rs`

- Configure `tracing_subscriber` with env filter from config
- Create a tower middleware layer that logs every request: method, path, status, duration, key headers (host, user-agent, x-forwarded-for)

### 8. `src/health.rs`

```
GET /health → {"status": "healthy", "service": "rustybin", "version": "0.1.0", "instance_id": "..."}
```

- Include the version from `Cargo.toml` using `env!("CARGO_PKG_VERSION")`
- Include the instance_id from config
- Support content negotiation (JSON/XML)

### 9. `certs/generate-certs.sh`

A bash script that uses `openssl` to generate:
- A self-signed CA cert and key
- A server cert signed by that CA (with SANs for `localhost`, `127.0.0.1`, `rustybin`)
- A client cert signed by that CA (for mTLS testing later)

Output files: `ca.crt`, `ca.key`, `server.crt`, `server.key`, `client.crt`, `client.key`

### 10. `Dockerfile`

Multi-stage build:
- **Stage 1 (builder):** Use `rust:1.82-slim` (or latest stable). Copy source, run `cargo build --release`.
- **Stage 2 (runtime):** Use `debian:bookworm-slim`. Copy the binary, certs script, and a minimal set of runtime deps (`ca-certificates`, `openssl`).
- Generate certs at build time OR have an entrypoint script that generates them on first run if they don't exist.
- Expose ports 80 and 443.
- Set `RUST_LOG=info` as default.

### 11. `docker-compose.yml`

```yaml
services:
  rustybin:
    build: .
    container_name: rustybin
    ports:
      - "80:80"
      - "443:443"
    environment:
      - RUSTYBIN_LOG_LEVEL=info
      - RUSTYBIN_INSTANCE_ID=rustybin-01
    volumes:
      - ./certs:/app/certs
    restart: unless-stopped
```

## Verification

Before finishing:
1. Run `cargo build` — must compile with zero errors
2. Run `cargo clippy` — fix any warnings
3. Verify `GET /health` returns valid JSON with all expected fields
4. Verify the Docker build completes successfully
