# Phase 0 Research: Agentic Browser Platform (MVP)

All `NEEDS CLARIFICATION` items from the plan's Technical Context and the spec's
deferred Open Questions are resolved below.

## R1. Workflow orchestration engine

- **Decision**: Define a `WorkflowEngine` interface; the MVP implementation is a
  custom **Redis-queue dispatch + PostgreSQL state machine** (durable workspace
  state, human-wait states modeled as persisted statuses, fan-out via queue,
  retries via worker re-claim). Temporal remains swappable behind the interface.
- **Rationale**: Temporal's operational overhead (cluster, multi-region, replay
  privacy review for data residency) is not justified for an MVP, but the spec
  requires durable state, retries, human-wait, and fan-out today. A Postgres
  state machine + queue delivers those with infrastructure we already run, while
  the interface preserves the option to adopt Temporal without contract changes.
- **Alternatives considered**: Temporal (rejected for MVP: ops complexity, data-
  residency replay concerns); LangGraph persistence (rejected: less mature for
  multi-service durable orchestration + human approvals at our scale); cloud
  durable functions (rejected: weaker portability across sovereign regions).

## R2. Durable, resumable event log

- **Decision**: PostgreSQL append-only `events` table is the system of record,
  with a **monotonic per-workspace `sequence`** allocated transactionally. Redis
  Streams carries live fan-out to connected SSE servers. The SSE endpoint
  resumes from a client `Last-Event-ID` by reading Postgres for any gap, then
  attaching to the live stream.
- **Rationale**: Satisfies FR-028/FR-029 (ordered, persisted, resumable, no
  single in-memory coordinator). Postgres guarantees ordering + durability +
  replay; Redis Streams gives low-latency fan-out without being the source of
  truth.
- **Alternatives considered**: Kafka/NATS as source of truth (rejected for MVP:
  heavier ops, and we still need a queryable store for audit/resume); in-memory
  pub/sub only (rejected: violates durability + resume).

## R3. Idempotency / exactly-once effects

- **Decision**: An idempotency middleware requires an `Idempotency-Key` header on
  all mutating endpoints. Store `(tenant_id, idempotency_key) → request
  fingerprint + stored response` with a UNIQUE constraint; replays return the
  stored response, conflicting fingerprints return `409`. Effect writes occur in
  the same transaction that records the key.
- **Rationale**: Satisfies FR-003 and constitution Principle III; the unique
  constraint makes double-submit/double-charge structurally impossible even under
  concurrent retries.
- **Alternatives considered**: Dedup by natural object identity (rejected:
  doesn't cover non-idempotent actions like execute/pay); client-only retry IDs
  (rejected: not enforceable server-side).

## R4. Browser engine adapter + egress enforcement

- **Decision**: `BrowserEngine` interface with the **TinyFish browser API** as
  the MVP default adapter (navigation, discovery, extraction). Each sandbox runs
  in an isolated container whose **network egress is forced through a
  per-workspace forward proxy** that enforces the allowlist/denylist; the proxy,
  not the engine, is the control point. Other engines (Playwright, browser-use,
  OpenAI computer use) implement the same interface.
- **Rationale**: Satisfies FR-013/FR-014/FR-018. Enforcing egress at the network
  layer means an LLM-driven or third-party engine cannot navigate off-allowlist
  regardless of its instructions — the single most important sandbox control.
- **Alternatives considered**: Enforcing allowlist by instructing the engine
  (rejected: not a security boundary); shared browser pool (rejected: violates
  per-workspace isolation + scoped credentials).

## R5. Sandbox isolation & cold start

- **Decision**: One container per sandbox (gVisor-isolated where available),
  dispatched from the Redis queue, auto-destroyed on terminal state or timeout.
  **No pre-warming in MVP** — cold start is budgeted within the 60s P95
  end-to-end target; pre-warm pools are a Milestone-6 optimization.
- **Rationale**: Keeps MVP infra simple while honoring isolation (FR-016) and the
  latency target; queue-based dispatch gives backpressure (FR-004).
- **Alternatives considered**: Warm pool now (rejected: premature for MVP cost);
  in-process headless browsers (rejected: no isolation/egress boundary).
- **Latency budget (within 60s P95)**: planner ≤ 8s; search ≤ 6s; sandbox
  cold-start + inspect (parallel, ≤10) ≤ 35s; compose ≤ 5s; overhead ≤ 6s.

## R6. Modality-capability matrix

- **Decision**:

  | Modality | Initiate | Answer questions | Render Workspace Document | Confirm approval |
  |----------|----------|------------------|---------------------------|------------------|
  | Web / Chat (rich) | Yes | Yes | Yes (full) | Yes |
  | Mobile | Yes | Yes | Yes (full) | Yes |
  | Voice / Telephony | Yes | Yes (spoken) | No (spoken summary only) | Yes (explicit spoken confirmation) |
  | Direct API | Yes | Yes | Consumer-rendered | Yes |

- **Rationale**: Resolves spec Open Question on modalities. The modality-neutral
  contract holds for intake/answers/approval; only *visual rendering* of the
  Workspace Document is modality-dependent. Voice participates via spoken
  summaries + confirmations, never visual.
- **Alternatives considered**: Forcing visual parity across modalities (rejected:
  meaningless for voice/telephony).

## R7. Content-preservation policy (composer)

- **Decision**: Default to **link-out + policy-governed thumbnails/snippets**;
  never rehost source pages as raw/executable HTML. Per-source preservation
  rules live in `PolicyProfile.content_preservation_policy`. Full-content
  preservation requires explicit per-source allowance and legal sign-off.
- **Rationale**: Resolves the legal/copyright + prompt-injection surface flagged
  in review; aligns with FR-019/FR-020. Legal review tracked as an external
  dependency, not a code blocker for MVP.
- **Alternatives considered**: Full visual clone of source pages (rejected:
  copyright/ToS/trademark exposure + injection risk).

## R8. Model provider capability adapters

- **Decision**: Capability contracts — `PlanningModel`, `ExtractionModel`,
  `VisionModel` — with an OpenAI-class default implementation; concrete model is
  config + policy-routed per region. "GPT-5.5-class"/"Realtime-2-class" treated
  as capability tiers, not SKUs. Voice model is post-MVP.
- **Rationale**: Satisfies FR-032 and Principle VI; lets sovereign regions swap
  to a local model without code changes.
- **Alternatives considered**: Hard-coding a single OpenAI model (rejected:
  blocks sovereign routing + future model bumps).

## R9. Ranking implementation

- **Decision**: **Deterministic** ranker — apply hard constraints (must-haves,
  budget) as a filter, then weighted sort `distance > rating > price`; weights
  configurable per `PolicyProfile`. Ranking is NOT LLM-driven.
- **Rationale**: Determinism makes FR-020 testable (SC-002) and ranking
  explainable; avoids non-reproducible LLM ordering.
- **Alternatives considered**: LLM re-ranking (rejected for MVP: non-deterministic,
  harder to test/audit; can be added as an optional signal later).

## R10. AuthN/Z & tenancy resolution

- **Decision**: Clients and services authenticate via OIDC/JWT bearer tokens
  validated at the gateway; the gateway resolves `tenant_id`, `user_id`,
  `region`, and `policy_profile_id` and injects a request-scoped principal.
  Entitlements/rate limits enforced at the gateway.
- **Rationale**: Satisfies FR-001/FR-004 with a standard, portable mechanism.
- **Alternatives considered**: mTLS-only service mesh (rejected for MVP: doesn't
  cover end-user identity); API keys per client (rejected: weak tenancy/identity).

## R11. Global scale & latency strategy (millions of users)

- **Decision**: Adopt a **"scale-ready seams in MVP, scale-out infra deferred"**
  posture. The MVP is single-region but includes the cheap-to-add seams so
  global scale is later config/infra, not a rewrite. The target end-state is a
  **regional cell architecture** with edge routing.
  - **MVP seams (build now)**: (a) `Cache` + **request-coalescing** interface so
    identical concurrent objectives share one search/inspect pass and public,
    non-personalized layers are cached by `(normalized_objective + geohash +
    constraints)` with freshness TTL + per-tenant privacy scoping; (b)
    **structured-first** search that prefers Places/web APIs and only launches a
    browser sandbox as a last resort; (c) model **tier-routing** adapters (cheap
    extraction model, premium planning) + prompt caching; (d) per-workspace
    **budget + sandbox fan-out caps** (cost/capacity governance); (e)
    **BrowserEnginePool / pre-warm interface** (on-demand in MVP); (f) stateless
    gateway + broker-based (Redis Streams) event fan-out; (g) `region` on every
    record (already in data model).
  - **Scale-out (deferred, planned workstream)**: regional **cells** (each a
    self-contained gateway + Postgres + Redis + sandbox pool + model routing);
    **GeoDNS/anycast** edge routing to nearest cell; per-region **pre-warmed
    sandbox pools** autoscaled by queue depth; **CDN** for artifacts; cross-region
    metadata only where policy permits.
- **Rationale**: This workload is long-running and expensive per request
  (sandboxes + LLM), not high-QPS CRUD — so the latency/overhead levers are
  *perceived-latency streaming*, *avoiding browser work via caching/coalescing
  and structured-first*, and *proximity via regional cells*, NOT raw request
  throughput. Cells scale linearly and satisfy data residency (Principle VII).
- **Alternatives considered**: Single global multi-master DB + global LB
  (rejected: latency, residency violations, hot-shard risk); building full
  multi-region now (rejected: premature, contradicts MVP-first, large overhead);
  ignoring scale and refactoring later (rejected: the seams above are cheap now
  and expensive to retrofit — e.g., coalescing and structured-first shape the
  service interfaces).

## Resolved unknowns summary

| Source marker | Resolution |
|---------------|-----------|
| Planner latency budget | ≤ 8s P95 (R5) |
| Sandbox cold-start budget | within 60s end-to-end; no pre-warm in MVP (R5) |
| Platform-wide concurrency SLO | MVP load-test target ~100 concurrent active workspaces (R5/R1) |
| Orchestration engine | WorkflowEngine interface + Postgres state machine (R1) |
| Modality matrix | Defined (R6) |
| Content-preservation policy | Link-out + thumbnails default (R7) |
