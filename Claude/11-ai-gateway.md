# Prompt 11 — AI Gateway: Mock LLM Endpoints

## Context

You are working on the Rustybin project — a Rust/axum HTTP stub service. Read the existing codebase in `src/` to understand the project structure, shared types, content negotiation helper, and how routers are merged in `main.rs`.

## Goal

Implement OpenAI-compatible mock LLM endpoints. These are critical for demonstrating Kong's AI Gateway plugins: ai-proxy, ai-prompt-guard, ai-rate-limiting (by tokens), ai-semantic-cache, ai-request-transformer, and ai-response-transformer. The mock must return realistic response shapes with token counting so the gateway plugins behave as they would against a real LLM.

## What to build

### File: `src/ai_gateway.rs`

### Routes

| Route | Method | Behaviour |
|---|---|---|
| `/ai/v1/chat/completions` | POST | OpenAI-compatible chat completions (streaming + non-streaming) |
| `/ai/v1/completions` | POST | Legacy completions format |
| `/ai/v1/embeddings` | POST | Return mock embedding vectors |
| `/ai/v1/models` | GET | List available "models" |

### Chat Completions (`/ai/v1/chat/completions`)

**Request format** (OpenAI-compatible):
```json
{
    "model": "rustybin-gpt",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello, how are you?"}
    ],
    "temperature": 0.7,
    "max_tokens": 100,
    "stream": false
}
```

**Non-streaming response (200):**
```json
{
    "id": "chatcmpl-{uuid}",
    "object": "chat.completion",
    "created": 1700000000,
    "model": "rustybin-gpt",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Hello! I'm Rustybin, a mock AI endpoint for API gateway testing. I received your message and I'm responding with a canned response. This is useful for testing AI proxy plugins, rate limiting by token count, prompt guardrails, and semantic caching."
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": <calculated>,
        "completion_tokens": <calculated>,
        "total_tokens": <calculated>
    }
}
```

**Token counting**: Approximate token count by splitting on whitespace and punctuation. Count prompt tokens from all input messages, completion tokens from the response. This doesn't need to match tiktoken exactly — it just needs to be consistent and realistic enough for the gateway's token-based rate limiting to work.

**Streaming response** (when `stream: true`):

Return `Content-Type: text/event-stream` with SSE chunks:

```
data: {"id":"chatcmpl-{uuid}","object":"chat.completion.chunk","created":1700000000,"model":"rustybin-gpt","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-{uuid}","object":"chat.completion.chunk","created":1700000000,"model":"rustybin-gpt","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-{uuid}","object":"chat.completion.chunk","created":1700000000,"model":"rustybin-gpt","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

... (one word per chunk)

data: {"id":"chatcmpl-{uuid}","object":"chat.completion.chunk","created":1700000000,"model":"rustybin-gpt","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

- Stream the canned response word by word with ~50ms delay between chunks
- This realistically simulates LLM token streaming
- The gateway's streaming proxy and SSE handling are exercised

**Canned responses**: Build a small map of canned responses keyed by simple keyword matching in the last user message. For example:
- Contains "hello" / "hi" → friendly greeting response
- Contains "code" / "python" / "function" → a short code example response
- Contains "json" / "data" → a structured data response
- Contains "long" / "essay" → a longer multi-paragraph response
- Default → a generic helpful response about being a mock endpoint

**Configurable delay**: Support query param `?delay=500` to add artificial latency (in ms) before responding. Useful for timeout testing.

### Legacy Completions (`/ai/v1/completions`)

**Request:**
```json
{
    "model": "rustybin-gpt",
    "prompt": "Once upon a time",
    "max_tokens": 50
}
```

**Response:**
```json
{
    "id": "cmpl-{uuid}",
    "object": "text_completion",
    "created": 1700000000,
    "model": "rustybin-gpt",
    "choices": [
        {
            "text": "...a mock API endpoint lived in a container...",
            "index": 0,
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": <calculated>,
        "completion_tokens": <calculated>,
        "total_tokens": <calculated>
    }
}
```

### Embeddings (`/ai/v1/embeddings`)

**Request:**
```json
{
    "model": "rustybin-embed",
    "input": "The quick brown fox"
}
```

Also accept `input` as an array of strings.

**Response:**
```json
{
    "object": "list",
    "data": [
        {
            "object": "embedding",
            "index": 0,
            "embedding": [0.0023, -0.0091, 0.0152, ...]
        }
    ],
    "model": "rustybin-embed",
    "usage": {
        "prompt_tokens": <calculated>,
        "total_tokens": <calculated>
    }
}
```

Generate a deterministic 1536-dimension vector from a hash of the input text. This means identical inputs always produce identical embeddings — which is what the semantic cache plugin needs to work correctly.

### Models (`/ai/v1/models`)

```json
{
    "object": "list",
    "data": [
        {"id": "rustybin-gpt", "object": "model", "created": 1700000000, "owned_by": "rustybin"},
        {"id": "rustybin-gpt-fast", "object": "model", "created": 1700000000, "owned_by": "rustybin"},
        {"id": "rustybin-embed", "object": "model", "created": 1700000000, "owned_by": "rustybin"}
    ]
}
```

### Error handling

If the request body is malformed or missing required fields, return OpenAI-compatible errors:
```json
{
    "error": {
        "message": "...",
        "type": "invalid_request_error",
        "param": null,
        "code": null
    }
}
```

### Router integration

Export `pub fn router() -> Router`. Merge into app router in `main.rs` under the `/ai` prefix.

## Verification

1. `cargo build` — compiles cleanly
2. `curl -X POST http://localhost/ai/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"rustybin-gpt","messages":[{"role":"user","content":"hello"}]}'` → 200, valid chat completion with usage
3. Same request with `"stream": true` → SSE stream with word-by-word chunks ending in `[DONE]`
4. `curl -X POST http://localhost/ai/v1/embeddings -H 'Content-Type: application/json' -d '{"model":"rustybin-embed","input":"test"}'` → 1536-dim embedding vector
5. Same input twice → identical embedding vectors (deterministic)
6. `curl http://localhost/ai/v1/models` → model list
7. Verify token counts are present and reasonable in all responses
