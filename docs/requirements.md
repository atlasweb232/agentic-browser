# Agentic Browser Infrastructure Requirements

## 1. Purpose

Agentic Browser is a standalone, globally deployable infrastructure platform for turning user requests into controlled browser-based task workspaces.

The platform must accept requests from any modality or client, decompose the request into structured subtasks, run search and browser sandboxes in parallel, analyze and filter results, and either present a ranked result set or generate a composable browser-friendly workspace where the user can continue into selected action paths such as booking, purchasing, applying, scheduling, calling, or publishing.

This repository is separate from desktop, mobile, voice, telephony, and web applications. Those applications consume Agentic Browser through APIs and event streams.

## 2. In Scope

- Generic agentic browser task infrastructure.
- Request decomposition and task planning.
- Location-aware search and browsing.
- Parallel browser sandbox execution.
- Search, scrape, extract, normalize, analyze, and rank workflows.
- User-facing composable workspaces with filtered results.
- Browser-friendly result pages preserving useful source details, images, icons, links, and action buttons where policy allows.
- Human-in-the-loop approvals for external, irreversible, payment, communication, and personal-data-sharing actions.
- Decoupled payment, communication, validation, security, and audit services.
- Global multi-region and sovereign data center deployment model.
- Horizontal scaling and spike handling.
- OpenAI-first model integration, including configurable GPT-5.5-class planning/search models and Realtime-2-class voice integration through external Voice API.
- Optional Temporal orchestration research and adapter design.
- Support for invocation from Windows, macOS, Linux, Android, iOS, web, chat, voice, telephony, and future modalities.

## 3. Out Of Scope

- Building the client apps themselves.
- Building the secure Voice API itself.
- Building telephony infrastructure itself.
- Agentic coding platforms.
- Self-evolving agent harnesses.
- Local data gathering agents for sovereign deployments, except where Agentic Browser consumes registered data sources.
- Unrestricted user-desktop control.
- Silent purchasing, booking, form submission, calling, messaging, or publishing without policy and approval checks.

## 4. Design Principles

1. **Modality neutral**: Every client invokes the same backend APIs.
2. **Sandbox first**: Browser work runs in isolated sandboxes, not the user's live browser by default.
3. **Structured first**: Prefer APIs and structured sources before visual browser automation.
4. **Parallel by design**: Candidate discovery and page inspection run concurrently when safe.
5. **Human control**: Risky actions require explicit approval.
6. **Composable results**: Search results become a workspace the user can inspect and act from.
7. **Provider pluggability**: OpenAI is first-class, but models and browser engines remain adapter based.
8. **Sovereign deployability**: Tenant and location policy control where data is processed and stored.
9. **Auditability**: Every search, browser action, extraction, approval, and external action is recorded.
10. **Least privilege**: Tools, credentials, cookies, payment tokens, and communication channels are scoped per task.

## 5. Example User Requests

- "Find restaurants within 5 miles and book a table for two tonight."
- "Find gyms near me, compare monthly pricing, and prepare the membership application."
- "Do my grocery shopping for this week under $120."
- "Book a doctor appointment next week with someone who accepts my insurance."
- "Plan a vacation to Tokyo in April with flights, hotel, and a 5-day itinerary."
- "Find three vendors for office catering and request quotes."
- "Research local permits and prepare the application, but do not submit."
- "Find the best laptop under $1,500 and prepare checkout."

## 6. High-Level Architecture

```text
Clients and Modalities
Windows / macOS / Linux / Web / Android / iOS / Voice API / Chat / Telephony
        |
        v
Agentic Browser Gateway
Auth, tenant/user policy, request validation, workspace lifecycle
        |
        v
Task Planner
Request decomposition, criteria extraction, subtask graph creation
        |
        +-------------------+--------------------+------------------+
        |                   |                    |                  |
        v                   v                    v                  v
Search Service       Browser Sandbox Pool   Validation Service  Approval Service
Places/Web/TinyFish  Isolated browsers      Facts, pricing,     HITL checkpoints
OpenAI web_search    Computer use adapters  risk, compliance    consent records
        |                   |                    |                  |
        +-------------------+--------------------+------------------+
                            |
                            v
Result Synthesis and Workspace Composer
Candidate normalization, ranking, visual workspace generation
                            |
                            v
Workspace API and Event Stream
Filtered results, browser previews, action paths, artifacts, audit
```

## 7. Core Services

### 7.1 Agentic Browser Gateway

Responsibilities:

- Authenticate client and service calls.
- Resolve `tenant_id`, `user_id`, `workspace_id`, `region`, and policy profile.
- Accept workspace creation requests.
- Expose event streams for progress and browser updates.
- Enforce tenant/user entitlements and rate limits.
- Route requests to the correct regional control plane.

### 7.2 Task Planner

Responsibilities:

- Convert natural language requests into a structured task graph.
- Extract constraints such as location, radius, date, budget, category, preferences, and exclusions.
- Identify missing information and ask follow-up questions.
- Decide which subtasks can run in parallel.
- Select tools and sandbox modes.
- Produce an auditable plan before risky actions.

Planner output example:

```json
{
  "objective": "Find gyms within 5 miles and prepare a membership application",
  "constraints": {
    "radius_miles": 5,
    "category": "gym",
    "budget_monthly_usd_max": 75
  },
  "subtasks": [
    {"type": "location_search", "parallel": false},
    {"type": "candidate_page_inspection", "parallel": true},
    {"type": "pricing_extraction", "parallel": true},
    {"type": "ranking", "parallel": false},
    {"type": "prepare_application", "requires_approval": true}
  ]
}
```

### 7.3 Search Service

Responsibilities:

- Run structured location-aware searches.
- Use provider adapters:
  - Google Places or equivalent local Places provider.
  - TinyFish search.
  - OpenAI web search.
  - Browser-based search fallback.
  - Domain-specific APIs when available.
- Normalize results into candidate records.
- Attach source provenance and confidence.

Search must support:

- radius search
- geographic bounding
- category/type filters
- open-hours filters
- rating, price, distance, availability, and source filters
- local-language and local-region policies

### 7.4 Browser Sandbox Service

Responsibilities:

- Create isolated browser sessions on demand.
- Support sandbox modes:
  - `discovery`
  - `extract`
  - `prepare`
  - `submit_approved`
  - `payment_approved`
  - `communication_approved`
- Execute browser automation using adapter engines:
  - TinyFish
  - browser-use
  - OpenAI computer use
  - Playwright deterministic flows
  - future proprietary browser agents
- Stream screenshots, page state, extracted data, and step logs.
- Enforce domain allowlists, blocked domains, credential boundaries, download policies, and timeout limits.
- Capture artifacts.

### 7.5 Workspace Composer

Responsibilities:

- Convert candidates and extracted data into a user-facing workspace.
- Preserve useful source context:
  - images
  - logos/icons where legally and technically allowed
  - names
  - ratings
  - prices
  - availability
  - links
  - phone numbers
  - action buttons
  - source citations
- Render action paths:
  - inspect
  - compare
  - call
  - prepare form
  - add to cart
  - book
  - apply
  - submit after approval
- Keep generated pages tied to source provenance and artifact IDs.

### 7.6 Approval Service

Responsibilities:

- Create approval checkpoints.
- Require explicit approval for:
  - purchases
  - payment authorization
  - booking commitments
  - membership applications
  - appointment booking
  - outbound phone calls
  - emails/messages
  - form submission
  - document upload
  - sharing personal data
  - accepting terms
  - publishing
- Record approval text, risk class, scope, expiration, actor, and timestamp.
- Support voice, chat, web, and mobile approval confirmations.

### 7.7 Payment Service Boundary

Responsibilities:

- Keep payment authorization separate from browser execution.
- Tokenize payment methods through approved payment providers.
- Require explicit payment approval.
- Prevent browser sandboxes from directly accessing raw payment credentials unless policy allows a controlled payment injection flow.
- Record receipts and payment artifacts.

### 7.8 Agentic Communication Service Boundary

Responsibilities:

- Decouple communication actions from browser planning.
- Support:
  - outbound calls
  - inbound callbacks
  - SMS
  - email
  - chat/contact forms
  - supplier quote requests
- Require approval for external communication unless tenant automation policy allows it.
- Store transcripts, summaries, and outcomes.

### 7.9 Validation Service

Responsibilities:

- Check extracted data for consistency and freshness.
- Detect hallucinated, stale, suspicious, or contradictory results.
- Validate prices, availability, terms, fees, ratings, and addresses against multiple sources where possible.
- Flag risk conditions:
  - hidden fees
  - auto-renewal
  - cancellation penalties
  - medical/legal/financial sensitivity
  - high payment amount
  - untrusted domain
  - login required

### 7.10 Workflow Orchestration

Temporal is a candidate, not yet a committed dependency.

The orchestration layer must support:

- durable task state
- retries
- timeouts
- human wait states
- parallel fan-out/fan-in
- long-running workflows
- callbacks and webhooks
- cancellation
- replayable audit history

Candidate options:

- Temporal
- LangGraph persistence
- custom queue plus Postgres state machine
- cloud-native durable functions per sovereign region

Temporal research must evaluate:

- operational complexity
- multi-region deployment
- data residency
- workflow replay privacy
- cost
- SDK fit
- human approval wait states
- high-throughput browser-sandbox fan-out

## 8. Workspace Lifecycle

```text
created
planning
needs_user_info
searching
inspecting
composing
ready_for_review
awaiting_approval
executing_action
completed
cancelled
failed
expired
```

## 9. Browser Sandbox Lifecycle

```text
requested
allocated
starting
ready
running
awaiting_approval
completed
failed
expired
destroyed
```

## 10. API Requirements

### 10.1 Create Workspace

```http
POST /v1/agentic-browser/workspaces
```

Request:

```json
{
  "tenant_id": "tenant_123",
  "user_id": "user_123",
  "objective": "Find gyms within 5 miles and compare membership options",
  "modality": "voice",
  "location": {
    "latitude": 42.3601,
    "longitude": -71.0589,
    "radius_miles": 5,
    "permission_id": "loc_perm_123"
  },
  "constraints": {
    "budget_monthly_usd_max": 75,
    "must_have": ["pool", "open after 9pm"]
  }
}
```

Response:

```json
{
  "workspace_id": "abw_123",
  "status": "planning",
  "event_stream_url": "/v1/agentic-browser/workspaces/abw_123/events"
}
```

### 10.2 Stream Workspace Events

```http
GET /v1/agentic-browser/workspaces/{workspace_id}/events
```

Events:

- `plan.created`
- `question.required`
- `search.started`
- `candidate.found`
- `sandbox.started`
- `sandbox.step`
- `artifact.created`
- `candidate.rank_updated`
- `workspace.composed`
- `approval.required`
- `action.completed`
- `workspace.completed`
- `error`

### 10.3 Submit User Answer

```http
POST /v1/agentic-browser/workspaces/{workspace_id}/answers
```

### 10.4 Approve Action

```http
POST /v1/agentic-browser/approvals/{approval_id}/approve
```

### 10.5 Deny Action

```http
POST /v1/agentic-browser/approvals/{approval_id}/deny
```

### 10.6 Start Sandbox For Candidate

```http
POST /v1/agentic-browser/workspaces/{workspace_id}/sandboxes
```

### 10.7 Execute Approved Sandbox Action

```http
POST /v1/agentic-browser/sandboxes/{sandbox_id}/actions
```

## 11. Data Model Requirements

### Workspace

- `workspace_id`
- `tenant_id`
- `user_id`
- `region`
- `objective`
- `modality`
- `status`
- `policy_profile_id`
- `location_context`
- `created_at`
- `updated_at`
- `expires_at`

### Plan

- `plan_id`
- `workspace_id`
- `objective`
- `constraints`
- `subtasks`
- `missing_information`
- `risk_summary`
- `created_by_model`
- `model_version`

### Candidate

- `candidate_id`
- `workspace_id`
- `name`
- `category`
- `source_url`
- `source_provider`
- `location`
- `distance`
- `rating`
- `price_summary`
- `availability`
- `images`
- `icons`
- `action_options`
- `risk_flags`
- `normalized_data`
- `source_provenance`
- `rank_score`

### BrowserSandbox

- `sandbox_id`
- `workspace_id`
- `candidate_id`
- `engine`
- `mode`
- `status`
- `allowed_domains`
- `blocked_domains`
- `browser_endpoint`
- `recording_artifact_id`
- `created_at`
- `destroyed_at`

### Approval

- `approval_id`
- `workspace_id`
- `sandbox_id`
- `risk_class`
- `action_type`
- `summary`
- `data_to_share`
- `cost_amount`
- `payment_required`
- `expires_at`
- `status`
- `approved_by`
- `approved_at`

### Artifact

- `artifact_id`
- `workspace_id`
- `sandbox_id`
- `type`
- `uri`
- `content_type`
- `redaction_status`
- `retention_expires_at`

## 12. Security Requirements

- All provider API keys remain server-side.
- Browser sandboxes run in isolated containers or equivalent isolation.
- Each sandbox has network egress controls.
- Domain allowlists and denylists are enforced per workspace.
- No unrestricted access to user cookies.
- Credentials are injected only through scoped credential brokers.
- Payment credentials are never exposed to models.
- Screenshots and page text are redacted according to tenant policy.
- Prompt injection from webpages must be treated as untrusted data.
- Browser instructions from webpages must not override system policy.
- Downloads are blocked by default and scanned when allowed.
- File uploads require explicit approval.
- Authenticated sites require explicit user consent and scoped session handling.
- Sandboxes expire automatically.

## 13. Human-In-The-Loop Requirements

The system must pause and obtain approval before:

- final submit
- payment
- booking
- phone call
- message/email send
- document upload
- terms acceptance
- account creation
- membership application
- medical/financial/legal action
- personal data sharing
- irreversible or hard-to-reverse action

Approval must include:

- plain-language summary
- exact destination
- data being shared
- estimated or exact cost
- cancellation/refund risk where known
- screenshots or source references where useful
- expiration time

## 14. Scaling Requirements

- Horizontally scalable gateway.
- Horizontally scalable sandbox workers.
- Queue-backed fan-out for candidate inspection.
- Per-tenant/user/workspace concurrency limits.
- Autoscaling for browser sandbox pools.
- Backpressure responses for overload.
- Regional routing based on tenant/user/data residency.
- No single in-memory coordinator in production.
- Event streams must support reconnect and resume.
- Sandboxes must be pre-warmed in high-traffic regions.

## 15. Global And Sovereign Deployment Requirements

The platform must support sovereign data centers in each deployment location.

Requirements:

- region-pinned processing
- region-pinned artifact storage
- local policy profiles
- local model/provider routing
- local audit logs
- local secrets management
- cross-region metadata only when policy permits
- tenant migration controls
- provider availability fallback subject to data policy

OpenAI model usage must be configurable by region and policy. The target model family is GPT-5.5-class for planning and analysis and Realtime-2-class for voice-mediated interaction through the external Voice API.

## 16. Observability Requirements

Track:

- workspace creation rate
- planning latency
- search latency
- sandbox startup latency
- candidate inspection latency
- extraction success rate
- approval wait time
- submit success/failure rate
- browser crash rate
- CAPTCHA/intervention rate
- provider/model cost
- token usage
- sandbox minutes
- queue depth
- region capacity
- policy denials
- security violations

## 17. Testing Requirements

Automated test categories:

1. Request decomposition.
2. Missing information detection.
3. Location-aware search.
4. Parallel sandbox dispatch.
5. Candidate extraction.
6. Ranking and filtering.
7. Workspace composition.
8. Approval blocking.
9. Denied action handling.
10. Approved submit path.
11. Payment approval boundary.
12. Communication approval boundary.
13. Prompt-injection resistance.
14. Tenant/user isolation.
15. Region residency.
16. Sandbox cleanup.
17. Event stream reconnect.
18. Spike/concurrency test.
19. Artifact creation and retention.
20. Browser engine adapter parity.

## 18. Acceptance Criteria

- A user can create a workspace from voice, chat, or API with the same backend contract.
- The system decomposes the task into auditable subtasks.
- Location-based search returns normalized candidates.
- Multiple candidates can be inspected in parallel browser sandboxes.
- Results are filtered and ranked according to user criteria.
- A composable workspace is generated with result details, images/icons when allowed, and action buttons.
- The user can select a candidate and proceed down an action path.
- Submit/payment/call/booking/publishing actions require approval.
- All actions and artifacts are auditable.
- The system runs without relying on a specific client app.
- The architecture supports horizontal scaling and sovereign regional deployment.

## 19. Open Research Questions

- Should Temporal be the primary workflow engine or an adapter behind a workflow interface?
- What sandbox engine should be default: TinyFish, browser-use, OpenAI computer use, Playwright, or a hybrid?
- How much webpage visual material can be preserved in composed workspaces under copyright, source terms, and security policy?
- What is the safest credential-injection pattern for authenticated workflows?
- How should payment approvals be represented across voice, chat, web, and mobile?
- How should CAPTCHA and bot-detection failures be surfaced?
- What model split should be used between planning, extraction, validation, and summarization?
- How should local sovereign deployments share non-sensitive global learnings without leaking tenant/user data?

## 20. Initial Milestones

### Milestone 1: Requirements And Contracts

- Finalize this requirements document.
- Create OpenAPI contracts.
- Define data model.
- Define threat model.
- Define Temporal research plan.

### Milestone 2: Workspace MVP

- Create workspace API.
- Create task planner interface.
- Implement location search adapter.
- Implement candidate model.
- Implement SSE event stream.

### Milestone 3: Browser Sandbox MVP

- Implement sandbox creation.
- Add TinyFish/browser-use adapter.
- Add discovery and extract modes.
- Capture screenshots and artifacts.

### Milestone 4: Composable Workspace

- Generate filtered result board.
- Add candidate cards.
- Add action buttons.
- Add source provenance and artifacts.

### Milestone 5: Approval And Action Paths

- Add approval service.
- Add prepare/submit split.
- Add payment and communication boundaries.
- Add audit logs.

### Milestone 6: Scale And Sovereignty

- Add queue-backed parallel execution.
- Add regional routing.
- Add tenant concurrency limits.
- Add sandbox autoscaling.
- Add sovereign deployment profile.
