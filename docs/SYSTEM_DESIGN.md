# System Design

> High-level system choices, goals, constraints, non-goals, key tradeoffs.

## 1. Purpose

Relay ASA (Automated Support Assistant) is a local JavaFX desktop application for managing SMM support tickets through the Perfect Panel Admin API. It centralizes ticket review, stores complete ticket history locally, generates AI-assisted draft replies, and keeps a human in the loop for every outbound reply in V1.

## 2. Goals

- Centralize support ticket management in one local desktop dashboard.
- Persist full ticket conversation history with high accuracy, including raw upstream JSON for recovery and debugging.
- Generate structured AI draft replies and intent classifications to speed up human review.
- Enforce fresh data checks before important ticket actions so stale local state does not cause bad replies or duplicate actions.
- Keep the V1 system safe enough to grow into more automation later without locking the project into a dead-end architecture.

## 3. Non-goals

- V1 will not auto-send AI replies.
- V1 will not perform provider-side browser automation or order actions.
- V1 will not require cloud hosting or multi-user remote access.
- V1 will not optimize for mobile access.
- V1 will not treat the latest ticket message as sufficient context; full conversation history remains mandatory whenever the API provides it.

## 4. Constraints

- Runtime stack is locked to Java 21 and JavaFX on a local desktop machine.
- Local persistence is SQLite, with schema migrations managed by Flyway.
- External integrations are Perfect Panel Admin API for ticket operations and OpenAI API for AI draft generation.
- HTTP calls use OkHttp and JSON parsing uses Jackson.
- Background scheduling is designed around `ScheduledExecutorService`.
- Perfect Panel calls must respect the documented limit of 5 requests per second per panel.
- Before viewing or acting on a ticket, the app must fetch the latest ticket state and refresh local storage first.
- V1 operates at automation Level 1: AI drafts plus human approval. Human approval is mandatory for every send.

## 5. Key Tradeoffs

| Decision | Chosen | Rejected | Why |
| -------- | ------ | -------- | --- |
| Deployment model | Local JavaFX desktop app | Hosted web app for V1 | Local execution reduces hosting complexity and fits the single-operator workflow while still allowing a later migration path. |
| Reply safety | Human review before every send | Auto-send in V1 | Support replies are customer-facing and potentially irreversible, so V1 optimizes for safety over throughput. |
| Ticket storage | Normalized tables plus raw upstream JSON | Latest-message-only or normalized-only storage | The dashboard and AI need structured fields, but raw JSON is required for auditability, debugging, and recovery. |
| Action freshness | Fresh fetch before view, draft, or send | Acting on cached local state only | Support decisions must be based on the current upstream ticket state to avoid stale context and duplicate actions. |
| Architecture direction | Modular local-first design with future automation hooks | Tightly coupled V1-only implementation | V1 stays local, but the module boundaries should not block later server migration or browser automation. |

## 6. High-Level Shape

The V1 system is a single local desktop process. It synchronizes ticket data from Perfect Panel into SQLite, uses that local store to drive the JavaFX dashboard, and asks OpenAI for structured draft replies only after the latest ticket state has been fetched and stored locally.

The human remains the final gate for all outbound replies. Future automation, including browser-based provider interactions, is intentionally deferred and treated as an extension point rather than a current runtime component.

## 7. Delivery Scope

### V1

- JavaFX desktop dashboard.
- SQLite local database.
- Perfect Panel ticket sync.
- Full ticket detail fetch before view, draft generation, and send.
- Full conversation history storage.
- AI draft generation with structured JSON output.
- Manual approve, edit, and send flow.
- AI response logging and duplicate reply protection.
- Editable local business knowledge.

### Explicitly not in V1

- Automatic AI replies.
- Provider or browser automation.
- Cloud-hosted deployment.
- Mobile access.

### V1.5

- Safe-intent auto-replies.
- Confidence-threshold controls beyond advisory review.
- Improved human review queue.
- More advanced duplicate protection.
- Better order ID extraction.
- More complete business-rule editing.

### V2

- Browser-based automation for provider interactions.
- Provider integrations beyond the panel API.
- Order status checking and refill or speed-up workflows.
- Provider balance monitoring.
- Broader SMM business analytics.
