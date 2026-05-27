# Implementation Plan: Agentic Browser Platform (MVP)

**Branch**: `001-agentic-browser-platform` | **Date**: 2026-05-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-agentic-browser-platform/spec.md`

## Summary

Deliver the MVP slice of the Agentic Browser platform: a modality-neutral
gateway accepts an objective, an LLM planner decomposes it into an auditable task
graph, a location-aware search service produces normalized candidates, isolated
TinyFish browser sandboxes inspect candidates in parallel (discovery/extract
only), and a composer emits a structured **Workspace Document** rendered by
trusted client components. The **safety control plane** is foundational, not
deferred: a deterministic, server-side policy engine + approval gate (denying
all execution-class actions in MVP), idempotent mutating endpoints, and a durable,
resumable, ordered event log. Technical approach favors a workflow *interface*
with a custom Postgres-backed state machine as the MVP engine (Temporal kept
swappable), Postgres as the system-of-record + durable event log, Redis Streams
for live event fan-out, and per-sandbox network egress enforced by a forward
proxy independent of the browser engine.

## Technical Context

**Language/Version**: Python 3.11+ (backend services), TypeScript 5.x / React 18 (frontend), Node.js 20+ (tooling)

**Primary Dependencies**: FastAPI + Uvicorn (async APIs + SSE), SQLAlchemy 2.x + Alembic (persistence/migrations), Pydantic v2 (contracts/validation), Redis (Streams for event fan-out, rate-limit, sandbox dispatch queue), TinyFish browser API SDK (default browser engine adapter), OpenAI SDK (planning/extraction/vision capability adapters), Vite + Tailwind (frontend Workspace Document renderer)

**Storage**: PostgreSQL (Cloud SQL) — system of record for Workspace, Plan, Candidate, BrowserSandbox, PolicyProfile, Approval, Artifact metadata, AuditRecord, and the append-only Event log (monotonic per-workspace sequence). Redis (Memorystore) — live event fan-out + rate limiting + dispatch queue. GCS — artifact blobs (screenshots, recordings) with redaction + retention.

**Testing**: pytest + pytest-asyncio (unit/integration/contract), schemathesis or pytest against the OpenAPI contract (contract tests), Vitest + Playwright (frontend), testcontainers for Postgres/Redis in integration tests

**Target Platform**: Linux server on GCP Cloud Run (gateway + stateless services), containerized browser sandbox workers (Cloud Run jobs / GKE node pool), single primary region for MVP

**Project Type**: Web application — `backend/` (FastAPI services) + `frontend/` (React renderer) + `sandbox-runtime/` (containerized browser worker) + `infra/` (Terraform)

**Performance Goals**: Time-to-ready workspace ≤ 60s P95 for a typical local-search objective (≤10 candidates inspected in parallel); planner latency target P95 [NEEDS CLARIFICATION: planner model latency budget]; sandbox cold-start target [NEEDS CLARIFICATION: acceptable cold-start without pre-warm in MVP]; per-workspace sandbox fan-out cap 10 (policy-configurable)

**Constraints**: No single in-memory coordinator in production (state in Postgres/Redis); idempotency key required on all mutating endpoints; per-workspace network egress allowlist enforced at proxy layer; provider secrets server-side only (GCP Secret Manager); region stored on every record (multi-region additive, not MVP)

**Scale/Scope**: MVP single region; horizontally scalable stateless gateway/services; sandbox worker pool autoscaled by queue depth; design targets ~100 concurrent active workspaces for MVP load testing [NEEDS CLARIFICATION: concrete platform-wide concurrency SLO for MVP]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Plan compliance | Status |
|---|-----------|-----------------|--------|
| I | Human Control over irreversible action | Deterministic `policy/` engine resolves approval requirement from `action_type`/`risk_class`; planner output is advisory only; MVP denies all execution-class sandbox modes | PASS |
| II | Approval binds to a concrete action | `Approval` carries `action_payload_hash` + `state_snapshot`; execution path re-validates live target before acting (contract defined now; execution post-MVP) | PASS |
| III | Exactly-once mutating effects | Idempotency-key middleware + unique constraint on `(tenant_id, idempotency_key)`; effects recorded transactionally | PASS |
| IV | Sandbox-first, least-privilege | Containerized sandboxes; egress allowlist at forward-proxy layer independent of TinyFish; scoped credentials via broker; no payment creds to models; page content untrusted | PASS |
| V | Durable, resumable, auditable state | Postgres append-only `events` (monotonic sequence) + Redis Streams fan-out; SSE resume via `Last-Event-ID`; immutable `AuditRecord` for every action | PASS |
| VI | Provider & engine pluggability | `adapters/` define capability contracts (planning/extraction/vision) + `BrowserEngine` interface (TinyFish default); `WorkflowEngine` interface (no hard Temporal dependency) | PASS |
| VII | Sovereign-by-construction | `region` on every record; provider routing + artifact storage region-pinnable via `PolicyProfile`; no silent cross-region aggregation | PASS (MVP single-region, no violating assumptions) |

**Result (pre-Phase 0)**: No violations. Complexity Tracking not required.

**Post-Phase 1 re-check**: Design artifacts (data-model.md, contracts/openapi.yaml)
preserve all gates — `Idempotency-Key` is required on every mutating endpoint
(III); `Approval` carries `action_payload_hash` + `state_snapshot` (II); the
policy engine resolves `requires_approval` on `ActionDescriptor`, not the planner
(I); execution-class sandbox modes return `403` in MVP (I); the `events` table
provides monotonic sequence + `Last-Event-ID` resume (V); engines/models sit
behind adapters and orchestration behind `WorkflowEngine` (VI); `region` is on
every entity (VII). No new violations introduced.

## Project Structure

### Documentation (this feature)

```text
specs/001-agentic-browser-platform/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (OpenAPI)
│   └── openapi.yaml
└── tasks.md             # Phase 2 output (/speckit.tasks — NOT created here)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── gateway/         # auth, tenant/user/region/policy resolution, workspace lifecycle, SSE endpoint
│   ├── planner/         # NL objective → task graph (planning model adapter), missing-info detection
│   ├── search/          # provider adapters (places/web/browser-fallback), normalization, cross-provider dedup
│   ├── sandbox/         # sandbox allocation/lifecycle, TinyFish adapter, egress-proxy control, artifact capture
│   ├── composer/        # filter + rank + Workspace Document generation
│   ├── policy/          # deterministic policy engine (action_type/risk_class → requires_approval)
│   ├── approval/        # approval lifecycle, payload-hash + snapshot binding, re-validation hook (execution post-MVP)
│   ├── validation/      # freshness/consistency checks + risk-flag detection
│   ├── events/          # append-only event log (sequence allocation), Redis Streams publish, resume reads
│   ├── orchestration/   # WorkflowEngine interface + MVP impl (queue + Postgres state machine)
│   ├── adapters/        # model capability adapters (OpenAI default), BrowserEngine interface (TinyFish)
│   ├── models/          # SQLAlchemy models + Pydantic schemas
│   └── common/          # secrets (GCP SM loader), idempotency middleware, audit writer, errors
└── tests/
    ├── contract/        # OpenAPI conformance
    ├── integration/     # cross-service flows (testcontainers Postgres/Redis)
    └── unit/

frontend/
├── src/
│   ├── components/      # trusted candidate-card + action-descriptor renderers (Aura design)
│   ├── pages/           # workspace view
│   └── services/        # API client, SSE client with Last-Event-ID resume
└── tests/

sandbox-runtime/         # containerized browser worker image + per-workspace egress forward proxy
infra/                   # Terraform: Cloud Run, Cloud SQL, Memorystore, GCS, Secret Manager, network/egress
```

**Structure Decision**: Web-application layout extended with a dedicated
`sandbox-runtime/` (isolated browser workers + egress proxy) and `infra/`
(Terraform). Backend is a modular monolith of FastAPI routers/services that can
be split into independently deployed Cloud Run services later without contract
changes; the gateway, sandbox workers, and orchestration workers are the natural
horizontal-scaling seams.

## Complexity Tracking

> No Constitution Check violations — section intentionally empty.
