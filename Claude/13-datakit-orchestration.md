# Prompt 13 — DataKit Orchestration Endpoints

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement a set of chained endpoints that simulate a multi-step backend orchestration flow. These are purpose-built for demonstrating Kong's DataKit plugin, which can call multiple upstream services in sequence, passing data between steps. The endpoints simulate a realistic payment processing pipeline.

## What to build

### File: `src/orchestration.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/orchestration/step/1` | POST | **Authenticate** — validate API key, return merchant metadata and a correlation token |
| `/orchestration/step/2` | POST | **Enrich** — accept correlation token, return enriched data (risk score, geo data, customer tier) |
| `/orchestration/step/3` | POST | **Validate** — accept enriched data, run "validation rules", return approval/denial |
| `/orchestration/step/4` | POST | **Process** — accept validated data, return final transaction result |
| `/orchestration/status` | GET | Returns info about all steps, their expected inputs/outputs, and example payloads |

### Step 1: Authenticate

**Expected input:**
- Header: `X-Api-Key` (any non-empty value accepted)
- Body: `{"merchant_id": "...", "request_type": "payment"}`

**Output (200):**
```json
{
    "step": 1,
    "name": "authenticate",
    "status": "success",
    "correlation_id": "<uuid>",
    "merchant": {
        "id": "<from input>",
        "name": "Demo Merchant Ltd",
        "tier": "gold",
        "mcc": "5411",
        "country": "GB"
    },
    "permissions": ["card_payment", "direct_debit", "refund"],
    "timestamp": "..."
}
```

**Failure:** If `X-Api-Key` is missing, return 401. If body is missing `merchant_id`, return 400.

### Step 2: Enrich

**Expected input:**
- Header: `X-Correlation-Id` (required — from step 1)
- Body: `{"merchant_id": "...", "amount": 5000, "currency": "GBP", "card_bin": "411111"}`

**Output (200):**
```json
{
    "step": 2,
    "name": "enrich",
    "status": "success",
    "correlation_id": "<from header>",
    "enrichment": {
        "risk_score": 15,
        "risk_level": "low",
        "card_type": "visa_credit",
        "issuing_bank": "Demo Bank PLC",
        "issuing_country": "GB",
        "customer_tier": "premium",
        "velocity_check": {
            "transactions_24h": 3,
            "amount_24h": 15000,
            "flagged": false
        }
    },
    "timestamp": "..."
}
```

Generate slightly different risk scores and velocity data using a hash of the correlation ID, so repeated calls with the same ID are consistent.

### Step 3: Validate

**Expected input:**
- Header: `X-Correlation-Id` (required)
- Body: `{"merchant_id": "...", "amount": 5000, "risk_score": 15, "risk_level": "low", "permissions": [...]}`

**Output (200) — approved:**
```json
{
    "step": 3,
    "name": "validate",
    "status": "success",
    "correlation_id": "<from header>",
    "validation": {
        "approved": true,
        "rules_evaluated": 5,
        "rules_passed": 5,
        "rules_failed": 0,
        "applied_rules": [
            {"rule": "amount_limit", "result": "pass", "detail": "Under £50,000 limit"},
            {"rule": "risk_threshold", "result": "pass", "detail": "Score 15 < threshold 70"},
            {"rule": "merchant_active", "result": "pass", "detail": "Merchant is active"},
            {"rule": "permission_check", "result": "pass", "detail": "card_payment permitted"},
            {"rule": "velocity_check", "result": "pass", "detail": "Under 24h velocity limit"}
        ]
    },
    "timestamp": "..."
}
```

**Logic:** If `risk_score` > 70, set `approved: false` and fail the `risk_threshold` rule. If `amount` > 5000000, fail the `amount_limit` rule.

### Step 4: Process

**Expected input:**
- Header: `X-Correlation-Id` (required)
- Header: `X-Validation-Result: approved` (required — from step 3)
- Body: `{"merchant_id": "...", "amount": 5000, "currency": "GBP"}`

**Output (200):**
```json
{
    "step": 4,
    "name": "process",
    "status": "success",
    "correlation_id": "<from header>",
    "transaction": {
        "id": "<uuid>",
        "merchant_id": "<from input>",
        "amount": 5000,
        "currency": "GBP",
        "status": "completed",
        "authorization_code": "AUTH-<6 random alphanumeric>",
        "processor_response": {
            "code": "00",
            "message": "Approved"
        }
    },
    "timestamp": "..."
}
```

**Failure:** If `X-Validation-Result` is not `approved`, return 403 with `{"error": "transaction_denied", "details": "Validation did not approve this transaction"}`.

### Status Endpoint (`/orchestration/status`)

Return a JSON document describing all four steps, their expected inputs (headers + body), and example payloads. This is essentially documentation-as-an-endpoint that helps users configure their DataKit flows.

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. Walk through the full 4-step flow with curl:
   - Step 1: `curl -X POST -H 'X-Api-Key: test' http://localhost/orchestration/step/1 -d '{"merchant_id":"M001","request_type":"payment"}' -H 'Content-Type: application/json'`
   - Step 2: Use correlation_id from step 1
   - Step 3: Use enrichment data from step 2
   - Step 4: Use validation result from step 3
3. Verify failure cases: missing API key, high risk score, missing validation header
4. `curl http://localhost/orchestration/status` → shows all step documentation
