# Prompt 14 — SOAP / XML Endpoint

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, and how routers are merged in `main.rs`.

## Goal

Implement a SOAP endpoint that accepts SOAP envelopes and returns SOAP responses. This tests Kong's xml-threat-protection plugin, request-transformer-advanced (XML↔JSON conversion), and any SOAP-to-REST transformation patterns customers want to demo.

## What to build

### File: `src/soap.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/soap` | POST | Accept a SOAP envelope, parse the operation, return a SOAP response |
| `/soap/wsdl` | GET | Return a WSDL document describing the available SOAP operations |

### SOAP Operations

Support these operations inside a `<soap:Body>`:

**1. `GetUser`**
```xml
<GetUser xmlns="http://rustybin.local/users">
    <userId>123</userId>
</GetUser>
```
Response:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <GetUserResponse xmlns="http://rustybin.local/users">
            <user>
                <id>123</id>
                <name>Jane Smith</name>
                <email>jane@rustybin.local</email>
                <role>ADMIN</role>
            </user>
        </GetUserResponse>
    </soap:Body>
</soap:Envelope>
```

**2. `CreateOrder`**
```xml
<CreateOrder xmlns="http://rustybin.local/orders">
    <merchantId>M001</merchantId>
    <amount>5000</amount>
    <currency>GBP</currency>
</CreateOrder>
```
Response includes a generated order ID and timestamp.

**3. `GetStatus`**
```xml
<GetStatus xmlns="http://rustybin.local/status">
    <orderId>ORD-001</orderId>
</GetStatus>
```
Response includes order status, timestamps, and tracking info.

### SOAP Fault Handling

If the operation is unrecognised or the XML is malformed, return a proper SOAP Fault:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <soap:Fault>
            <faultcode>soap:Client</faultcode>
            <faultstring>Unknown operation</faultstring>
            <detail>The requested operation 'Foo' is not supported. Available operations: GetUser, CreateOrder, GetStatus</detail>
        </soap:Fault>
    </soap:Body>
</soap:Envelope>
```

### WSDL (`/soap/wsdl`)

Return a valid WSDL 1.1 document that describes all three operations with proper message types, port types, bindings, and service location. Set `Content-Type: text/xml`.

### Parsing approach

Use `quick-xml` to parse the incoming SOAP envelope. Extract the first child element of `<soap:Body>` to determine the operation. You don't need a full SOAP framework — simple XML parsing is sufficient.

Set `Content-Type: text/xml; charset=utf-8` on all SOAP responses.
Add `SOAPAction` header echoing in responses where present in requests.

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs`.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -X POST http://localhost/soap -H 'Content-Type: text/xml' -d '<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"><soap:Body><GetUser xmlns="http://rustybin.local/users"><userId>123</userId></GetUser></soap:Body></soap:Envelope>'` → valid SOAP response
3. `curl http://localhost/soap/wsdl` → valid WSDL document
4. Send an unknown operation → proper SOAP Fault
5. Send malformed XML → proper SOAP Fault with `faultcode=soap:Client`
