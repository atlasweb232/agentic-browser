# Phase 1 Data Model: Agentic Browser Platform (MVP)

Persistence: PostgreSQL system of record. All IDs are opaque strings (prefixed,
e.g. `abw_`, `cand_`). All tables carry `region` and timestamps. Money is stored
as integer minor units + currency code. JSON columns use `jsonb`.

## Entity: Workspace

The root task session.

| Field | Type | Notes |
|-------|------|-------|
| `workspace_id` | string (PK) | `abw_…` |
| `tenant_id` | string | indexed |
| `user_id` | string | indexed |
| `region` | string | region-pinned processing/storage |
| `objective` | text | original NL request |
| `modality` | enum | `web`,`chat`,`mobile`,`voice`,`telephony`,`api` |
| `status` | enum | see state machine below |
| `policy_profile_id` | string (FK) | → PolicyProfile |
| `location_context` | jsonb | lat/lng/radius/permission_id (nullable) |
| `idempotency_key` | string | unique with `tenant_id` |
| `version` | int | optimistic concurrency |
| `created_at`/`updated_at`/`expires_at` | timestamptz | |

**Validation**: `objective` non-empty; `(tenant_id, idempotency_key)` UNIQUE;
`expires_at > created_at`.

**State transitions** (MVP):
`created → planning → needs_user_info → searching → inspecting → composing → ready_for_review`
with `→ cancelled / failed / expired` reachable from any non-terminal state.
Reserved (post-MVP execution): `ready_for_review → awaiting_approval → executing_action → completed`.

## Entity: Plan

Auditable decomposition produced by the planner.

| Field | Type | Notes |
|-------|------|-------|
| `plan_id` | string (PK) | |
| `workspace_id` | string (FK) | |
| `objective` | text | normalized |
| `constraints` | jsonb | radius, date, budget, category, must_have, exclusions |
| `subtasks` | jsonb | `[{type, parallel, tool, mode, risk_hint(advisory)}]` |
| `missing_information` | jsonb | list of required-but-absent fields |
| `risk_summary` | jsonb | advisory only — never gates approval |
| `created_by_model` / `model_version` | string | provenance |
| `created_at` | timestamptz | |

**Validation**: `risk_hint`/`risk_summary` are advisory and MUST NOT be read by
the policy engine.

## Entity: Candidate

A normalized result; the unit later stages operate on.

| Field | Type | Notes |
|-------|------|-------|
| `candidate_id` | string (PK) | |
| `workspace_id` | string (FK) | |
| `dedup_key` | string | cross-provider real-world identity; indexed |
| `name` / `category` | string | |
| `source_url` / `source_provider` | string | |
| `location` | jsonb | lat/lng/address |
| `distance` | numeric | meters from query origin |
| `rating` | numeric | nullable |
| `price_summary` | jsonb | nullable |
| `availability` | jsonb | nullable |
| `images` / `icons` | jsonb | URIs + license/preservation flags |
| `action_options` | jsonb | candidate-level available actions |
| `risk_flags` | jsonb | from validation service |
| `normalized_data` | jsonb | engine-extracted structured data |
| `source_provenance` | jsonb | multi-source: `[{provider, url, fetched_at, confidence}]` |
| `confidence` | numeric | 0–1 |
| `rank_score` | numeric | composer output |

**Validation**: candidates sharing `dedup_key` within a workspace MUST be merged
(provenance unioned); `distance` ≤ requested radius for inclusion.

## Entity: BrowserSandbox

Isolated browser session inspecting a candidate.

| Field | Type | Notes |
|-------|------|-------|
| `sandbox_id` | string (PK) | |
| `workspace_id` / `candidate_id` | string (FK) | |
| `engine` | enum | `tinyfish`(default),`playwright`,`browser_use`,`computer_use` |
| `mode` | enum | MVP allows `discovery`,`extract`; `prepare`/`submit_approved`/`payment_approved`/`communication_approved` DENIED |
| `status` | enum | see state machine |
| `allowed_domains` / `blocked_domains` | jsonb | enforced at proxy |
| `browser_endpoint` | string | internal, scoped |
| `recording_artifact_id` | string (FK) | nullable |
| `created_at` / `destroyed_at` | timestamptz | |

**State transitions**:
`requested → allocated → starting → ready → running → completed`
with `→ awaiting_approval` (post-MVP), `→ failed / expired → destroyed`.

**Validation**: allocation in any execution-class `mode` is rejected by the
policy engine in MVP (FR-026); single sandbox failure MUST NOT fail the workspace.

## Entity: PolicyProfile

Tenant/region policy resolved at the gateway.

| Field | Type | Notes |
|-------|------|-------|
| `policy_profile_id` | string (PK) | |
| `region_pinning` | jsonb | processing/storage/routing regions |
| `action_risk_classes` | jsonb | `action_type → risk_class → requires_approval` |
| `default_allowlist` / `default_denylist` | jsonb | sandbox egress defaults |
| `content_preservation_policy` | jsonb | per-source render rules (default link-out) |
| `provider_routing` | jsonb | model/search provider routing |
| `automation_allowances` | jsonb | tenant-permitted auto actions (none risky in MVP) |
| `sandbox_fanout_cap` | int | default 10 |
| `ranking_weights` | jsonb | distance/rating/price weights |

## Entity: Approval

Bound authorization for one concrete action (contract present in MVP; execution post-MVP).

| Field | Type | Notes |
|-------|------|-------|
| `approval_id` | string (PK) | |
| `workspace_id` / `sandbox_id` | string (FK) | |
| `risk_class` | enum | resolved by policy engine |
| `action_type` | enum | `purchase`,`payment`,`booking`,`application`,`submission`,`account_creation`,`call`,`message`,`upload`,`terms_acceptance`,`publishing`,`personal_data_share` |
| `summary` | text | plain-language |
| `action_payload_hash` | string | binds approval to exact action |
| `state_snapshot` | jsonb | price/terms/destination/data-to-share at approval time |
| `data_to_share` | jsonb | |
| `cost_amount` | jsonb | minor units + currency (nullable) |
| `payment_required` | bool | |
| `expires_at` | timestamptz | approvals expire |
| `status` | enum | `pending → approved / denied / expired / voided` |
| `approved_by` / `approved_at` | string / timestamptz | actor + time |

**Validation**: execution MUST verify live target against `state_snapshot`
(material divergence → halt + re-prompt); expired/voided approvals never authorize.

## Entity: WorkspaceDocument

The composed, renderable workspace model (no raw source HTML).

| Field | Type | Notes |
|-------|------|-------|
| `document_id` | string (PK) | |
| `workspace_id` | string (FK) | |
| `version` | int | recomposed as candidates update |
| `cards` | jsonb | ordered typed candidate cards (name/rating/price/availability/provenance/artifact refs) |
| `action_descriptors` | jsonb | `[{action_type, risk_class, target_ref, requires_approval}]` (resolved by policy engine) |
| `filters_applied` | jsonb | |
| `rank_basis` | jsonb | weights + order used |
| `created_at` | timestamptz | |

**Validation**: contains no raw/executable source HTML; every `action_descriptor`
with a risky `action_type` MUST have `requires_approval = true`.

## Entity: Event

Durable, ordered, resumable workspace event.

| Field | Type | Notes |
|-------|------|-------|
| `event_id` | string (PK) | |
| `workspace_id` | string (FK) | |
| `sequence` | bigint | monotonic per workspace; UNIQUE `(workspace_id, sequence)` |
| `type` | enum | `plan.created`,`question.required`,`search.started`,`candidate.found`,`sandbox.started`,`sandbox.step`,`artifact.created`,`candidate.rank_updated`,`workspace.composed`,`approval.required`,`action.completed`,`workspace.completed`,`error` |
| `payload` | jsonb | |
| `created_at` | timestamptz | |

**Validation**: `sequence` allocated transactionally with the state change it
describes; events are immutable.

## Entity: Artifact

Captured output blob metadata (blob in GCS).

| Field | Type | Notes |
|-------|------|-------|
| `artifact_id` | string (PK) | |
| `workspace_id` / `sandbox_id` | string (FK) | |
| `type` | enum | `screenshot`,`recording`,`extracted_doc`,`receipt`(post-MVP) |
| `uri` | string | GCS, region-pinned |
| `content_type` | string | |
| `redaction_status` | enum | `pending`,`redacted`,`not_required` |
| `retention_expires_at` | timestamptz | |

## Entity: AuditRecord

Immutable ledger of every meaningful action.

| Field | Type | Notes |
|-------|------|-------|
| `audit_id` | string (PK) | |
| `workspace_id` | string (FK) | |
| `actor` | string | user/service/system |
| `action` | string | e.g. `search.run`,`extract`,`policy.deny`,`approval.grant` |
| `inputs_ref` | jsonb | references, not raw sensitive payloads |
| `outcome` | jsonb | |
| `provenance` | jsonb | |
| `created_at` | timestamptz | append-only |

## Entity: IdempotencyRecord

Backs exactly-once mutating effects.

| Field | Type | Notes |
|-------|------|-------|
| `tenant_id` + `idempotency_key` | composite PK | UNIQUE |
| `request_fingerprint` | string | hash of method+path+body |
| `response_snapshot` | jsonb | replayed on retry |
| `created_at` | timestamptz | TTL-eligible |

## Relationships

- Workspace 1—1 Plan, 1—1 (latest) WorkspaceDocument, 1—N Candidate, 1—N Event, 1—N AuditRecord.
- Candidate 1—N BrowserSandbox; BrowserSandbox 1—N Artifact.
- Workspace N—1 PolicyProfile.
- Approval N—1 Workspace, optional N—1 BrowserSandbox.
