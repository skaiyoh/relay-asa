# AI Behavior

> Prompts, models, structured output schemas, confidence thresholds, fallback behavior, safety policy.

## 1. Models in Use

| Purpose | Provider | Model |
| ------- | -------- | ----- |
| Ticket draft generation and intent classification | OpenAI | Configurable via environment variable (e.g. `gpt-4o`) |

## 2. Prompt Contract — `ticket_reply_draft`

**Goal:** read the full fresh-fetched ticket conversation, classify the latest customer intent, and produce a human-reviewable draft reply.

**Inputs:**
- Full ticket conversation history from local storage
- Current ticket metadata
- User-editable business context and rules
- Optional verified order details with linked service catalog facts and agent notes
- Optional ticket-level context note

**Output contract:** strict JSON object:

| Field | Type | Required | Semantics |
| ----- | ---- | -------- | --------- |
| `response` | string | Yes | Human-reviewable draft text. Always populated, even when `should_reply` is `false`. |
| `confidence` | number | Yes | Model-estimated confidence from 0.0 through 1.0. |
| `intent` | string enum | Yes | One of the supported intent labels. |
| `should_reply` | boolean | Yes | Advisory recommendation. `true` = safe draft available; `false` = manual handling recommended but best-effort draft still provided. |

**Example output:**

```json
{
  "response": "Hello, thanks for reaching out. Please allow a little more time for the order to begin.",
  "confidence": 0.91,
  "intent": "order_status",
  "should_reply": true
}
```

## 3. Intent Taxonomy

Supported intent labels:

- `add_funds`
- `payment_question`
- `order_status`
- `order_speedup`
- `refund_request`
- `cancel_request`
- `refill_request`
- `api_help`
- `service_question`
- `greeting`
- `unknown`

V1 does not allow additional model-output labels. High-risk messages use the closest supported label and set `should_reply=false`.

## 4. Reply Suppression

The model returns `should_reply=false` for:

- Refund, cancellation, refill, replacement, chargeback, compensation requests
- Account-balance corrections, external-provider checks, order changes
- Business exceptions
- Hostile, threatening, or legally sensitive messages
- Staff-authored or insufficient-context messages

In all cases the model still produces a best-effort `response` draft. The operator can choose to use the suppressed draft after review.

**Exception — cancelled-order inquiry:** when verified order context shows an already-cancelled order and the customer is asking about that order's status (not requesting a new cancellation), the model treats it as `order_status` with `should_reply=true`.

## 5. Order and Service Grounding

The prompt assembler optionally receives verified order details and linked service catalog facts:

- **Per-verified-order:** ID, status, charge, link, quantity, start/remains, service info, provider, timestamps, agent note
- **Per-unavailable-order:** order ID and state label only (blocked, not_found, api_error, ambiguous, unresolved)
- **Service catalog facts:** name, type, category, rate, min/max, refill/cancel
- **Freshness metadata:** when data was last fetched, catalog staleness

Safety rules enforced by code:
- Only ownership-verified orders included in model context
- Wrong-user, blocked, unresolved orders excluded
- Agent notes used as context but not echoed verbatim
- `should_reply=false` maintained for refund/cancel/refill requests regardless of order context
- No model-side tools or function calls — the application handles all retrieval and verification

## 6. Ticket-Level Context Notes

Optional notes attached to tickets (independent of orders) that provide additional context for AI drafts. The model uses the note to tailor responses but does not expose note text verbatim.

## 7. Confidence Rules

| Decision | Threshold | Notes |
| -------- | --------- | ----- |
| Flag as high-confidence | 0.85 | Advisory in V1 — no auto-send regardless |
| Missing/malformed confidence | — | Route to human review |

## 8. Failure Modes

- **Malformed JSON:** reject, log, route to human review
- **Missing required fields:** treat as invalid, do not use for send decisions
- **Refusal or empty response:** log, leave for human handling
- **Timeout or transport error:** no reply sent; operator can retry later
- **Draft context mismatch at send time:** discard stale draft, require new generation
- **Policy violation:** draft matches prohibited pattern (see §9), persisted as failure

## 9. Draft Safety Policy

`DraftSafetyPolicy` prevents the AI from producing content reserved for operator-controlled workflows:

1. **Prompt-level isolation:** the `ticket_closure` key is stripped from rules before model input, so the model never sees the configured close/lock final message
2. **Output-level guard:** after generation, draft text is compared against the closure message — matches are converted to `POLICY_VIOLATION` failures
3. **Retroactive application:** persisted AI logs are checked when loaded for display, so previously generated policy-violating drafts are also caught

## 10. Eval Coverage

Deterministic tests cover:
- Well-formed JSON output
- Intent classification across the supported label set
- Safe handling of unsupported or hostile messages
- Correct use of local business knowledge
- Rejection of missing fields, extra fields, unsupported labels, invalid confidence
- Optional live OpenAI smoke eval (runs only when credentials and environment flag are set)
