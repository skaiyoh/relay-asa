# Automation Rules

> Allowed and disallowed automated actions, safety boundaries, kill switches, escalation paths.

## 1. Principle

V1 is intentionally conservative. The system automates read-heavy sync work and AI draft preparation, but every customer-facing reply remains human-reviewed and manually sent. Any irreversible action, business action, or uncertain ticket state stops at a human checkpoint.

## 2. Operating Level

| Level | Description | V1? |
| ----- | ----------- | --- |
| 0 | AI drafts only | — |
| 1 | AI drafts plus human approval | **Current** |
| 2 | Auto-reply for safe intents only | Future |
| 3 | AI can trigger safe predefined actions | Future |
| 4 | Fully automated support | Future |

## 3. Allowed Automated Actions

| Action | Reversible? |
| ------ | ----------- |
| Background ticket list sync (60s scheduler) | Yes |
| Queued full ticket fetch (new, changed, or pre-action) | Yes |
| Fresh fetch before view, draft, send, or row refresh | Yes |
| AI draft generation on user request | Yes |
| AI intent classification and confidence scoring | Yes |
| Logging and audit trail writes | No (append-only, internal) |
| Close & Lock with final message (human-initiated with confirmation) | No |

## 4. Disallowed Automated Actions

| Action | Reason |
| ------ | ------ |
| Auto-send AI replies | Customer-facing, must be human-controlled |
| Auto-run refund, cancel, refill, speed-up | Operationally risky, not confirmed safe |
| Browser/provider automation | Future scope only |
| Assume send changed ticket status without post-send fetch | API doesn't confirm final state |
| Act on stale cached ticket state | Fresh upstream state required |
| Run order actions | First slice is read-only assist only |
| Call Service API write endpoints | Outside read-only catalog scope |

## 5. Safety Boundaries

- **Fresh fetch mandatory** before ticket view, AI draft, approval, edit-and-send, manual reply, and Close & Lock
- **Pre-send verification:** ticket not locally locked, latest message unchanged, ticket still actionable, duplicate protection passes
- **Post-send re-fetch:** immediate re-fetch treats API response as source of truth
- **Rate limiting:** 5 requests/second with throttle-back on 429
- **Priority queuing:** user-triggered fetches outrank background work
- **Draft safety policy:** AI cannot produce the configured close/lock message (stripped from prompt input, rejected in output)
- **Order context safety:** only ownership-verified orders reach the AI; blocked/unresolved orders excluded

## 6. Intent-Based Handling

### Future safe auto-reply candidates
`add_funds`, `payment_question`, `api_help`, `greeting`

### Draft only
`order_status`, `order_speedup`, `service_question`

### Human required
`refund_request`, `cancel_request`, `refill_request`, `unknown`, `angry_customer`, `payment_dispute`, `account_balance_issue`, `large_order_complaint`

All categories are advisory in V1 — every send requires a human.

## 7. Kill Switches

### Always locked in V1
- Auto-send is off
- All sends require manual action

### Operator controls

| Flag | Effect when disabled/true |
| ---- | ------------------------ |
| `sync_enabled = false` | Stops polling tickets |
| `ai_drafts_enabled = false` | Blocks AI draft generation |
| `sending_enabled = false` | Disables all send actions |
| `read_only_mode = true` | View only — dominant over all other flags |

Stored in JSON file with environment overrides. Fail-closed parsing: if the file can't be read, defaults to all-disabled + read-only.

## 8. Ticket Status Policy

Conservative allowlist approach — only statuses confirmed from API evidence are classified.

### Replyable (send allowed after human review)
`open`, `pending`

### Completed (send blocked, leaves working set)
`closed`, `answered`, `locked`

### Unknown (send blocked)
Any unrecognized status blocks send and flags for human review. The allowlist expands only with API evidence.

## 9. Escalation

The system hands off to a human when:
- Ticket status is unknown
- AI output is malformed or untrusted
- Intent is human-required
- Confidence is low or missing
- Latest customer message doesn't match draft context
- Duplicate protection can't confirm safety
- Upstream fetch or send calls fail
