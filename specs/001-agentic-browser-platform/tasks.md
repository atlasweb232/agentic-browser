---
description: "Task list for Agentic Browser Platform (MVP)"
---

# Tasks: Agentic Browser Platform (MVP)

**Input**: Design documents from `/specs/001-agentic-browser-platform/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Tests**: INCLUDED — the constitution mandates tests-before-implementation for
safety-critical paths (policy gate, approval binding, idempotency, prompt-injection,
isolation, event resume). Non-safety stories include integration tests for their
acceptance scenarios.

**Organization**: Grouped by user story. MVP delivery = US1 + US2 + US5 (safety
gate) on top of Setup + Foundational. Global scale work is an explicit deferred
workstream (Phase 9) — the MVP only builds the cheap scale-ready *seams* (see R11).

## Format: `[ID] [P?] [Story] Description`

- **[P]** = can run in parallel (different files, no dependency)
- **[Story]** = US1..US6, FND (foundational), SCALE, POLISH

## Path conventions (from plan.md)

`backend/src/{gateway,planner,search,sandbox,composer,policy,approval,validation,events,orchestration,adapters,models,common}`,
`backend/tests/{contract,integration,unit}`, `frontend/src/{components,pages,services}`,
`sandbox-runtime/`, `infra/`.

---

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 Create monorepo structure (`backend/`, `frontend/`, `sandbox-runtime/`, `infra/`) per plan.md
- [ ] T002 [P] Initialize backend Python project (FastAPI, Uvicorn, SQLAlchemy 2.x, Alembic, Pydantic v2, redis, openai, tinyfish SDK) in `backend/`
- [ ] T003 [P] Initialize frontend (React 18 + TS + Vite + Tailwind, Aura design tokens) in `frontend/`
- [ ] T004 [P] Configure lint/format/type (ruff, black, mypy; eslint, prettier) + pre-commit
- [ ] T005 [P] `docker-compose.yml` for Postgres + Redis; testcontainers config for integration tests
- [ ] T006 [P] CI pipeline: `pytest` (unit/integration/contract) + `npm test` + OpenAPI lint

**Checkpoint**: Project builds, lints, and CI runs.

---

## Phase 2: Foundational (Blocking Prerequisites)

**⚠️ CRITICAL**: No user story work begins until this phase is complete.

- [ ] T007 GCP Secret Manager loader `loadAllSecrets()` (gcp-secret refs, no `.env`) in `backend/src/common/secrets.py`
- [ ] T008 Alembic migrations for ALL entities per data-model.md (Workspace, Plan, Candidate, BrowserSandbox, PolicyProfile, Approval, WorkspaceDocument, Event, Artifact, AuditRecord, IdempotencyRecord) incl. `region` on every table, UNIQUE `(tenant_id, idempotency_key)`, UNIQUE `(workspace_id, sequence)`
- [ ] T009 [P] SQLAlchemy models + Pydantic schemas in `backend/src/models/`
- [ ] T010 [P] [FND] Idempotency middleware (require `Idempotency-Key`, store fingerprint+response, replay/409) in `backend/src/common/idempotency.py`
- [ ] T011 [P] [FND] Immutable audit writer in `backend/src/common/audit.py`
- [ ] T012 [FND] Durable event log: append + transactional monotonic per-workspace `sequence` in `backend/src/events/log.py` (depends T008/T009)
- [ ] T013 [FND] Redis Streams publisher + durable-resume reader in `backend/src/events/stream.py` (depends T012)
- [ ] T014 [FND] AuthN/Z + tenancy resolution (OIDC/JWT → principal, resolve `tenant_id`/`user_id`/`region`/`policy_profile_id`) in `backend/src/gateway/auth.py`
- [ ] T015 [FND] API routing + middleware skeleton + structured error handling + rate/concurrency limits in `backend/src/gateway/app.py`
- [ ] T016 [FND] **Deterministic policy engine core** (`action_type`/`risk_class` → `requires_approval`, deny-by-default; planner input ignored) in `backend/src/policy/engine.py`
- [ ] T017 [P] [FND] `Cache` + request-coalescing interface (single-region Redis impl) in `backend/src/common/cache.py` *(scale seam, R11)*
- [ ] T018 [P] [FND] Model capability adapter interfaces (`PlanningModel`/`ExtractionModel`/`VisionModel`) + OpenAI default + tier-routing hook in `backend/src/adapters/models/` *(scale seam, R11)*
- [ ] T019 [P] [FND] `BrowserEngine` interface + `BrowserEnginePool`/pre-warm interface (on-demand impl) in `backend/src/adapters/browser/` *(scale seam, R11)*
- [ ] T020 [P] [FND] `WorkflowEngine` interface + Postgres state-machine + Redis-queue impl in `backend/src/orchestration/`

**Checkpoint**: Foundation ready — durable events, idempotency, auth, policy core, and scale-ready interfaces exist. User stories can begin.

---

## Phase 3: User Story 1 - Create workspace & auditable plan (Priority: P1) 🎯 MVP

**Goal**: Modality-neutral intake → persisted workspace → auditable plan; same contract across modalities.

**Independent Test**: POST objective → workspace `planning`, `plan.created` event with constraints + parallel subtask graph; duplicate idempotency key → same workspace.

### Tests (write first, must fail)
- [ ] T021 [P] [US1] Contract test `POST /workspaces` (201 + idempotent replay + 409) in `backend/tests/contract/test_create_workspace.py`
- [ ] T022 [P] [US1] Integration test: objective → `plan.created` with extracted constraints; missing-info → `needs_user_info` + `question.required` in `backend/tests/integration/test_planning.py`

### Implementation
- [ ] T023 [US1] Planner service: NL objective → task graph + constraint extraction + parallel flags via `PlanningModel` in `backend/src/planner/service.py`
- [ ] T024 [US1] Missing-information detection → `needs_user_info` transition in `backend/src/planner/missing_info.py`
- [ ] T025 [US1] `POST /workspaces` endpoint (idempotent, lifecycle status, emits `plan.created`) in `backend/src/gateway/workspaces.py`
- [ ] T026 [US1] `POST /workspaces/{id}/answers` (resume planning) + `GET /workspaces/{id}` in `backend/src/gateway/workspaces.py`
- [ ] T027 [US1] Audit + event wiring for planning operations

**Checkpoint**: A workspace + auditable plan can be created from any modality and streamed.

---

## Phase 4: User Story 2 - Location-aware search → normalized candidates (Priority: P1)

**Goal**: Structured, location-aware search across adapters → normalized, deduped, provenance-tagged candidates. **Structured-first** (browser is last resort).

**Independent Test**: plan with `location_search` → ≥1 candidate within radius, provenance present; same place from 2 providers → 1 candidate.

### Tests (write first)
- [ ] T028 [P] [US2] Integration test: candidates within radius + non-empty provenance; cross-provider dedup; provider fallback in `backend/tests/integration/test_search.py`

### Implementation
- [ ] T029 [P] [US2] Provider adapters: Places-class, web-search, browser-fallback behind capability contract in `backend/src/search/providers/`
- [ ] T030 [US2] Candidate normalization → common shape + confidence in `backend/src/search/normalize.py`
- [ ] T031 [US2] Cross-provider dedup (`dedup_key`, union provenance) in `backend/src/search/dedup.py`
- [ ] T032 [US2] Search orchestration: structured-first ordering, cache + coalescing usage (T017), `candidate.found` events in `backend/src/search/service.py`
- [ ] T033 [US2] Radius/category/open-hours/rating/price/availability/source filters in `backend/src/search/filters.py`

**Checkpoint**: Planned objective yields normalized, deduped candidates with provenance.

---

## Phase 5: User Story 5 - Policy gate & approval binding (Priority: P1, SAFETY-CRITICAL)

**Goal**: Structurally prevent any irreversible action without a deterministic policy decision + valid bound approval. Gate exists in MVP; execution is denied/out-of-scope.

**Independent Test**: risky action w/o approval → denied (even if planner said false); snapshot divergence → halt/re-prompt; double execute (same key) → single effect; execution-class sandbox mode → 403.

### Tests (write first — SAFETY)
- [ ] T034 [P] [US5] Policy-gate test: risky `action_type` w/o approval → denied; planner `requires_approval:false` ignored in `backend/tests/integration/test_policy_gate.py`
- [ ] T035 [P] [US5] Approval-binding test: live price/terms diverge from `state_snapshot` → halt + re-approve in `backend/tests/integration/test_approval_binding.py`
- [ ] T036 [P] [US5] Idempotency test: approved action executed twice (same key) → one effect in `backend/tests/integration/test_exec_idempotency.py`
- [ ] T037 [P] [US5] Execution-class sandbox mode (`submit_approved` etc.) → 403 in `backend/tests/contract/test_sandbox_mode_denied.py`
- [ ] T038 [P] [US5] Prompt-injection test: page instruction to auto-submit does not alter policy/approval in `backend/tests/integration/test_prompt_injection.py`

### Implementation
- [ ] T039 [US5] Approval lifecycle (`pending→approved/denied/expired/voided`) + `expires_at` in `backend/src/approval/service.py`
- [ ] T040 [US5] Approval binding: `action_payload_hash` + `state_snapshot` capture + live re-validation hook in `backend/src/approval/binding.py`
- [ ] T041 [US5] `POST /approvals/{id}/approve` + `/deny` endpoints in `backend/src/gateway/approvals.py`
- [ ] T042 [US5] Enforce execution-class sandbox-mode denial in allocation; `POST /sandboxes/{id}/actions` returns 403 in MVP (FR-026) in `backend/src/sandbox/policy_guard.py`
- [ ] T043 [US5] Audit every policy decision + approval/denial (`policy.deny`, `approval.grant`)

**Checkpoint**: No execution path can bypass the gate; safety suite green.

---

## Phase 6: User Story 3 - Parallel browser sandbox inspection (Priority: P2)

**Goal**: Isolated TinyFish sandboxes inspect candidates concurrently (discovery/extract), egress enforced at proxy, auto-expire, fail-isolated.

**Independent Test**: N sandboxes concurrent; off-allowlist navigation blocked at proxy; one crash ≠ workspace failure.

### Tests (write first)
- [ ] T044 [P] [US3] Integration test: N concurrent sandboxes up to cap; crash isolation in `backend/tests/integration/test_sandbox_fanout.py`
- [ ] T045 [P] [US3] Egress test: off-allowlist navigation blocked at proxy + recorded as denial in `backend/tests/integration/test_egress_allowlist.py`

### Implementation
- [ ] T046 [US3] Sandbox allocation/lifecycle via queue dispatch + auto-expire/destroy in `backend/src/sandbox/service.py`
- [ ] T047 [US3] TinyFish `BrowserEngine` adapter (discovery/extract) in `backend/src/adapters/browser/tinyfish.py`
- [ ] T048 [US3] `sandbox-runtime/` container image + **per-workspace egress forward proxy** (allowlist enforced at network layer) in `sandbox-runtime/`
- [ ] T049 [US3] Artifact capture (screenshots/extracted data) → GCS + `Artifact` records + redaction hook in `backend/src/sandbox/artifacts.py`
- [ ] T050 [US3] Sandbox events (`sandbox.started`/`sandbox.step`/`artifact.created`) + extracted data → candidate
- [ ] T051 [US3] Per-workspace fan-out cap (default 10, policy-configurable) + backpressure in `backend/src/sandbox/dispatch.py`
- [ ] T052 [US3] `POST /workspaces/{id}/sandboxes` endpoint (discovery/extract only) in `backend/src/gateway/sandboxes.py`

**Checkpoint**: Candidates enriched by parallel, isolated, egress-controlled inspection.

---

## Phase 7: User Story 4 - Composable ranked workspace + gated actions (Priority: P2)

**Goal**: Filter/rank candidates, emit a Workspace Document of typed cards + policy-gated action descriptors; trusted client renders it; link-out + thumbnails default.

**Independent Test**: enriched candidates + criteria → `workspace.composed`, ranked cards with provenance/artifacts, risky actions rendered approval-gated.

### Tests (write first)
- [ ] T053 [P] [US4] Integration test: filter+rank order (constraint→distance→rating→price); provenance cited; risky actions gated in `backend/tests/integration/test_composer.py`
- [ ] T054 [P] [US4] Contract test `GET /workspaces/{id}/document` schema (no raw HTML; gated descriptors) in `backend/tests/contract/test_workspace_document.py`

### Implementation
- [ ] T055 [P] [US4] Deterministic ranker (constraint filter → weighted distance>rating>price) in `backend/src/composer/rank.py`
- [ ] T056 [P] [US4] Validation service (freshness/consistency + risk flags) in `backend/src/validation/service.py`
- [ ] T057 [US4] Workspace Document generation (typed cards + action descriptors via policy engine T016; link-out/thumbnails) in `backend/src/composer/document.py`
- [ ] T058 [US4] `GET /workspaces/{id}/document` endpoint + `workspace.composed` event in `backend/src/gateway/workspaces.py`
- [ ] T059 [P] [US4] Frontend: trusted candidate-card + action-descriptor renderers (Aura) in `frontend/src/components/`
- [ ] T060 [P] [US4] Frontend: workspace page + API client in `frontend/src/pages/` and `frontend/src/services/`

**Checkpoint**: A user reaches a ranked, provenance-cited workspace with gated actions.

---

## Phase 8: User Story 6 - Reconnect & resume event stream (Priority: P2)

**Goal**: SSE stream resumes from `Last-Event-ID` with no loss/duplication; survives instance restart.

**Independent Test**: receive to seq N, disconnect, reconnect with `Last-Event-ID=N` → N+1.. in order, no gaps/repeats.

### Tests (write first)
- [ ] T061 [P] [US6] Integration test: reconnect resume (no loss/dupe) + serving-instance restart in `backend/tests/integration/test_event_resume.py`

### Implementation
- [ ] T062 [US6] `GET /workspaces/{id}/events` SSE endpoint with `Last-Event-ID` gap-fill from Postgres then attach to live stream in `backend/src/gateway/events.py` (uses T012/T013)
- [ ] T063 [P] [US6] Frontend SSE client with auto-reconnect + cursor in `frontend/src/services/eventStream.ts`

**Checkpoint**: MVP complete — full happy path streamable and resumable, safety gate enforced.

---

## Phase 9: Global Scale & Latency (Scale-out — DEFERRED, planned) 🌍

**Purpose**: Serve millions globally at low perceived latency / least overhead (R11). NOT required for MVP demo; build on the seams from Phase 2. Each item is independently shippable.

- [ ] T064 [SCALE] Request coalescing: share one search/inspect pass across identical in-flight objectives (concrete impl over T017) in `backend/src/common/coalesce.py`
- [ ] T065 [SCALE] Public-layer result/document cache: key `(normalized_objective+geohash+constraints)`, freshness TTL, strict per-tenant privacy scoping in `backend/src/common/cache_policy.py`
- [ ] T066 [SCALE] Model tier-routing + prompt caching (cheap extraction / premium planning) over T018
- [ ] T067 [SCALE] Per-region pre-warmed sandbox pool (autoscale by queue depth) over T019 in `backend/src/sandbox/pool.py` + `sandbox-runtime/`
- [ ] T068 [SCALE] Per-workspace budget caps + cost governance control plane in `backend/src/common/budget.py`
- [ ] T069 [SCALE] Regional cell topology (self-contained gateway+Postgres+Redis+pool+routing per region) in `infra/cells/` (Terraform)
- [ ] T070 [SCALE] GeoDNS/anycast edge routing to nearest cell + stateless edge gateway in `infra/edge/`
- [ ] T071 [SCALE] CDN for artifacts (thumbnails/screenshots) in `infra/cdn/`
- [ ] T072 [SCALE] Cross-region metadata policy (residency-aware) + tenant migration controls
- [ ] T073 [SCALE] Load/spike tests: MVP ~100 concurrent workspaces, then per-cell scale + backpressure validation in `backend/tests/load/`

---

## Phase 10: Polish & Cross-Cutting

- [ ] T074 [P] [POLISH] Observability: metrics from spec §16 (planning/search/sandbox-start latency, queue depth, cost/token, policy denials, intervention rate) + tracing
- [ ] T075 [P] [POLISH] Security hardening: download-block/scan, upload-approval, screenshot/page-text redaction per policy
- [ ] T076 [POLISH] Run `quickstart.md` end-to-end validation + acceptance→SC mapping
- [ ] T077 [P] [POLISH] Unit tests for normalization/dedup/ranking/policy edge cases in `backend/tests/unit/`

---

## Dependencies & Execution Order

- **Setup (P1)** → no deps.
- **Foundational (P2)** → depends on Setup; BLOCKS all stories. T016 (policy core), T012/T013 (events), T010 (idempotency) are hard prerequisites.
- **US1, US2, US5 (all P1)** → MVP-critical; start after Foundational. US5's gate must be green before US3/US4 expose actions.
- **US3, US4, US6 (P2)** → after Foundational; US4 depends on US2 (candidates) + US3 (enrichment) + US5 (gating); US6 depends on T012/T013.
- **Scale-out (Phase 9)** → after MVP stories; builds on Phase 2 seams; independently shippable.
- **Polish (Phase 10)** → after desired stories.

### Within each story
Tests (safety/integration) written first and failing → models → services → endpoints → integration → frontend.

### Parallel opportunities
- Setup T002–T006 parallel; Foundational T009/T010/T011/T017/T018/T019/T020 parallel.
- After Foundational: US1, US2, US5 can be staffed in parallel; US3 parallel once US5 gate exists.
- All `[P]` tests within a story run in parallel.

---

## Implementation Strategy

### MVP (safety-complete) = Setup + Foundational + US1 + US2 + US5
Deliver a modality-neutral intake → auditable plan → searched candidates, with the
deterministic policy gate + approval binding + idempotency enforced and execution
denied. **STOP and validate the safety suite (T034–T038) before adding US3/US4.**

### Then incremental: US3 → US4 → US6
Adds parallel inspection, composable workspace, and resumable streaming — each
independently testable.

### Scale-out (Phase 9) only after MVP validated
Turn on coalescing/caching/model-tiering first (largest overhead wins), then
pre-warm pools, then regional cells + edge routing + CDN.

## Notes
- `[P]` = different files, no dependency. `[Story]` maps to spec user stories for traceability.
- Commit after each task or logical group. Verify safety tests fail before implementing.
- No implementation task starts before its spec + plan pass the Constitution Check (T016/T010/T012 enforce Principles I/III/V).
