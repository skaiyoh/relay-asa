# Engineering Process

> Development methodology, documentation workflow, implementation approach, and quality assessment.

## 1. Documentation-Driven Development

Every code change in this project follows a strict docs-first workflow:

1. **Understand** — restate the task and identify affected behavior
2. **Read docs first** — start in `docs/`, not in source files
3. **Determine affected docs** — use the source-of-truth map to find the owning document
4. **Update docs** — before or alongside the code change, never after
5. **Make code changes** — code must match the documentation just written
6. **Update changelog** — add an entry under the current release section
7. **Verify** — run the documentation audit checklist
8. **Report** — end with a structured docs-guardian report

This applies to "quick fixes" too. The documentation surface may shrink, but the audit and report still run.

## 2. Source-of-Truth Map

Every fact lives in **exactly one** canonical document. Cross-link instead of duplicating.

| Topic | Owning document |
| ----- | --------------- |
| High-level system choices | SYSTEM_DESIGN.md |
| Runtime structure, modules, data flow | ARCHITECTURE.md |
| Database tables and migrations | DATABASE_SCHEMA.md |
| External APIs | API_INTEGRATION.md |
| Prompts, models, AI outputs | AI_BEHAVIOR.md |
| Allowed/disallowed automation | AUTOMATION_RULES.md |
| Future work | TODO.md |
| Completed changes | CHANGELOG.md |

**Hard rules:**
- No code behavior change without a doc update
- No undocumented new feature
- No stale or contradictory docs
- No duplicate source of truth

## 3. Implementation Methodology

The entire V1 was built through a **22-step task-runner checklist**, where each step:

1. Starts with reading the relevant documentation
2. Has an explicit scope boundary ("implement only this task")
3. Includes a task-specific checklist of deliverables
4. Requires tests to be added alongside implementation
5. Runs verification (`./gradlew test`) before completion
6. Requires docs and changelog to be current before final handoff
7. Stops for review before moving to the next task

### Milestone progression

| # | Milestone | What it covered |
|---|-----------|----------------|
| 1 | Startup/Config | App bootstrap, env config, database migration, operator flags, knowledge seeding |
| 2 | Persistence | Repository layer with idempotent upserts, hashing and deduplication primitives |
| 3 | API/Sync | Panel API client, ticket mapping, rate limiting, background sync, fresh-fetch gate |
| 4 | UI Foundation | JavaFX shell, navigation, ticket lists, review queue, ticket detail |
| 5 | AI Draft Flow | OpenAI client, prompt assembly, draft generation, UI wiring |
| 6 | Reply Flow | Send service, duplicate protection, stale-context rejection, UI wiring |
| 7 | Admin Screens | Settings, business rules editing, logs viewer, retention cleanup |
| 8 | Hardening | Full regression, distribution build, live smoke test, docs cleanup |

Each milestone was a vertical slice — the app was functional (though incomplete) after each one.

## 4. Testing Strategy

- **93 test files** covering repository, API client, sync, AI validation, reply safety, queue classification, config, and utility behavior
- **MockWebServer** for HTTP client testing — real HTTP calls against a local mock server rather than mocked interfaces
- **Injectable clocks** throughout rate limiter, retention cleanup, and timestamp formatting for deterministic time-dependent testing
- **Deterministic AI evals** — validate output structure, intent classification, hostile message handling, and business knowledge usage without requiring live API calls
- **Optional live smoke eval** — runs against real OpenAI API only when explicitly enabled via environment flag

## 5. Structural Health Assessment

Periodic project reviews grade the codebase across 8 dimensions:

| Area | Assessment | Notes |
|------|-----------|-------|
| Architecture | Strong | Clean layers, sealed types, no circular dependencies |
| Documentation | Excellent | 11 canonical docs with automated sync workflow |
| Test coverage | Good | 93 tests; UI layer coverage lighter than service/repository |
| Safety/Security | Strong | Ownership gate, duplicate detection, kill switches, stale-context rejection |
| Code size discipline | Good | 2 files approaching size limits, identified for decomposition |
| Threading | Adequate | Functional but ad-hoc executor lifecycle in some UI panels |
| UX/Ergonomics | Functional | No shortcuts, notifications, or bulk operations yet |
| CI/Ops | Needs work | No CI pipeline; local Gradle only |

## 6. Design Principles

**Safety over throughput** — V1 optimizes for correctness. Every customer-facing action goes through a human checkpoint. Fresh data is mandatory before decisions. The system fails closed rather than open.

**Conservative allowlists** — ticket statuses, intent labels, and automated actions use explicit allowlists. Unknown values are blocked by default and only added when evidence confirms them.

**Dual storage** — normalized fields for application queries plus raw upstream JSON for recovery, debugging, and auditability. The system can reconstruct state from raw payloads if normalization logic changes.

**Modular boundaries** — even as a desktop app, the architecture maintains clear layer separation (UI → Service → Repository → Database) to enable future migration to server-hosted or multi-user deployment without rewiring business logic.

**Immutable domain** — domain records are immutable data carriers. State changes produce new records rather than mutating existing ones.
