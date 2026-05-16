# Bond Data Intelligence Platform

> Unified query platform for time-split bond data — intelligent routing across dual MongoDB, NLP query interface, RAG-grounded prospectus validation, event-driven agentic reconciliation, and immutable audit trail with crypto-proof integrity.

---

**TL;DR**
An AI-powered platform for financial security master systems that:
- unifies time-split data across dual MongoDB instances into a single seamless query interface
- enables natural language querying with LLM-extracted parameters
- validates data using RAG over PDF prospectus documents
- automates reconciliation using agentic workflows with human-in-the-loop approval
- tracks every data override immutably with crypto-proof integrity (Day 5 — Tier 1 ✅)

---

## Why This Exists

This project is inspired by common challenges in financial data systems, where security master data for bonds is often siloed across multiple systems with time boundaries, making unified querying impossible without custom routing logic.

---

## Why This Problem Is Hard

- **Data split across systems with inconsistent boundaries** — historical and live data live in separate databases with a hard time cutoff; no unified view exists without custom routing logic.
- **No single source of truth** — the security master, prospectus documents, legacy DB, and incoming feeds can all disagree, each for legitimate reasons.
- **Prospectus documents are unstructured** — authoritative bond terms are buried in PDFs; extracting them reliably requires vector search and grounded LLM generation.
- **Reconciliation requires contextual judgment** — knowing *which* value is correct for a given field demands reasoning about source reliability, not just flagging differences.
- **High risk of incorrect overrides without auditability** — automated decisions over financial master data must be human-approved and immutably logged.

---

## Design Philosophy

- **Deterministic where possible** — routing, comparison, and field matching are pure Python with zero LLM cost and fully predictable behaviour.
- **AI where necessary** — interpretation, source arbitration, and natural language explanation are delegated to the LLM only when rule-based logic cannot reason about context.
- **Human where required** — all agent recommendations require explicit operator approval before any master data is mutated; every decision is immutably logged.

---

## Data Architecture

```
Bloomberg Terminal
      ↓
Historical Data Pull
      ↓
Cleanse & Deduplicate (upstream — before platform)
      ↓                             ↓
Legacy MongoDB               Current MongoDB
(pre 2026-01-01)             (from 2026-01-01)
      ↓                             ↓
      └────────── TemporalRouter ───┘
                       ↓
              Unified API Response
                       ↓
              (Day 4: Agent Reconciliation)
                       ↓
              (Day 5: Immutable Audit Trail)
```

**Data integrity note:** Deduplication happens upstream during the Bloomberg data pull — before data enters this platform. Legacy MongoDB is guaranteed clean with no overlap against Current MongoDB.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| API Framework | **FastAPI** | Async REST API with auto-generated docs |
| DB Driver | **Motor** | Async MongoDB driver for Python |
| Data Models | **Pydantic v2** | Validation and serialisation |
| Parallel Queries | **asyncio** | Both DBs queried simultaneously when needed |
| Databases | **MongoDB × 2** | Legacy (port 27017) and Current (port 27018) |
| Containerisation | **Docker Compose** | Spins up full infrastructure instantly |
| Testing | **pytest + pytest-asyncio** | Async test support |
| Server | **uvicorn** | ASGI server for FastAPI |
| Language | **Python 3.11** | |
| Blockchain (optional) | **Web3.py** | Hash immutability on Polygon/Ethereum |

---

## Release Roadmap

| Release | Capability | AI Involvement |
|---------|-----------|----------------|
| Day 1 ✅ | Structured API — `GET /api/v1/bonds` — explicit ISIN + date range, temporal routing across dual MongoDB | None — pure intelligent routing |
| Day 2 ✅ | NLP Query — `POST /api/v1/query` — free-text query, LLM extracts ISIN + date range | LLM extracts structured parameters from natural language |
| Day 3 ✅ | NLP + RAG — `POST /api/v3/validate/{isin}` — answers grounded in bond prospectus PDFs, mismatch detection vs security master | RAG over prospectus PDFs — vector retrieval + LLM grounding |
| Day 4 ✅ | Event-Driven Agentic Reconciliation — `POST /api/v4/ingest` — streaming file ingestion, per-record mismatch detection, concurrent AI agent instances (Plan→Execute→Validate→Resolve), human-in-the-loop approval | 4-phase agent reasoning with LLM tool-calling |
| Day 5 ✅ | **Tier 1 Complete** — Immutable Audit Trail & Compliance — `GET /api/v5/audit-trail` — every data override cryptographically hashed, immutable logging, regulatory compliance | None — pure data integrity + hashing |

**Day 5 Status:** ✅ **Tier 1 Complete** (Immutable audit trail + SHA-256 hashing + compliance queries)  
**Day 5 Roadmap:** ⏳ Tier 2 (Blockchain backing) — ⏳ Tier 3 (Compliance dashboard)

---

## Quick Start

### Prerequisites
- Docker and Docker Compose installed

### Run with Docker Compose

```bash
# Clone the repo
git clone https://github.com/ashish620/bond-data-intelligence-platform.git
cd bond-data-intelligence-platform

# Copy environment variables
cp .env.example .env

# Start all services (MongoDB ×2 + seed data + API)
docker compose up --build
```

The API will be available at **http://localhost:8000**.

Interactive API docs (Swagger UI): **http://localhost:8000/docs**

---

## API Documentation

### `GET /api/v1/bonds`

Query bond snapshots by ISIN and date range.

**Query parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `isin` | ✅ | — | Bond ISIN identifier e.g. `XS1234567890` |
| `from_date` | ✅ | — | Start date `YYYY-MM-DD` |
| `to_date` | ✅ | — | End date `YYYY-MM-DD` |
| `page` | ❌ | `1` | Page number |
| `page_size` | ❌ | `20` | Results per page (max 100) |

**Example — Legacy DB only (pre-2026):**

```bash
curl "http://localhost:8000/api/v1/bonds?isin=XS1234567890&from_date=2025-01-01&to_date=2025-12-31"
```

```json
{
  "data": [
    {
      "isin": "XS1234567890",
      "snapshot_date": "2025-01-15",
      "issuer_name": "Acme Corp",
      "maturity_date": "2030-01-15",
      "coupon_rate": 3.5,
      "currency": "USD",
      "face_value": 1000.0,
      "source": "legacy"
    }
  ],
  "total": 3,
  "page": 1,
  "page_size": 20,
  "sources": "legacy"
}
```

**Example — Both DBs (query spans 2026-01-01):**

```bash
curl "http://localhost:8000/api/v1/bonds?isin=XS1234567890&from_date=2025-06-01&to_date=2026-06-30"
```

```json
{
  "data": [...],
  "total": 4,
  "page": 1,
  "page_size": 20,
  "sources": "both"
}
```

### `POST /api/v1/query`

Send a free-text natural language query. The LLM extracts the ISIN and date range, then routes to the appropriate MongoDB instance(s).

**Example:**

```bash
curl -X POST "http://localhost:8000/api/v1/query" \
  -H "Content-Type: application/json" \
  -d '{"query": "Show me XS1234567890 bond data from January to June 2025"}'
```

```json
{
  "extracted_isin": "XS1234567890",
  "extracted_from_date": "2025-01-01",
  "extracted_to_date": "2025-06-30",
  "natural_query": "Show me XS1234567890 bond data from January to June 2025",
  "sources": "legacy",
  "data": [
    {
      "isin": "XS1234567890",
      "snapshot_date": "2025-01-15",
      "issuer_name": "Acme Corp",
      "maturity_date": "2030-01-15",
      "coupon_rate": 3.5,
      "currency": "USD",
      "face_value": 1000.0,
      "source": "legacy"
    }
  ]
}
```

---

## Day 4 — Agentic Reconciliation (`/api/v4`)

### The Problem It Solves

Operations teams receive daily feeds of security records from counterparties, custodians, or upstream systems. Reconciling each incoming record against the internal security master is tedious, error-prone, and requires expert judgment.

### Architecture

- **Event-driven, not batch** — records are streamed one-by-one through a `FileIngestor`; each mismatch immediately publishes a `ReconciliationEvent` to an async `EventBus`.
- **One agent class, one per event** — a `ReconciliationAgent` is spawned per event; the event bus processes events sequentially, but within each agent run, source fetches (Phase 2) and field assessments (Phase 3) run concurrently.
- **4-phase reasoning** — each agent reasons through Plan → Execute → Validate → Resolve, using tool-calls against legacy DB, current DB, and the prospectus RAG index.
- **Human-in-the-loop** — agent findings are stored as PENDING; a human operator calls `/api/v4/decide` to APPROVE or REJECT.
- **Immutable audit trail** — every decision (with timestamp, operator, and notes) is written to an append-only MongoDB collection and exposed via `/api/v5/audit-trail`.

---

## Day 5 — Immutable Audit Trail & Compliance (`/api/v5`)

**See:** [`audit/README.md`](audit/README.md) for full implementation details.

### The Problem It Solves

Every data override in Day 4 must be immutably logged for regulatory compliance (SEC Rule 17a-4, MiFID II, FINRA, SOX). Operations teams currently document these in Confluence (editable!) — which is a compliance violation.

### Tier 1 Solution (✅ Complete)

Day 5 Tier 1 replaces Confluence with:
- ✅ Immutable, append-only audit trail (MongoDB)
- ✅ SHA-256 hashing for data integrity
- ✅ User attribution + timestamps
- ✅ Reason tracking (why was this override approved?)
- ✅ `/api/v5/audit-trail` for querying audit entries
- ✅ `/api/v5/compliance-report` for auditor-ready reports
- ✅ `/api/v5/audit-trail/verify` for integrity verification

### Why Confluence Is Not Enough

| Aspect | Confluence | Day 5 |
|--------|-----------|-------|
| Can be edited? | ✅ Yes (compliance violation) | ❌ No (immutable) |
| Cryptographic proof? | ❌ No | ✅ Yes (SHA-256) |
| Regulatory admissible? | ⚠️ Weak | ✅ Yes |
| Audit query capability? | ❌ Manual search | ✅ `/api/v5/audit-trail` |

### API Endpoints (Day 5 — Tier 1)

#### `GET /api/v5/audit-trail`
Query the immutable audit trail with optional filters.

**Parameters:** `isin`, `field_name`, `user_id`, `from_date`, `to_date`, `reason_contains`, `limit`, `offset`

**Example:**
```bash
curl "http://localhost:8000/api/v5/audit-trail?isin=XS1234567890&limit=50"
```

#### `POST /api/v5/audit-trail/verify`
Verify that an audit entry hasn't been tampered with.

**Example:**
```bash
curl -X POST "http://localhost:8000/api/v5/audit-trail/verify?audit_id=550e8400-e29b-41d4-a716-446655440000"
```

#### `GET /api/v5/compliance-report/{isin}`
Generate compliance report for auditors.

**Example:**
```bash
curl "http://localhost:8000/api/v5/compliance-report/XS1234567890?from_date=2026-01-01&to_date=2026-04-24"
```

---

## Running Tests

```bash
# Install dependencies
pip install -r requirements.txt

# Run all tests
pytest

# Run with verbose output
pytest -v

# Run specific test file
pytest tests/test_router.py -v
pytest tests/test_api.py -v
pytest tests/test_nlp.py -v
pytest tests/test_rag.py -v
pytest tests/test_reconciliation_comparator.py -v
pytest tests/test_audit.py -v
```

Tests do **not** require running MongoDB instances — all DB interactions are mocked.

---

## Project Structure

```
bond-data-intelligence-platform/
│
├── app/
│   ├── main.py              ← FastAPI app entry point
│   ├── config.py            ← CUTOFF_DATE and DB settings
│   ├── router.py            ← TemporalRouter core engine
│   ├── models.py            ← Bond data models (Pydantic v2)
│   └── api/
│       └── bonds.py         ← API endpoints (Day 1 + Day 2)
│
├── rag/                     ← RAG prospectus validation
│   ├── ingestion/           ← PDF ingestion + ChromaDB
│   ├── rag/                 ← RAG query engine
│   └── api/                 ← POST /api/v3/validate/{isin}
│
├── reconciliation/          ← Event-driven agentic reconciliation
│   ├── agent/               ← ReconciliationAgent (4-phase)
│   ├── pipeline/            ← EventBus, FileIngestor, Comparator
│   ├── store/               ← MasterStore + DecisionStore (MongoDB)
│   └── api/                 ← /api/v4 endpoints
│
├── audit/                   ← Immutable audit trail & compliance
│   ├── __init__.py
│   ├── README.md            ← Implementation guide
│   ├── models.py            ← Pydantic models
│   ├── store/
│   │   ├── __init__.py
│   │   └── audit_store.py   ← MongoDB persistence
│   ├── api/
│   │   ├── __init__.py
│   │   └── audit.py         ← /api/v5 endpoints
│   └── blockchain/          ← (Tier 2 — future)
│
├── docs/
│   └── AUDIT_TRAIL_JUSTIFICATION.md ← Why immutable audit trails are essential
│
├── tests/
│   ├── test_router.py       ← Routing logic and boundary condition tests
│   ├── test_api.py          ← API endpoint and pagination tests
│   ├── test_nlp.py          ← NLP extractor tests
│   ├── test_rag.py          ← RAG pipeline tests
│   ├── test_reconciliation_comparator.py ← Day 4 comparator unit tests
│   └── test_audit.py        ← Day 5 audit trail tests
│
├── seed/
│   └── seed_data.py         ← Seed script for both MongoDB containers
│
├── .github/
│   └── workflows/
│       └── tests.yml        ← CI/CD pipeline (auto-test on push)
│
├── CONTRIBUTING.md          ← Contribution guidelines
├── docker-compose.yml       ← Two MongoDB containers + seed service + API
├── Dockerfile
├── requirements.txt
├── requirements.lock        ← Pinned exact versions for production
├── .env.example
└── README.md
```

---

## Compliance & Regulatory Context

This platform is designed to meet:
- **MiFID II Art. 24** — Immutable trading & data records
- **SEC Rule 17a-4** — Audit trails for financial data mutations
- **SOX Section 302/404** — Data integrity certification
- **FINRA 4511(c)** — Immutable original entry records

**See:** [`docs/AUDIT_TRAIL_JUSTIFICATION.md`](docs/AUDIT_TRAIL_JUSTIFICATION.md)

---

## Next Steps (Day 5 Future Enhancements)

### Tier 2 — Blockchain Backing (⏳ Future)
- [ ] Write hashes to Polygon smart contract for crypto-proof
- [ ] Handle async retries and confirmation tracking
- [ ] Add `GET /api/v5/blockchain-status/{audit_id}` endpoint

### Tier 3 — Compliance Dashboard (⏳ Future)
- [ ] React frontend for audit trail queries
- [ ] Visual compliance report generation
- [ ] Blockchain verification status display

---

## License

MIT
