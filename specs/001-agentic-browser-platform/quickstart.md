# Quickstart: Agentic Browser Platform (MVP)

Local development + smoke validation for the MVP slice.

## Prerequisites

- Python 3.11+, Node.js 20+, Docker (Postgres + Redis via testcontainers/compose)
- `gcloud` authenticated; access to GCP Secret Manager project `solid-topic-466217-t9`
- Secrets are referenced as `gcp-secret:<name>` and loaded via `loadAllSecrets()`
  at startup — never place keys in `.env`. Required secrets:
  - `openai-api-key` (planning/extraction/vision capability adapters)
  - `tinyfish-api-key` (default browser engine)
  - `places-api-key`, `websearch-api-key` (search adapters)

## Run local infrastructure

```bash
docker compose up -d postgres redis        # Postgres = system of record, Redis = event fan-out/queue
alembic upgrade head                        # create schema (workspaces, events, approvals, audit, idempotency, …)
```

## Run backend

```bash
cd backend
uv sync                                     # or: pip install -r requirements.txt
python -m src.common.secrets_check          # verifies GCP SM references resolve
uvicorn src.gateway.app:app --reload --port 8080
```

## Run frontend (Workspace Document renderer)

```bash
cd frontend
npm install
npm run dev                                 # Vite dev server; renders the Workspace Document via trusted components
```

## Smoke test: create a workspace and watch it compose

```bash
# 1. Create a workspace (idempotency key required)
curl -sS -X POST localhost:8080/v1/agentic-browser/workspaces \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
        "objective": "Find gyms within 5 miles and compare membership pricing",
        "modality": "api",
        "location": {"latitude": 42.3601, "longitude": -71.0589, "radius_miles": 5},
        "constraints": {"budget_monthly_usd_max": 75, "must_have": ["pool"]}
      }'
# -> { "workspace_id": "abw_…", "status": "planning", "event_stream_url": "…/events" }

# 2. Stream events (resumable via Last-Event-ID)
curl -N localhost:8080/v1/agentic-browser/workspaces/$WS/events \
  -H "Authorization: Bearer $TOKEN"
# expect ordered: plan.created → search.started → candidate.found(*) →
#   sandbox.started/step → artifact.created → candidate.rank_updated → workspace.composed

# 3. Fetch the composed Workspace Document
curl -sS localhost:8080/v1/agentic-browser/workspaces/$WS/document \
  -H "Authorization: Bearer $TOKEN"
```

## Validate the safety gate (must hold in MVP)

```bash
# Execution-class sandbox mode is denied (FR-026)
curl -sS -o /dev/null -w "%{http_code}\n" -X POST \
  localhost:8080/v1/agentic-browser/workspaces/$WS/sandboxes \
  -H "Authorization: Bearer $TOKEN" -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"candidate_id":"cand_x","mode":"submit_approved"}'
# expect: 403

# Idempotent create: same key twice -> same workspace_id, one row
# (run step 1 twice with the SAME Idempotency-Key)

# Off-allowlist egress is blocked at the proxy (assert in integration test, not curl)
```

## Acceptance mapping (what to assert in tests)

| Check | Spec ref |
|-------|----------|
| Same idempotency key → one workspace/effect | SC-003, FR-003 |
| Candidates within radius + provenance present | SC-002, FR-010/012 |
| N sandboxes concurrent; one crash ≠ workspace fail | SC-004, FR-016 |
| Off-allowlist navigation blocked at proxy | SC-005, FR-014 |
| Risky action without bound approval denied | SC-006, FR-021/022/026 |
| Event stream reconnect: no loss/dupe | SC-007, FR-028/029 |
| Every action audited | SC-008, FR-030 |
| Workspace ready ≤ 60s P95 | SC-009 |

## Test suites

```bash
cd backend && pytest tests/unit tests/integration tests/contract
cd frontend && npm run test
```
