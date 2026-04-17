# Prompt 10 — Auth: mTLS Client Certificate

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types (`AuthResponse`), content negotiation helper, TLS setup in `main.rs`, and how routers are merged.

## Goal

Implement mutual TLS authentication endpoints. These test Kong's mTLS plugin, client certificate validation, and certificate-based routing.

## What to build

### File: `src/auth_mtls.rs`

### CA and Client Cert Generation

At startup (in `main.rs` or a dedicated `src/certs.rs` module), generate a demo PKI if certs don't already exist:

1. **CA cert + key** — self-signed root CA (`CN=Rustybin Demo CA`)
2. **Server cert + key** — signed by the CA, SANs: `localhost`, `127.0.0.1`, `rustybin`, `*.rustybin.local`
3. **Client cert + key** — signed by the CA, `CN=demo-client`, `O=Rustybin Demo`

Use the `rcgen` crate for generation. Store the generated PEM files and also keep the client cert/key in memory for the download endpoint.

Create a shared `CertState`:
```rust
pub struct CertState {
    pub ca_cert_pem: String,
    pub client_cert_pem: String,
    pub client_key_pem: String,
}
```

### Environment variable: `RUSTYBIN_MTLS_IN_HEADER`

When set to a header name (e.g. `X-Client-Cert`), the mTLS endpoint reads the client certificate as URL-encoded PEM from that header instead of from the TLS handshake. This is critical for Kong demos because Kong terminates TLS and forwards the client cert in a header.

Add this to the config.

### Routes

All routes support **all HTTP methods**: GET, POST, PUT, PATCH, DELETE.

| Route | Method | Behaviour |
|---|---|---|
| `/auth/mtls` | ALL | Validate client certificate. Check TLS peer cert OR the configured header. |
| `/auth/mtls/get-client-cert` | GET | Download the demo client cert and key as JSON with `cert_pem` and `key_pem` fields. |
| `/auth/mtls/get-ca-cert` | GET | Download the CA certificate PEM. Useful for configuring Kong's trusted CA. |

### Behaviour — mTLS Validation (`/auth/mtls`)

1. **Header mode** (if `RUSTYBIN_MTLS_IN_HEADER` is set):
   - Read the specified header
   - URL-decode the value to get PEM
   - Parse the PEM to extract DN and issuer
   
2. **TLS mode** (default):
   - Extract the peer certificate from the TLS connection
   - This requires the HTTPS listener to be configured with `rustls` requesting (but not requiring) client certs, with the demo CA as the trusted root

3. **Success (200):** Return `AuthResponse` with:
   - `authenticated: true`
   - `auth_type: "mtls"`
   - `client_dn`: the subject DN of the client cert (e.g. `CN=demo-client, O=Rustybin Demo`)
   - `client_ca`: the issuer DN (e.g. `CN=Rustybin Demo CA`)

4. **Failure (401):** Return `{"authenticated": false, "error": "unauthorized"}` with details about what's missing (no cert, invalid cert, wrong CA).

### Behaviour — Get Client Cert (`/auth/mtls/get-client-cert`)

Return:
```json
{
    "cert_pem": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "key_pem": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----",
    "usage": "curl --cert client.crt --key client.key https://localhost:443/auth/mtls"
}
```

### Behaviour — Get CA Cert (`/auth/mtls/get-ca-cert`)

Return:
```json
{
    "ca_cert_pem": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "usage": "Configure this as the trusted CA in your API gateway"
}
```

### TLS Configuration Update

Update the HTTPS listener in `main.rs` to:
- Use the generated server cert
- Configure `rustls` to optionally request client certificates (not require — so non-mTLS HTTPS still works)
- Set the demo CA as the trusted client CA root

### Router integration

Export `pub fn router(cert_state: Arc<CertState>) -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl http://localhost/auth/mtls/get-client-cert` → returns cert and key PEM
3. Save cert/key to files, then: `curl --cert client.crt --key client.key --cacert ca.crt https://localhost:443/auth/mtls` → 200, authenticated
4. `curl https://localhost:443/auth/mtls --cacert ca.crt` → 401 (no client cert)
5. Test header mode: set `RUSTYBIN_MTLS_IN_HEADER=X-Client-Cert`, then `curl -H "X-Client-Cert: <url-encoded-pem>" http://localhost/auth/mtls` → 200
