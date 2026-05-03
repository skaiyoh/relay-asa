# API Integration

> External APIs called — endpoints, auth, request/response shapes, error handling, rate limits, retry policy.

## 1. Perfect Panel Admin API

**Purpose:** sync support tickets, fetch fresh ticket detail, and send admin replies.
**Auth:** API key header.
**HTTP client:** OkHttp 4.12.0 with Jackson 2.17.0.
**Rate limits:** 5 requests per second per panel. Full ticket fetches are queued; user-triggered work outranks background sync.

### Endpoints used

| Method | Path | Purpose | Errors handled |
| ------ | ---- | ------- | -------------- |
| `GET` | `/tickets` | Background list sync | Network failures, non-200, throttling |
| `GET` | `/tickets/{ticket_id}` | Fresh full ticket fetch before view, draft, or send | Network failures, non-200, throttling, malformed payloads |
| `POST` | `/tickets/{ticket_id}/reply` | Send a message from an admin | Network failures, non-200, validation errors, throttling |
| `GET` | `/orders/{order_id}` | Fetch order details for support agent tools | Network failures, non-200, ownership mismatch |

### Reply request shape

```json
{
  "staff_name": "Admin",
  "message": "Hello, thanks for reaching out..."
}
```

### Reply response

```json
{
  "data": { "ticket_id": 255 },
  "error_message": null,
  "error_code": null
}
```

The reply response confirms only the ticket ID — it does not return the created message. The app must immediately re-fetch the ticket after sending.

### Error handling

All failures surface as checked `PanelApiException` carrying HTTP status code and response body. The client does not implement retry or rate limiting — those are owned by dedicated services. Send failure diagnostics include HTTP status and a truncated response snippet (max 200 chars) without exposing API keys.

### Order lookup diagnostics

| Failure mode | Diagnostic shown |
| ------------ | ---------------- |
| HTTP error (non-404) | `HTTP {status} from /orders/{id} — {body snippet}` |
| Network error | `Network error: {cause} — path: /orders/{id}` |
| Malformed response | `Malformed response for order {id}: expected vs actual` |
| Ownership mismatch | `Ownership mismatch — ticket user: {x} / order user: {y}` |

Body snippets are truncated to 250 characters. JSON responses containing a `"data"` field are suppressed to prevent leaking other customers' data.

## 2. Generic Service API

**Purpose:** fetch the panel's service catalog for agent lookup and order enrichment.
**Auth:** request parameter key.
**First-slice action:** `services` only (read-only catalog fetch).

| Method | Purpose | Request | Response |
| ------ | ------- | ------- | -------- |
| `POST` | Fetch all services | Form-encoded `key` + `action=services` | Bare JSON array of service entries |

Row-by-row parsing: entries with invalid or missing service IDs are skipped rather than failing the entire batch. Service API write/action endpoints (`add`, `refill`, `cancel`, `status`, `balance`) are explicitly out of scope for V1.

## 3. OpenAI API

**Purpose:** generate AI-assisted draft replies and intent classifications.
**Endpoint:** `POST /v1/responses` (Responses API).
**Auth:** bearer token.

### Request policy

- Strict structured output via `text.format` JSON schema (`type=json_schema`, `strict=true`)
- Schema requires `response`, `confidence`, `intent`, and `should_reply` fields
- `store=false` — raw responses stored locally, not on OpenAI
- No streaming, no model-side tools, no conversation state

### Response policy

- Accept only `completed` responses with valid structured output
- Store raw response payload locally before parsing
- Treat refusals, empty output, schema violations, and malformed JSON as failures — leave ticket in human review

### Retry policy

- At most 3 total attempts per draft generation
- Retry only transport failures, timeouts, HTTP 408/409/429/5xx
- Do not retry auth failures, validation errors, quota exhaustion, refusals, or malformed output
- `Retry-After` header support capped at 30 seconds
- Full-jitter exponential backoff: `random(0, min(15s, 1s × 2^n))`
- Stop when next attempt would exceed 90-second operation budget

### Timeout policy

- Connect: 10s
- Write: 10s
- Read: 45s per attempt
- Total operation budget: 90s (all attempts + backoff)
