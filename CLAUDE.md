# CLAUDE.md — Agentic Browser

Standalone, globally deployable infrastructure that turns requests from any
modality into controlled, auditable browser-based task workspaces. Clients
(desktop/mobile/web/voice/telephony) consume this platform via APIs + events;
they are NOT built here.

## Active feature

`specs/001-agentic-browser-platform/` — MVP platform. Lifecycle artifacts:
`spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/openapi.yaml`,
`quickstart.md`.

## Non-negotiable principles (see `.specify/memory/constitution.md`)

1. **Human control** — irreversible actions (pay/book/submit/call/message/upload/
   publish/share PII) require an explicit recorded approval. The requirement is
   set by a **deterministic server-side policy engine** keyed on `action_type`/
   `risk_class` — never by model output or page content.
2. **Approval binds to the action** — approvals carry `action_payload_hash` +
   `state_snapshot`; execution re-validates the live target and halts on divergence.
3. **Exactly-once** — every mutating endpoint requires an `Idempotency-Key`.
4. **Sandbox-first / least-privilege** — browser work runs in isolated sandboxes;
   egress allowlist enforced at the **network/proxy layer**, not by the engine;
   page content is untrusted; payment creds never reach models.
5. **Durable, resumable, auditable** — ordered persisted event log (monotonic
   per-workspace sequence), SSE resume via `Last-Event-ID`, immutable audit records.
6. **Pluggability** — models and browser engines behind capability-contract
   adapters; orchestration behind a `WorkflowEngine` interface.
7. **Sovereign-by-construction** — `region` on every record; region-pinnable
   processing/storage/routing; no silent cross-region aggregation.

## Stack

- Backend: Python 3.11+, FastAPI, SQLAlchemy 2.x + Alembic, Pydantic v2
- Data: PostgreSQL (system of record + durable event log), Redis (event fan-out
  via Streams, rate-limit, sandbox dispatch queue), GCS (artifacts)
- Browser engine: **TinyFish browser API** (default, behind `BrowserEngine` interface)
- Models: OpenAI-class via capability adapters (planning/extraction/vision)
- Frontend: React 18 + TypeScript + Tailwind + Vite (renders the Workspace Document)
- Sandbox runtime: containerized browser worker + per-workspace egress forward proxy
- Infra: GCP Cloud Run + Cloud SQL + Memorystore + GCS + Secret Manager; Terraform

## Secrets

GCP Secret Manager only (project `solid-topic-466217-t9`). Reference as
`gcp-secret:<name>`; load via `loadAllSecrets()` at startup. Never hardcode keys
or place them in `.env`.

## MVP boundaries

In: gateway/intake, planner, location search, discovery/extract sandboxes,
composer + Workspace Document, policy engine + approval gate (denying
execution-class actions), durable resumable events, idempotency, audit.
Out: execution of pay/book/submit/communicate, multi-region/autoscaling/pre-warm,
committing to Temporal, full source-content rehosting.

## Commands

```bash
cd backend && pytest tests/unit tests/integration tests/contract
cd frontend && npm run test && npm run lint
```

## Speckit lifecycle

specify → clarify → plan → tasks → implement. Scaffolding in `.specify/`
(scripts, templates, constitution). Do not begin an implementation task before
its spec + plan pass the Constitution Check.
