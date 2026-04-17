# Prompt 15 — Upstream Identity

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement an identity endpoint that reveals which instance is responding. When you run multiple Rustybin instances behind Kong's upstream load balancer, this endpoint lets you visually demonstrate round-robin, consistent hashing, canary weight shifting, and blue/green deployments.

## What to build

### File: `src/identity.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/identity` | ALL methods | Return instance identity and request metadata |

### Response

```json
{
    "instance_id": "rustybin-01",
    "hostname": "abc123def",
    "version": "0.1.0",
    "uptime_seconds": 3456,
    "request_count": 142,
    "port": {
        "http": 80,
        "https": 443
    },
    "environment": {
        "rust_version": "1.82.0",
        "profile": "release"
    },
    "request": {
        "remote_ip": "192.168.1.100",
        "forwarded_for": "10.0.0.1",
        "host": "api.example.com",
        "via": "kong/3.9.0"
    },
    "timestamp": "2026-04-14T15:30:00Z"
}
```

### Behaviour

1. **`instance_id`**: From `RUSTYBIN_INSTANCE_ID` config (auto-generated UUID if not set)
2. **`hostname`**: From `gethostname()` system call
3. **`version`**: From `CARGO_PKG_VERSION`
4. **`uptime_seconds`**: Time since server start (store startup `Instant` in state)
5. **`request_count`**: Atomic counter incremented on every request to this endpoint. Use `AtomicU64`.
6. **`request.remote_ip`**: Peer IP
7. **`request.forwarded_for`**: Raw `X-Forwarded-For` value if present
8. **`request.host`**: From `Host` header
9. **`request.via`**: From `Via` header if present (Kong sets this)
10. Support content negotiation (JSON/XML)

### State

```rust
pub struct IdentityState {
    pub start_time: Instant,
    pub request_count: AtomicU64,
}
```

### Docker Compose addition

Add a comment block to `docker-compose.yml` showing how to run multiple instances for load balancing demos:

```yaml
# Uncomment to run multiple instances for load balancing demos:
#  rustybin-02:
#    build: .
#    container_name: rustybin-02
#    environment:
#      - RUSTYBIN_INSTANCE_ID=rustybin-02
#      - RUSTYBIN_HTTP_PORT=80
#    ports:
#      - "8081:80"
#
#  rustybin-03:
#    build: .
#    container_name: rustybin-03
#    environment:
#      - RUSTYBIN_INSTANCE_ID=rustybin-03
#      - RUSTYBIN_HTTP_PORT=80
#    ports:
#      - "8082:80"
```

### Router integration

Export `pub fn router(config: Arc<Config>, identity_state: Arc<IdentityState>) -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/identity` → shows instance ID, hostname, uptime
3. `curl http://localhost/identity` again → request_count incremented
4. Set `RUSTYBIN_INSTANCE_ID=test-01` → shows in response
