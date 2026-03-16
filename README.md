# Meridian

**Architecture-first control plane for production-grade RAG and agentic AI systems.**

---

## At a Glance

Meridian is a governed RAG control plane for enterprise AI systems.  
It separates deterministic governance from probabilistic LLM inference, enabling reliable AI deployments with retrieval validation, evaluation pipelines, and provider-agnostic model integration.

**Core capabilities**

- **AI Operations Agent** — ReAct reasoning over ServiceNow incidents, changes, and knowledge base
- Hybrid retrieval (pgvector + Azure AI Search)
- Confidence-gated generation with optional calibrated scoring (ADR-0016)
- Provider-agnostic LLM integration (Ollama / Azure OpenAI)
- **Evaluation metrics** — aggregate telemetry persisted to Azure SQL (confidence, latency, refusal rate, per-query feedback)
- Multi-turn conversation history (client-owned, retrieval-independent)
- Enterprise connectors (ServiceNow Knowledge Base)
- Structured telemetry and evaluation harness
- Terraform-based infrastructure deployment
- CI/CD pipelines for reliable operations

**Primary technologies**

Python • FastAPI • Azure OpenAI (function calling) • Azure SQL • Azure AI Search • Terraform • Docker

---

## Overview

Meridian is a reference implementation of a retrieval-governed control plane for AI systems. It establishes a strict boundary between probabilistic inference and deterministic governance: the control plane decides when generation is permitted; the LLM decides what to say.

It enforces:

- Deterministic retrieval thresholds
- Explicit failure semantics
- Citation validation
- Offline evaluation discipline
- Versioned architectural decisions (ADRs)
- Structured telemetry logging

Meridian separates probabilistic reasoning from deterministic control.

Control precedes generation.
Observability precedes scale.
Governance precedes automation.

---
## Deployment Architecture

Meridian is designed to run as a containerized control plane service.

Typical deployment architecture:

```
User / API Client
      │
      ▼
Meridian API (FastAPI)
      │
      ├── Retrieval Layer
      │     • Chroma (local dev)
      │     • Azure AI Search (production)
      │
      ├── Model Providers
      │     • Ollama (local)
      │     • Azure OpenAI (cloud)
      │
      ├── Ingestion Pipeline
      │     • parse → chunk → embed → index
      │     • txt, md, pdf, docx
      │
      ├── AI Operations Agent
      │     • ReAct executor (GPT-4o function calling)
      │     • ServiceNow tools (incidents, changes)
      │     • Knowledge base tool (existing RAG)
      │
      ├── Calibration (ADR-0016)
      │     • Isotonic regression: raw scores → P(relevant)
      │     • Optional — disabled by default, raw scores pass through
      │
      ├── Evaluation + Telemetry
      │     • Azure SQL telemetry store
      │     • Aggregate metrics (confidence, latency, refusal rate)
      │
      └── Structured Logging
            • JSON telemetry on every request
            • Per-stage timing (t_retrieve_ms, t_generate_ms, t_total_ms)
```

Infrastructure is provisioned using Terraform and deployed through CI/CD pipelines.
---

## Provider Invariance

The control plane is provider-agnostic by construction. Threshold gating, refusal semantics, citation requirements, and telemetry are implemented once and are identical regardless of which LLM or retrieval backend is active. Provider selection is an adapter-layer concern. Governance is not.

```
┌─────────────────────────────────────────────────────────────┐
│                     Control Plane                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Threshold  │  │   Refusal   │  │     Telemetry       │  │
│  │   Gating    │  │  Semantics  │  │     Logging         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                    (Provider-Invariant)                     │
└─────────────────────────────────────────────────────────────┘
                            │
           ┌────────────────┴────────────────┐
           │                                 │
    ┌──────▼──────┐                   ┌──────▼──────┐
    │ LLMProvider │                   │  Retrieval  │
    │    (ABC)    │                   │Adapter (ABC)│
    └──────┬──────┘                   └──────┬──────┘
           │                                 │
     ┌─────┴─────┐                     ┌─────┴─────┐
     │           │                     │           │
┌────▼───┐ ┌─────▼──────┐         ┌─────▼───┐ ┌─────▼──────┐
│ Ollama │ │Azure OpenAI│         │ Chroma  │ │Azure Search│
└────────┘ └────────────┘         └─────────┘ └────────────┘
```

Provider selection is config-driven via two environment variables — no code changes required:

| Variable | `local` (default) | `azure` |
|---|---|---|
| `LLM_PROVIDER` | Ollama | Azure OpenAI |
| `RETRIEVAL_PROVIDER` | Chroma | Azure AI Search |

### Adapter Configurations

| Mode | LLM | Retrieval | Context |
|------|-----|-----------|---------|
| `local` | Ollama | Chroma | Development |
| `azure` | Azure OpenAI | Azure AI Search | Production |
| `hybrid` | Azure OpenAI | Chroma | Transitional |

### Azure Adapter Setup

```bash
cp .env.example .env
# Configure Azure credentials

python scripts/setup_azure_index.py
python scripts/seed_azure_data.py

LLM_PROVIDER=azure RETRIEVAL_PROVIDER=azure \
  python -m uvicorn api.main:app --reload
```

See [ADR-0006: Multi-Cloud Provider Strategy](adr/0006-multi-cloud-provider-strategy.md) for the architectural rationale.

---

## v0 Scope

Meridian v0 establishes a single-agent, retrieval-governed control plane.

### Implemented in v0:

- Single-agent RAG with deterministic control discipline
- Fixed-window chunking
- Local embeddings + persistent Chroma vector store
- Provider abstraction layer (LLM + retrieval)
- Confidence scoring with configurable threshold and optional calibration (isotonic regression, ADR-0016)
- Structured QueryResponse schema
- Explicit control states:
  - OK (HTTP 200)
  - REFUSED (HTTP 422)
  - UNINITIALIZED (HTTP 503)
- Lazy embedding initialization (runtime-safe)
- JSON structured telemetry logging with per-stage RAG timing (`t_retrieve_ms`, `t_generate_ms`, `t_total_ms`)
- Health and evaluation pre-flight enforcement
- Offline evaluation harness
- Versioned Architecture Decision Records (ADRs)
- MCP transport layer (stdio and HTTP/SSE) with CORS support for browser agents
- Azure AI service layer (Language, Vision, Speech, Document Intelligence)
- Architecture diagram (`docs/architecture-diagram.html`)
- Application version injectable via `VERSION` env var (CI/CD-friendly)
- Multi-turn conversation history (client-owned, threaded through LLM providers)
- ServiceNow Knowledge Base connector with delta sync (`POST /ingest/servicenow`, `GET /ingest/servicenow/status`)
- AI Operations Agent with ReAct reasoning over ServiceNow + KB (`POST /agent/query`)
- Evaluation metrics persisted to Azure SQL (`GET /evaluation/metrics`, `GET /evaluation/queries`)
- Per-query feedback collection (`POST /evaluation/queries/{trace_id}/feedback`)
- Calibrated confidence scoring via isotonic regression (`CALIBRATION_ENABLED=true`, ADR-0016)
- Cold start optimization: DB connection pool warmup, HTTP health probes, `minReplicas: 1`
- Azure AD / Entra ID authentication with role-based endpoint protection (ADR-0018)
- Intelligent container heartbeat — Azure Function keeps containers warm during business hours (ADR-0019)
- SSE streaming for `POST /query` — first token in ~1s vs full response wait (#14)
- Runtime temperature lock — operators can adjust LLM temperature (0.0–2.0) via `POST /settings`
- Enterprise integration: Semantic Kernel plugin + MCP API key auth + Claude Desktop support (ADR-0020)

Meridian separates probabilistic inference from deterministic control.
The control plane governs when inference is allowed.

### Non-goals for v0:

- Multi-agent orchestration (single agent with tool-use in v1.0)
- Multi-tenancy
- gRPC transport
- Cloud provisioning
- Observability tracing (OpenTelemetry)

---

## Architecture
Meridian is structured as a layered service model with explicit separation between API surface, control plane, providers, and infrastructure adapters.

See:

- [ARCHITECTURE.md](ARCHITECTURE.md)
- [PROCESS.md](PROCESS.md)
- [adr/](adr/)

---

## MCP Transport

Meridian's control plane is accessible over the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), allowing agent frameworks and Claude Desktop to query the governed knowledge base directly.

Two transports are provided:

| Transport | Entry point | Use case |
|-----------|-------------|----------|
| stdio | `server_mcp/server.py` | Claude Desktop, CLI agents |
| HTTP/SSE | `server_mcp/http_server.py` | Web agents, remote integration |

**Tools exposed:**

| Tool | Behaviour |
|------|-----------|
| `query_knowledge_base` | Returns grounded answer or structured refusal — governance semantics preserved |
| `check_health` | Returns system status and document count |

**stdio (Claude Desktop):**

```bash
python -m server_mcp.server
```

Claude Desktop config (`~/.config/claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "meridian": {
      "command": "python",
      "args": ["-m", "server_mcp.server"],
      "cwd": "/path/to/meridian"
    }
  }
}
```

**HTTP/SSE:**

```bash
uvicorn server_mcp.http_server:app --port 8001
```

MCP is a transport adapter only. Threshold gating, refusal semantics, and telemetry are enforced identically regardless of transport.

See [ADR-0007: MCP Integration](adr/0007-mcp-integration.md) and [docs/mcp-integration.md](docs/mcp-integration.md) for the full integration guide.

---

## Azure AI Services

Meridian routes Azure Cognitive Services calls server-side, keeping credentials out of the client and applying consistent telemetry on every request.

**Endpoints:**

| Endpoint | Operation |
|----------|-----------|
| `POST /azure-ai/language/sentiment` | Sentiment analysis |
| `POST /azure-ai/language/entities` | Named entity recognition |
| `POST /azure-ai/language/key-phrases` | Key phrase extraction |
| `POST /azure-ai/language/detect` | Language detection |
| `POST /azure-ai/vision/analyze` | Image analysis (caption, tags, objects, people) |
| `POST /azure-ai/vision/ocr` | Text extraction (OCR) |
| `POST /azure-ai/speech/transcribe` | Speech-to-text (upload WAV audio) |
| `POST /azure-ai/speech/synthesize` | Text-to-speech (returns WAV bytes) |
| `POST /azure-ai/document/analyze` | Document Intelligence — layout, forms, invoices, receipts, IDs |

**Configuration** — add to `.env`:
```bash
AZURE_LANGUAGE_ENDPOINT=https://<resource>.cognitiveservices.azure.com/
AZURE_LANGUAGE_KEY=<key>

AZURE_VISION_ENDPOINT=https://<resource>.cognitiveservices.azure.com/
AZURE_VISION_KEY=<key>

AZURE_SPEECH_KEY=<key>
AZURE_SPEECH_REGION=eastus

AZURE_DOCUMENT_ENDPOINT=https://<resource>.cognitiveservices.azure.com/
AZURE_DOCUMENT_KEY=<key>
```

Services return `503` when credentials are not set. See [ADR-0008](adr/0008-azure-ai-integration.md) and [ADR-0009](adr/0009-speech-document-intelligence.md).

---

## Document Ingestion

`POST /ingest` accepts file uploads and runs them through the full ingestion pipeline: **parse → chunk → embed → index**.

```bash
# Single file
curl -X POST http://localhost:8000/ingest -F "files=@docs/runbook.pdf"

# Multiple files
curl -X POST http://localhost:8000/ingest \
  -F "files=@docs/runbook.pdf" \
  -F "files=@docs/architecture.md"
```

**Response:**
```json
{"ingested": 2, "chunks": 34, "message": "2 documents ingested (34 chunks)"}
```

| Stage | What happens |
|-------|-------------|
| Parse | Extract text from `.txt`, `.md`, `.pdf` (PyMuPDF), `.docx` (python-docx) |
| Chunk | Split into ~2000-char passages with 200-char overlap |
| Embed | Handled by the retrieval adapter (`SentenceTransformer` auto-embed) |
| Index | Write to configured vector store (Chroma or Azure AI Search) |

Unsupported file types return HTTP 400. Empty files are skipped (not counted in `ingested`).

The pipeline specification is in `docs/internal/INGEST_SPEC.md` (not tracked — see local copy).

---

## ServiceNow Knowledge Base Connector

`POST /ingest/servicenow` connects to a ServiceNow instance, fetches KB articles, strips HTML, and indexes them through the same chunk → embed → index pipeline.

```bash
curl -X POST http://localhost:8000/ingest/servicenow \
  -H "Content-Type: application/json" \
  -d '{
    "instance_url": "https://dev12345.service-now.com",
    "username": "admin",
    "password": "password",
    "kb_name": "IT Knowledge Base",
    "limit": 50
  }'
```

**Response:**
```json
{"ingested": 12, "chunks": 87, "message": "12 ServiceNow articles ingested (87 chunks)"}
```

| Field | Required | Description |
|-------|----------|-------------|
| `instance_url` | No* | ServiceNow instance URL |
| `username` | No* | API user |
| `password` | No* | API user password |
| `kb_name` | No | Filter by knowledge base name |
| `category` | No | Filter by KB category |
| `since` | No | ISO timestamp for delta sync (only articles updated after this time) |
| `limit` | No | Maximum articles to fetch (0 = all) |

\* Credentials can be provided via environment variables (`SERVICENOW_INSTANCE_URL`, `SERVICENOW_USERNAME`, `SERVICENOW_PASSWORD`). Request body values take precedence.

**Delta sync** — fetch only articles updated since the last sync:
```bash
curl -X POST http://localhost:8000/ingest/servicenow \
  -H "Content-Type: application/json" \
  -d '{"since": "2026-03-09T00:00:00"}'
```

**Sync status** — check connection state and sync history:
```bash
curl http://localhost:8000/ingest/servicenow/status
```

```json
{
  "configured": true,
  "last_sync": {
    "started_at": "2026-03-09T10:00:00+00:00",
    "status": "success",
    "ingested": 12,
    "chunks": 87,
    "delta": false
  },
  "history": [...]
}
```

**Configuration** — add to `.env`:
```bash
SERVICENOW_INSTANCE_URL=https://dev12345.service-now.com
SERVICENOW_USERNAME=admin
SERVICENOW_PASSWORD=password
```

Get a free Personal Developer Instance at [developer.servicenow.com](https://developer.servicenow.com).

See [ADR-0014: ServiceNow Knowledge Base Connector](adr/0014-servicenow-knowledge-base-connector.md) for the architectural rationale.

---

## AI Operations Agent

`POST /agent/query` runs a multi-step reasoning agent that can investigate operational questions by querying ServiceNow incidents, change requests, and the Meridian knowledge base.

```bash
curl -X POST http://localhost:8000/agent/query \
  -H "Content-Type: application/json" \
  -d '{"question": "Why are login requests failing for region us-east?"}'
```

**Response:**
```json
{
  "trace_id": "abc-123",
  "status": "OK",
  "answer": "Based on INC0010042, the auth service in us-east experienced a certificate expiration...",
  "steps": [
    {"step": 1, "tool": "search_incidents", "input": {"query": "login failure us-east"}, "elapsed_ms": 340},
    {"step": 2, "tool": "get_incident_detail", "input": {"incident_number": "INC0010042"}, "elapsed_ms": 280},
    {"step": 3, "tool": "query_knowledge_base", "input": {"question": "certificate renewal procedure"}, "elapsed_ms": 150}
  ],
  "steps_taken": 3,
  "elapsed_ms": 4200
}
```

**Agent tools (read-only):**

| Tool | ServiceNow Table | Description |
|------|-----------------|-------------|
| `search_incidents` | `incident` | Search by keyword, priority, category, state |
| `get_incident_detail` | `incident` | Full incident with work notes and resolution |
| `search_changes` | `change_request` | Deployment and change history |
| `query_knowledge_base` | — | Existing RAG pipeline (retrieval + governance) |

**Governance constraints:**
- Maximum step budget per query (default: 5, max: 10)
- Read-only ServiceNow access — no mutations
- Every tool call logged with `trace_id` and `elapsed_ms`
- All agent activity persisted to Azure SQL for evaluation

**List available tools:**
```bash
curl http://localhost:8000/agent/tools
```

See [ADR-0015: AI Operations Agent](adr/0015-ai-operations-agent.md) for the architectural rationale.

---

## Evaluation Metrics

`GET /evaluation/metrics` returns aggregate telemetry computed from the Azure SQL query log — proving system reliability over time.

```bash
curl http://localhost:8000/evaluation/metrics
```

**Response:**
```json
{
  "configured": true,
  "total_queries": 847,
  "avg_confidence": 0.7423,
  "retrieval_precision": 0.8912,
  "refusal_rate": 0.0614,
  "latency_p50_ms": 580,
  "latency_p95_ms": 1240,
  "queries_by_status": {"OK": 795, "REFUSED": 52},
  "queries_by_source": {"query": 810, "agent": 37},
  "period_start": "2026-02-10T00:00:00+00:00",
  "period_end": "2026-03-10T18:00:00+00:00"
}
```

| Metric | Description |
|--------|-------------|
| `avg_confidence` | Mean best-chunk confidence across all queries |
| `retrieval_precision` | Ratio of chunks above threshold to total retrieved |
| `refusal_rate` | Fraction of queries refused by governance |
| `latency_p50_ms` / `latency_p95_ms` | Response time percentiles |
| `queries_by_status` | Breakdown by OK / REFUSED / UNINITIALIZED |
| `queries_by_source` | Breakdown by query (RAG) vs agent |

**Recent queries:**
```bash
curl "http://localhost:8000/evaluation/queries?limit=20"
```

**Submit feedback (thumbs-up / thumbs-down):**
```bash
curl -X POST http://localhost:8000/evaluation/queries/<trace_id>/feedback \
  -H "Content-Type: application/json" \
  -d '{"rating": "up"}'
```

Returns `200` on success, `404` if trace not found, `422` if rating is not `"up"` or `"down"`, `503` if DB not configured.

**Configuration** — add to `.env`:
```bash
DATABASE_URL=mssql+pyodbc://<user>:<pass>@<server>.database.windows.net/<db>?driver=ODBC+Driver+18+for+SQL+Server
```

Evaluation is optional — all endpoints return graceful responses when `DATABASE_URL` is not configured.

---

## Calibrated Confidence Scoring

By default, `confidence_score` is a raw similarity proxy (`max(1 - L2_distance)`). When calibration is enabled (ADR-0016), raw scores are mapped to calibrated probabilities via isotonic regression — making the threshold gate's decision probabilistically meaningful.

**How it works:**

```
retrieval → raw distances → 1 - distance → calibrate() → P(relevant) → threshold gate
```

When calibration is disabled (default), `confidence_score` and `raw_confidence` are identical. When enabled, `confidence_score` is the calibrated probability and `raw_confidence` preserves the original uncalibrated score.

**Setup:**

1. Generate labeled query-relevance pairs (see `data/calibration/sample_labels.json` for format)
2. Fit the calibration model:
   ```bash
   python scripts/fit_calibration.py --data data/calibration/labels.json --output data/calibration/calibration_model.pkl
   ```
3. Enable in `.env`:
   ```bash
   CALIBRATION_ENABLED=true
   CALIBRATION_MODEL_PATH=data/calibration/calibration_model.pkl
   ```

**Configuration:**

| Variable | Default | Description |
|----------|---------|-------------|
| `CALIBRATION_ENABLED` | `false` | Enable calibrated scoring |
| `CALIBRATION_MODEL_PATH` | (empty) | Path to fitted `.pkl` model |

When disabled, the system behaves identically to previous versions. See [ADR-0016: Calibrated Confidence Scoring](adr/0016-calibrated-confidence-scoring.md).

---

## Authentication (Azure AD / Entra ID)

Meridian supports JWT-based authentication via Azure AD (Entra ID). When `AUTH_ENABLED=True`, all API endpoints require a valid Bearer token and enforce role-based access control.

**Roles:**

| Role | Access |
|------|--------|
| `viewer` | Query, read settings, evaluation data, agent tools, sync status |
| `operator` | All viewer permissions + ingest, settings changes, Azure AI services |

**Open endpoints** (no auth required): `GET /ping`, `GET /health`

**Local development:** `AUTH_ENABLED=False` (default) returns a synthetic operator user — all endpoints work without tokens. Zero breaking changes.

**Configuration** — add to `.env`:
```bash
AUTH_ENABLED=true
AUTH_TENANT_ID=<azure-ad-tenant-id>
AUTH_CLIENT_ID=<app-registration-client-id>
AUTH_OPERATOR_GROUP_ID=<optional-group-oid>
AUTH_JWKS_CACHE_TTL_S=3600
```

**Token flow:**
```
Authorization: Bearer <JWT>
  → PyJWT validates signature via Azure AD JWKS
  → Extract claims (oid, preferred_username, roles)
  → UserInfo dataclass → route handler
  → user.oid flows to QueryLog.user_id
```

Role extraction checks the `roles` JWT claim (Azure AD app roles) first, then falls back to group membership matching via `AUTH_OPERATOR_GROUP_ID`. If no operator role is found, the user defaults to `viewer`.

See [ADR-0018: Azure AD Authentication](adr/0018-azure-ad-authentication.md) for the architectural rationale.

---

## SSE Streaming

`POST /query?stream=true` returns Server-Sent Events (SSE), delivering the first token in ~1 second instead of waiting for the full response.

**Request:**
```bash
curl -N -X POST "http://localhost:8000/query?stream=true" \
  -H "Content-Type: application/json" \
  -d '{"question": "How do I rollback a deployment?"}'
```

**SSE events:**

| Event | When | Payload |
|-------|------|---------|
| `metadata` | After retrieval, before generation | `trace_id`, `status`, `confidence_score`, `threshold`, `retrieval_scores`, `t_retrieve_ms` |
| `token` | Each LLM token chunk | `{"text": "..."}` |
| `done` | Generation complete | `trace_id`, `t_retrieve_ms`, `t_generate_ms`, `t_total_ms` |
| `error` | Refusal or failure | `status`, `refusal_reason`, `confidence_score` |

**Example stream:**
```
event: metadata
data: {"trace_id":"abc-123","status":"OK","confidence_score":0.87,"t_retrieve_ms":120}

event: token
data: {"text":"Based on"}

event: token
data: {"text":" the deployment guide"}

event: done
data: {"trace_id":"abc-123","t_retrieve_ms":120,"t_generate_ms":3400,"t_total_ms":3520}
```

**Governance invariant:** Retrieval, confidence scoring, and the refusal gate execute *before* the first token is streamed. If the query is refused, a single `error` event is sent and the stream ends — no partial generation.

Without `?stream=true`, `POST /query` returns the same blocking JSON response as before (100% backward compatible).

---

## Enterprise Integration

Meridian exposes its knowledge engine to agent frameworks via thin adapters (ADR-0020). The REST API is the stable boundary — plugins wrap it.

### Semantic Kernel Plugin

```python
from semantic_kernel import Kernel
from integrations.semantic_kernel import MeridianPlugin

kernel = Kernel()
kernel.add_plugin(MeridianPlugin(
    base_url="https://meridian-api.azurecontainerapps.io",
    api_key="your-bearer-token",
))
```

| Kernel Function | Endpoint | Description |
|----------------|----------|-------------|
| `query_knowledge` | `POST /query` | Query the governed knowledge base |
| `query_with_agent` | `POST /agent/query` | Run the AI Operations Agent |
| `get_status` | `GET /health` | Check system health |

### Claude Desktop (MCP)

Configure Claude Desktop to connect via Streamable HTTP transport:

```json
{
  "mcpServers": {
    "meridian": {
      "url": "https://mcp.vplsolutions.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_API_KEY"
      }
    }
  }
}
```

Config location: `%APPDATA%\Claude\claude_desktop_config.json` (Windows) or `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS).

### MCP Authentication

When `MCP_API_KEY` is set on the MCP Container App, all tool endpoints require `Authorization: Bearer <key>`. Health and root endpoints remain unauthenticated for probes.

| Endpoint | Auth Required |
|----------|--------------|
| `GET /`, `GET /health` | No |
| `GET /tools`, `POST /tools/call`, `POST /mcp` | Yes (when `MCP_API_KEY` is set) |

See [`integrations/README.md`](integrations/README.md) for full setup instructions.

---

## Intelligent Container Heartbeat

An Azure Function (Consumption plan) pings `/health` on each Container App at configurable intervals to prevent idle-to-zero scaling during business hours, while allowing containers to sleep during nights and weekends (ADR-0019).

**Architecture:**
```
Azure Function App (Consumption plan)
  └── heartbeat_timer (Timer trigger, every 3 min)
        ├── GET meridian-api/health
        ├── GET meridian-mcp/health
        └── GET meridian-studio/health
```

**Features:**
- Business-hours scheduling (default: 7 AM – 7 PM CST weekdays)
- Configurable active window, days, and timezone
- Consecutive failure tracking with webhook alerting (Teams/Slack)
- Estimated 50-70% cost reduction vs always-on `minReplicas: 1`

**Configuration** — set in Azure Function App settings:
```bash
HEARTBEAT_TARGETS=https://meridian-api.azurecontainerapps.io,https://meridian-mcp.azurecontainerapps.io
HEARTBEAT_ACTIVE_START=07:00        # business hours start (default)
HEARTBEAT_ACTIVE_END=19:00          # business hours end (default)
HEARTBEAT_ACTIVE_DAYS=Mon,Tue,Wed,Thu,Fri
HEARTBEAT_ALERT_THRESHOLD=3         # consecutive failures before alert
HEARTBEAT_ALERT_WEBHOOK=            # Teams/Slack webhook URL (optional)
```

**Deployment:**
```bash
cd functions/heartbeat
func azure functionapp publish <function-app-name>
```

See [ADR-0019: Intelligent Container Heartbeat](adr/0019-intelligent-container-heartbeat.md) for the architectural rationale.

---

## Running Locally

**Seed the vector store** (manual — or use `POST /ingest` above)
```bash
python scripts/seed_data.py
```

**Start the API**
```bash
python -m uvicorn api.main:app --host 127.0.0.1 --port 8000 --reload
```

---

## Tests

The test suite lives in `tests/` and uses pytest. Run from the project root:

```bash
python -m pytest tests/ -v
```

**Control plane tests (`tests/test_control_plane.py`)**

| Test | What it covers |
|---|---|
| `test_strong_match` | Returns `status: "OK"` with `answer`, `trace_id`, `confidence_score >= 0.20` |
| `test_irrelevant_query_refused` | Returns `status: "REFUSED"` with `refusal_reason: "Retrieval confidence below threshold"` and `confidence_score < 0.20` |
| `test_no_documents_refused` | Returns `status: "REFUSED"` with `refusal_reason: "No documents retrieved"` and `confidence_score == 0.0` |
| `test_conversation_history_forwarded_to_provider` | Conversation history reaches the LLM provider via `handle_query()` |
| `test_query_endpoint_accepts_conversation_history` | `POST /query` accepts optional `conversation_history` field and forwards it through the pipeline |
| `test_refusal_schema` | HTTP `/query` returns 422 with a flat `QueryResponse` body — `status`, `trace_id`, `confidence_score`, `refusal_reason` at the top level; no `detail` wrapper |

All control plane tests stub the LLM via `FakeProvider` — no Ollama or Azure connection required. `test_strong_match` and `test_irrelevant_query_refused` require the Chroma store to be seeded first. An unseeded store returns HTTP 503 with `status: "UNINITIALIZED"`.

**MCP transport tests (`tests/test_mcp.py`)**

| Test | What it covers |
|---|---|
| `test_root_returns_server_identity` | `GET /` returns `name`, `version`, `protocol` |
| `test_list_tools_*` | `GET /tools` exposes both tools with valid schemas |
| `test_call_query_ok` | `POST /tools/call` returns `status: "OK"` with `answer` when control plane approves |
| `test_call_query_refused` | `POST /tools/call` returns `status: "REFUSED"` with `reason` and `threshold` |
| `test_call_query_missing_question_returns_error` | Missing `question` argument returns `status: "ERROR"` |
| `test_call_health_*` | Health tool returns `healthy`, `uninitialized`, or `degraded` |
| `test_call_unknown_tool_returns_error` | Unknown tool name returns error body |
| `test_health_endpoint_*` | `GET /health` reflects store state correctly |
| `test_mcp_initialize` | `POST /mcp` initialize handshake returns server info and capabilities |
| `test_mcp_tools_list` | `POST /mcp` tools/list returns full tool manifest |
| `test_mcp_tools_call_dispatches` | `POST /mcp` tools/call dispatches and returns result |
| `test_mcp_unknown_method_returns_error` | Unrecognised MCP method returns error |

All MCP tests stub `handle_query` and `get_system_status` — no Chroma, Ollama, or Azure connection required.

**Azure AI tests (`tests/test_azure_ai.py`)**

| Test group | What it covers |
|---|---|
| `test_client_*` | Auth header injection, `AzureAICallMeta` on success, 4xx raises `AzureAIError`, retry count on 429/5xx |
| `test_language_*` | `sentiment`, `entities`, `key_phrases`, `detect_language` dispatch to correct `kind` |
| `test_vision_*` | `analyze_image` default features, `ocr` uses `Read` feature URL param |
| `test_speech_*` | `transcribe` returns recognized text, `NoMatch` raises `SpeechError`, retry on 429, `synthesize` returns WAV bytes |
| `test_document_*` | `analyze` returns structured result, poll `failed` raises `DocumentError`, retry on 429 submit |
| `test_endpoint_*` | All 9 HTTP endpoints return correct responses; 503 on missing config; errors map to upstream status codes |

All Azure AI tests stub network calls — no real Azure connection required.

**Hardening tests (`tests/test_hardening.py`)**

| Test group | What it covers |
|---|---|
| `TestMCPCors::test_cors_*` | MCP server CORS uses settings-based origins, wildcard removed |
| `TestAgentDeadline::test_agent_timeout_*` | Agent timeout kwarg passed to LLM provider |
| `TestIngestFileSize::test_*_file_*` | Oversized file rejected (413), small file accepted |
| `TestServiceNowSanitization::test_*` | Caret stripping, newline stripping, injection prevention |
| `TestPydanticConfig::test_*` | No deprecation warning, model_config present |
| `TestOllamaTimeout::test_*` | Default 60s, timeout used by provider |
| `TestErrorMessages::test_*` | Empty KB message references /ingest API |
| `TestNewConfigFields::test_*` | Agent timeout defaults, max upload size default |
| `TestFeedback::test_submit_feedback_*` | Up/down persisted, invalid rating 422, trace not found 404, DB unconfigured 503 |
| `TestWarmDbPool::test_warm_db_pool_*` | Pool warmup success path, no-engine no-op |

**Ingestion tests (`tests/test_ingest.py`)**

| Test | What it covers |
|---|---|
| `test_parsers_txt` / `test_parsers_md` | Text extraction from `.txt` and `.md` files |
| `test_parsers_unsupported` | `ValueError` for unknown file extensions |
| `test_chunker_small_text` | Text shorter than chunk size returns 1 chunk |
| `test_chunker_basic` / `test_chunker_overlap` | Multi-chunk splitting with correct overlap |
| `test_ingest_txt_file` | `POST /ingest` returns `ingested: 1` with mocked store |
| `test_ingest_multiple_files` | Two-file upload returns `ingested: 2` |
| `test_ingest_empty_file` | Empty file skipped, `ingested: 0` |
| `test_ingest_unsupported_format` | `.xyz` upload returns HTTP 400 |

All ingestion tests mock the vector store — no Chroma or Azure connection required.

**ServiceNow connector tests (`tests/test_servicenow.py`)**

| Test | What it covers |
|---|---|
| `test_strip_html_*` | HTML stripping: basic tags, plain text passthrough, whitespace collapse, empty, nested |
| `test_connector_fetches_articles` | Fetches articles, strips HTML, returns clean text with metadata |
| `test_connector_filters_by_kb_name` | `kb_name` filter appears in Table API query params |
| `test_connector_filters_by_category` | `category` filter appears in Table API query params |
| `test_connector_delta_sync_since` | `since` parameter adds `sys_updated_on` filter to query |
| `test_connector_respects_limit` | Returns at most `limit` articles |
| `test_connector_connection_error` | `RuntimeError` on unreachable instance |
| `test_connector_http_error` | `RuntimeError` with HTTP status on auth failure |
| `test_connector_empty_body_skipped` | Articles with empty body are returned (pipeline skips them) |
| `test_endpoint_missing_credentials` | Returns 400 when no credentials provided |
| `test_endpoint_ingests_articles` | `POST /ingest/servicenow` returns correct counts |
| `test_endpoint_with_filters` | Filters passed through to pipeline |
| `test_endpoint_runtime_error_returns_502` | Unreachable instance returns 502 |
| `test_endpoint_uses_env_credentials` | Falls back to `SERVICENOW_*` env vars |
| `test_endpoint_delta_sync_passes_since` | `since` field forwarded to pipeline |
| `test_status_endpoint_unconfigured` | Returns `configured: false` when env vars empty |
| `test_status_endpoint_tracks_sync_history` | Records successful sync in history |
| `test_status_endpoint_tracks_error` | Records failed sync with error message |

All ServiceNow tests mock HTTP calls — no real ServiceNow instance required.

**Agent tests (`tests/test_agent.py`)**

| Test group | What it covers |
|---|---|
| `TestToolRegistry::test_registry_*` | Tool registry contains all 4 tools, definitions match, valid OpenAI function schemas |
| `TestToolExecution::test_search_incidents_*` | ServiceNow incident search via Table API, unconfigured returns error |
| `TestToolExecution::test_get_incident_detail` | Incident detail retrieval by number |
| `TestToolExecution::test_search_changes` | Change request search |
| `TestToolExecution::test_query_knowledge_base_tool` | KB tool delegates to existing RAG pipeline |
| `TestToolExecution::test_execute_tool_logs_event` | Every tool call emits structured telemetry |
| `TestReActExecutor::test_agent_no_openai_config` | Returns error when Azure OpenAI not configured |
| `TestReActExecutor::test_agent_direct_answer` | LLM answers without tool calls |
| `TestReActExecutor::test_agent_tool_call_then_answer` | LLM calls tool → reasons → returns answer |
| `TestReActExecutor::test_agent_respects_step_budget` | Agent stops at max_steps and summarizes |
| `TestReActExecutor::test_agent_handles_llm_error` | LLM failure returns structured error |
| `TestAgentEndpoints::test_agent_query_*` | `POST /agent/query` returns structured response, validates max_steps |
| `TestAgentEndpoints::test_agent_tools_endpoint` | `GET /agent/tools` returns 4 tools |

All agent tests mock Azure OpenAI and ServiceNow API calls — no external connections required.

**Evaluation tests (`tests/test_evaluation.py`)**

| Test group | What it covers |
|---|---|
| `TestQueryLogModel::test_create_*` | SQLAlchemy model creation and field persistence |
| `TestQueryLogModel::test_*_to_dict` | Model serialization to dict |
| `TestQueryLogModel::test_agent_step_relationship` | QueryLog → AgentStep relationship |
| `TestEvaluationStore::test_*_no_db` | Graceful no-op when DATABASE_URL not configured |
| `TestEvaluationStore::test_get_metrics_with_data` | Aggregate metrics computed correctly from seeded data |
| `TestEvaluationStore::test_get_metrics_empty_period` | Zero-query period returns informative message |
| `TestEvaluationEndpoints::test_metrics_endpoint_*` | `GET /evaluation/metrics` returns structured response |
| `TestEvaluationEndpoints::test_queries_endpoint_*` | `GET /evaluation/queries` pagination and no-db fallback |
| `TestDatabaseInit::test_is_configured_*` | Database configuration detection |
| `TestDatabaseInit::test_init_db_no_config` | init_db is a no-op without DATABASE_URL |

All evaluation tests use in-memory SQLite — no Azure SQL connection required.

**Calibration tests (`tests/test_calibration.py`)**

| Test group | What it covers |
|---|---|
| `TestCalibratedScorer::test_passthrough_*` | Unfitted scorer returns raw scores unchanged |
| `TestCalibratedScorer::test_fit_and_calibrate` | Fitted model produces monotonic probabilities in [0, 1] |
| `TestCalibratedScorer::test_fit_minimum_pairs_enforced` | Rejects < 10 labeled pairs |
| `TestCalibratedScorer::test_fit_invalid_labels` | Rejects non-binary labels |
| `TestCalibratedScorer::test_save_and_load` | Model round-trips through serialization |
| `TestCalibratedScorer::test_out_of_bounds_clipped` | Scores outside training range clipped to [0, 1] |
| `TestControlPlaneCalibration::test_raw_confidence_in_refused_response` | REFUSED response includes `raw_confidence` |
| `TestControlPlaneCalibration::test_calibration_disabled_*` | Raw equals calibrated when disabled |
| `TestControlPlaneCalibration::test_calibration_enabled_*` | Scores transformed when enabled |
| `TestQueryLogRawConfidence::test_query_log_*` | QueryLog model accepts and serializes `raw_confidence` |

All calibration tests mock the retrieval store and scorer — no real model fitting in the test suite.

**Authentication tests (`tests/test_auth.py`)**

| Test group | What it covers |
|---|---|
| `TestAuthDisabled::test_*` | Endpoints work without token when auth disabled, local user is operator, ping always open |
| `TestGetCurrentUser::test_*` | Auth disabled returns local user, missing/invalid Bearer → 401, valid token → UserInfo |
| `TestTokenValidation::test_*` | Expired/wrong-audience/wrong-issuer/JWKS-failure tokens → 401 |
| `TestRoleExtraction::test_*` | App roles claim, unknown roles filtered, group OID fallback, default viewer |
| `TestEndpointProtection::test_*` | Operator rejects viewer (403), allows operator, viewer allows any auth user |
| `TestUserIdentityFlow::test_*` | user_id stored in QueryLog, included in to_dict(), forwarded by handle_query and run_agent |
| `TestJWKSClient::test_*` | PyJWKClient lazily created and cached |

All auth tests mock JWT validation and Azure AD — no real identity provider required.

**Streaming tests (`tests/test_streaming.py`)**

| Test group | What it covers |
|---|---|
| `TestSSEEvent::test_*` | SSE event formatting: correct `event:` and `data:` lines, JSON serialization |
| `TestBaseProviderStream::test_*` | Default `generate_stream()` fallback yields full response as single chunk |
| `TestOllamaStream::test_*` | Ollama streaming: NDJSON chunk parsing, stream=True flags, connection error |
| `TestAzureOpenAIStream::test_*` | Azure OpenAI streaming: SDK stream=True, delta content extraction |
| `TestHandleQueryStream::test_*` | Control plane streaming: metadata→tokens→done flow, uninitialized KB, refused low confidence |
| `TestStreamEndpoint::test_*` | `POST /query?stream=true` returns SSE, non-stream backward compatible, error events |

All streaming tests mock LLM providers and retrieval — no real model or network calls required.

#### Temperature Lock Tests (`tests/test_hardening.py`)

| Test | What it covers |
|---|---|
| `TestTemperatureLock::test_default_temperature_is_0_7` | Default AZURE_OPENAI_TEMPERATURE is 0.7 |
| `TestTemperatureLock::test_settings_response_includes_temperature` | `GET /settings` returns current temperature |
| `TestTemperatureLock::test_operator_can_update_temperature` | `POST /settings` with temperature updates the value |
| `TestTemperatureLock::test_temperature_rejects_below_zero` | Rejects temperature < 0.0 (422) |
| `TestTemperatureLock::test_temperature_rejects_above_two` | Rejects temperature > 2.0 (422) |
| `TestTemperatureLock::test_temperature_accepts_boundary_values` | Accepts 0.0 and 2.0 |
| `TestTemperatureLock::test_ollama_sends_temperature` | OllamaProvider passes temperature in request options |
| `TestTemperatureLock::test_null_temperature_preserves_current` | Omitting temperature preserves current value |

**Heartbeat tests (`tests/test_heartbeat.py`)**

| Test group | What it covers |
|---|---|
| `TestBusinessHours::test_*` | Business hours logic: weekday/weekend, before/after hours, boundary times, custom window, custom days |
| `TestConfiguration::test_*` | Environment variable parsing: targets CSV, empty targets, trailing slashes, alert threshold |
| `TestPingTarget::test_*` | Health check: healthy response, unhealthy status, timeout, connection error |
| `TestAlerts::test_*` | Webhook alerting: sends payload, skips when unconfigured, handles webhook failure |
| `TestHeartbeatTimer::test_*` | Timer orchestration: pings all targets, skips outside hours, skips no targets, failure tracking with threshold alert, counter reset on success |

All heartbeat tests mock HTTP calls and azure.functions — no Azure Function runtime required.

**Semantic Kernel plugin tests (`tests/test_semantic_kernel.py`)**

| Test group | What it covers |
|---|---|
| `TestMeridianPluginInit::test_*` | Default URL, trailing slash strip, API key header, no-key header |
| `TestQueryKnowledge::test_*` | Successful query with confidence/trace, refused query with reason/threshold |
| `TestQueryWithAgent::test_*` | Agent query with steps, elapsed time, trace ID |
| `TestGetStatus::test_*` | Health check JSON formatting |
| `TestKernelFunctionDecorators::test_*` | All three functions have SK metadata |

All SK tests mock HTTP calls and semantic_kernel — no real SK or Meridian server required.

**MCP API key auth tests (`tests/test_mcp.py :: TestMCPApiKeyAuth`)**

| Test | What it covers |
|---|---|
| `test_no_key_configured_allows_all` | No MCP_API_KEY = open endpoints |
| `test_key_configured_rejects_missing_token` | Missing Bearer token → 401 |
| `test_key_configured_rejects_wrong_token` | Wrong API key → 401 |
| `test_key_configured_accepts_correct_token` | Correct key passes auth |
| `test_health_is_unauthenticated` | GET /health open even with key set |
| `test_root_is_unauthenticated` | GET / open even with key set |
| `test_tools_call_requires_auth` | POST /tools/call protected |
| `test_mcp_endpoint_requires_auth` | POST /mcp protected |
| `test_mcp_endpoint_with_valid_key` | POST /mcp works with valid key |

---

## License

Apache 2.0
