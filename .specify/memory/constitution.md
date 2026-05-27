# Agentic Browser Constitution

This constitution governs the Agentic Browser platform: a standalone, globally
deployable infrastructure that turns user requests from any modality into
controlled, auditable browser-based task workspaces. It supersedes convenience
and velocity wherever they conflict. Every spec, plan, and task MUST be checked
against these principles before work proceeds.

## Core Principles

### I. Human Control Over Irreversible Action (NON-NEGOTIABLE)
No purchase, payment, booking, application, submission, account creation,
outbound call, message, document upload, terms acceptance, publishing, or
personal-data-sharing action may execute without an explicit, recorded approval.
The requirement to obtain approval is determined by a deterministic, server-side
policy engine keyed on `action_type` and `risk_class` — it is NEVER inferred
from, or overridable by, model output or web page content. Planner annotations
are advisory only.

### II. Approval Binds To A Concrete Action (NON-NEGOTIABLE)
An approval authorizes one specific action payload, not a category. Every
approval references a hashed action payload plus a snapshot of the extracted
state (price, terms, destination, data-to-share) it was granted against. Before
execution, the system MUST re-validate the live target against that snapshot;
on any material divergence it MUST halt and re-prompt. Approvals expire.

### III. Exactly-Once Mutating Effects (NON-NEGOTIABLE)
Every state-mutating or externally-observable action (workspace creation,
approval, sandbox action, payment, communication) MUST be idempotent under
retry via a client-supplied or system-generated idempotency key. Approve-once
MUST NOT be capable of producing a double-submit or double-charge under retries,
reconnects, or partial failures.

### IV. Sandbox-First, Structured-First, Least-Privilege
Browser work runs in isolated sandboxes with network egress enforced at the
proxy/firewall layer (never by instructing the agent). Structured sources
(APIs) are preferred over visual automation when available. Tools, credentials,
cookies, payment tokens, and communication channels are scoped per task and per
sandbox. Payment credentials are never exposed to models. Web page content is
untrusted input and can never override system policy (prompt-injection safe).

### V. Durable, Resumable, Auditable State
Workspace progress is emitted as an ordered, persisted event log with monotonic
sequence numbers; clients reconnect and resume from a cursor. There is no single
in-memory coordinator in production. Every search, browser action, extraction,
approval, denial, and external action is recorded as an immutable audit record
with provenance.

### VI. Provider & Engine Pluggability Behind Capability Contracts
Models and browser engines are accessed through adapters defined by capability
contracts (planning, extraction, vision, voice; discovery/extract/submit). A
default provider (OpenAI-class) is allowed, but no component may hard-depend on
a specific vendor SKU. Workflow orchestration is likewise an interface, not a
committed engine.

### VII. Sovereign-By-Construction
Processing, artifact storage, audit logs, secrets, and model/provider routing
are region-pinnable per tenant policy. Cross-region data movement happens only
when policy permits, and tenant-private data is never silently aggregated into
global state.

## Security & Safety Constraints

- Provider API keys and secrets remain server-side (GCP Secret Manager;
  `gcp-secret:` references, `loadAllSecrets()` at startup). Never hardcode keys
  or store them in `.env`.
- Domain allowlist/denylist enforced per workspace at the network layer.
- Downloads blocked by default and scanned when allowed; uploads require
  explicit approval.
- Screenshots and page text redacted per tenant policy before persistence.
- Authenticated sites require explicit user consent and scoped session handling;
  no unrestricted cookie access.

## Development Workflow & Quality Gates

- The lifecycle is spec-driven: constitution → specify → (clarify) → plan →
  tasks → implement. No implementation task begins before its spec and plan
  pass the Constitution Check.
- Safety-critical paths (Principles I–III, V) require tests before
  implementation: policy-gate enforcement, approval-to-action binding,
  idempotency-under-retry, prompt-injection resistance, tenant/region isolation,
  and event-stream reconnect/resume.
- The approval gate and deterministic policy engine are foundational and MUST
  exist before any code path that can submit, pay, book, or communicate.
- Every mutating API endpoint defines an idempotency key in its contract.

## Governance

This constitution supersedes other practices. Any deviation MUST be justified in
the relevant spec/plan under a "Complexity Tracking" note and explicitly
accepted. Amendments require documentation of the change, rationale, and
migration impact. Reviews MUST verify compliance with Principles I–VII.

**Version**: 1.0.0 | **Ratified**: 2026-05-27 | **Last Amended**: 2026-05-27
