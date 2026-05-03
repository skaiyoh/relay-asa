# Database Schema

> Tables, columns, types, indexes, constraints, relationships, migration history.

## 1. Engine and Version

- **Engine:** SQLite via `org.xerial:sqlite-jdbc:3.45.3.0`
- **Migration tool:** Flyway SQL migrations via `org.flywaydb:flyway-core`
- **Migration naming:** Flyway versioned SQL files use `V###__description.sql`
- Applied migrations are immutable. Schema evolution uses new migration files.

## 2. Tables

### `tickets`

Store the current known state of each upstream support ticket.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key (AUTOINCREMENT) |
| `remote_ticket_id` | INTEGER | No | Unique upstream ticket ID |
| `user_id` | INTEGER | Yes | Upstream customer ID |
| `username` | TEXT | Yes | Upstream customer username |
| `subject` | TEXT | Yes | Ticket subject |
| `status` | TEXT | Yes | Last known upstream status |
| `assignee` | TEXT | Yes | Assigned staff member |
| `is_read` | INTEGER | Yes | Boolean-like flag |
| `created_at` | TEXT | Yes | Upstream timestamp |
| `created_timestamp` | INTEGER | Yes | Parsed timestamp |
| `last_update_at` | TEXT | Yes | Upstream timestamp |
| `last_update_timestamp` | INTEGER | Yes | Parsed timestamp |
| `last_full_sync_at` | TEXT | Yes | Last full detail sync timestamp |
| `needs_human_review` | INTEGER | Yes | Boolean-like flag (default 0) |
| `is_ai_answered` | INTEGER | Yes | Boolean-like flag (default 0) |
| `raw_json` | TEXT | Yes | Raw upstream JSON |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp for retention |

**Indexes:** `remote_ticket_id` (unique), `status`

### `ticket_messages`

Individual messages in ticket conversations.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` |
| `remote_ticket_id` | INTEGER | No | Upstream ticket ID for hashing |
| `message_hash` | TEXT | No | SHA-256 deduplication key (unique) |
| `body` | TEXT | Yes | Message body |
| `sender_name` | TEXT | Yes | Sender display name |
| `is_staff` | INTEGER | Yes | Boolean-like flag |
| `is_unread` | INTEGER | Yes | Boolean-like flag |
| `created_at` | TEXT | Yes | Upstream timestamp |
| `created_timestamp` | INTEGER | Yes | Parsed timestamp |
| `sort_index` | INTEGER | Yes | Ordering aid within a ticket |
| `raw_json` | TEXT | Yes | Raw upstream JSON |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp |

**Indexes:** `ticket_id`, `message_hash` (unique)

### `ticket_attachments`

File attachments on messages.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_message_id` | INTEGER | No | FK → `ticket_messages.id` |
| `name` | TEXT | Yes | Filename |
| `size` | INTEGER | Yes | File size |
| `url` | TEXT | Yes | Attachment URL |
| `raw_json` | TEXT | Yes | Raw upstream JSON |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp |

### `ai_response_logs`

AI-generated draft responses, classifications, and metadata.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` |
| `draft_created_for_message_hash` | TEXT | Yes | Customer-message hash the draft targets |
| `prompt_preview` | TEXT | Yes | Short prompt snapshot |
| `response_text` | TEXT | Yes | Draft reply text |
| `confidence` | REAL | Yes | Model confidence 0.0–1.0 |
| `intent` | TEXT | Yes | Classified intent label |
| `should_reply` | INTEGER | Yes | Advisory recommendation |
| `auto_sent` | INTEGER | Yes | Must remain 0 in V1 |
| `model_name` | TEXT | Yes | Model identifier |
| `created_at` | TEXT | Yes | Local timestamp |
| `raw_json` | TEXT | Yes | Raw model response |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp |

**Indexes:** `ticket_id`

### `sent_reply_logs`

Audit trail of every reply sent from the application.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` |
| `reply_hash` | TEXT | No | Duplicate-protection key (unique) |
| `body` | TEXT | No | Exact reply body sent |
| `sent_at` | TEXT | Yes | Local timestamp |
| `sent_by` | TEXT | Yes | Operator name |
| `ai_response_log_id` | INTEGER | Yes | FK → `ai_response_logs.id` |
| `raw_api_response` | TEXT | Yes | Raw send-reply response |
| `raw_api_response_captured_at` | TEXT | Yes | Capture timestamp |

**Indexes:** `ticket_id`, `reply_hash` (unique)

### `ticket_action_logs`

Audit trail for operator actions (Close & Lock).

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` |
| `remote_ticket_id` | INTEGER | No | Upstream ticket ID |
| `action_type` | TEXT | No | e.g. `LOCAL_CLOSE_LOCK` |
| `final_message_text` | TEXT | Yes | Final message sent |
| `result` | TEXT | Yes | e.g. `success` |
| `warning` | TEXT | Yes | Warning if partial success |
| `error_message` | TEXT | Yes | Error detail |
| `performed_by` | TEXT | Yes | Operator name |
| `performed_at` | TEXT | Yes | ISO timestamp |

### `ticket_local_states`

Local-only workflow status overrides.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` (unique — one per ticket) |
| `remote_ticket_id` | INTEGER | No | Upstream ticket ID |
| `local_status` | TEXT | No | Default `locked` |
| `closed_at` | TEXT | Yes | ISO timestamp |
| `locked_at` | TEXT | Yes | ISO timestamp |
| `performed_by` | TEXT | Yes | Operator name |
| `action_log_id` | INTEGER | Yes | FK → `ticket_action_logs.id` |

### `orders`

Admin API orders that passed the ownership verification gate.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `remote_order_id` | INTEGER | No | Unique upstream order ID |
| `external_id` | TEXT | Yes | External order ID |
| `order_user` | TEXT | Yes | Upstream order user |
| `normalized_order_user` | TEXT | Yes | Trim+lowercase for matching |
| `service_id` | INTEGER | Yes | Maps to `panel_services` |
| `service_name` | TEXT | Yes | Service name |
| `service_type` | TEXT | Yes | Service type |
| `status` | TEXT | Yes | Order status |
| `quantity` | INTEGER | Yes | Order quantity |
| `start_count` | INTEGER | Yes | Start count |
| `remains` | INTEGER | Yes | Remaining count |
| `charge` | TEXT | Yes | Charge as string |
| `link` | TEXT | Yes | Order link URL |
| `provider` | TEXT | Yes | Provider name |
| `raw_json` | TEXT | Yes | Raw API JSON |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp |
| `last_fetched_at` | TEXT | Yes | Last fetch timestamp |

**Indexes:** `remote_order_id` (unique), `normalized_order_user`, `service_id`, `status`

### `ticket_order_links`

Verified relationships between tickets and customer-owned orders.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `ticket_id` | INTEGER | No | FK → `tickets.id` |
| `remote_ticket_id` | INTEGER | No | Upstream ticket ID |
| `order_id` | INTEGER | No | FK → `orders.id` |
| `remote_order_id` | INTEGER | No | Upstream order ID |
| `link_source` | TEXT | No | `extracted`, `manual`, or `batch` |
| `lookup_state` | TEXT | No | Always `VERIFIED` |
| `verification_result` | TEXT | Yes | Verification detail |
| `verified_at` | TEXT | Yes | ISO timestamp |

**Indexes:** `(ticket_id, remote_order_id)` (unique), `ticket_id`, `remote_order_id`

### `order_context_notes` / `ticket_context_notes`

Agent-provided context notes for orders and tickets, used to enrich AI drafts.

One note per order/ticket (500-character limit enforced by UI). Notes flow into AI prompt assembly as additional context but are never echoed verbatim by the model.

### `panel_services`

Service catalog from the Generic Service API.

| Column | Type | Null | Notes |
| ------ | ---- | ---- | ----- |
| `id` | INTEGER | No | Local primary key |
| `service_id` | INTEGER | No | Unique service API ID |
| `name` | TEXT | Yes | Service name |
| `type` | TEXT | Yes | Service type |
| `category` | TEXT | Yes | Category |
| `rate` | TEXT | Yes | Rate as string |
| `min_quantity` | INTEGER | Yes | Min order quantity |
| `max_quantity` | INTEGER | Yes | Max order quantity |
| `refill` | INTEGER | Yes | Boolean-like flag |
| `cancel` | INTEGER | Yes | Boolean-like flag |
| `raw_json` | TEXT | Yes | Raw API JSON |
| `raw_json_captured_at` | TEXT | Yes | Capture timestamp |
| `last_synced_at` | TEXT | No | Last sync timestamp |

**Indexes:** `service_id` (unique), `category`, `type`, `name`

## 3. Relationships

```
tickets ──< ticket_messages ──< ticket_attachments
tickets ──< ai_response_logs
tickets ──< sent_reply_logs ──> ai_response_logs (optional)
tickets ──< ticket_action_logs
tickets ──1 ticket_local_states ──> ticket_action_logs (optional)
tickets ──< ticket_order_links >── orders
tickets ──1 ticket_context_notes
orders  ──1 order_context_notes
orders  ··· panel_services (via service_id)
```

## 4. Derived Identity Rules

### Message deduplication

```
message_hash = SHA-256(remote_ticket_id + created_timestamp + sender_name + is_staff + normalized_body)
```

The upstream API does not provide stable message IDs, so the app derives a stable hash for idempotent upserts.

### Reply duplicate protection

```
reply_hash = SHA-256(remote_ticket_id + latest_customer_message_hash + normalized_reply_body)
```

## 5. Retention Policy

Raw payload columns use two retention windows measured from local capture time:

- **180-day** — tickets, messages, attachments, AI logs, sent reply logs
- **30-day** — orders, panel services

Cleanup sets expired `raw_json` and `*_captured_at` to NULL. Normalized records, audit rows, and verified links survive cleanup. All updates run in a single transaction with rollback on failure.

## 6. Migration History

| Version | Summary |
| ------- | ------- |
| V001 | Initial ticket, message, attachment, AI log, and send log tables |
| V002 | Raw payload capture timestamp columns for retention clock |
| V003 | Close & Lock tables (ticket_action_logs, ticket_local_states) |
| V004 | Panel services table for service catalog |
| V005 | Catalog sync metadata table |
| V006 | Orders and ticket-order links tables |
| V007 | Order link URL column |
| V008 | Order context notes table |
| V009 | Ticket context notes table |
