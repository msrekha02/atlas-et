# ATLAS — Industrial Knowledge Intelligence Platform

> One queryable brain over every drawing, work order, procedure, inspection and incident in an industrial plant — turning fragmented documents into connected, actionable, continuously-updated intelligence at the point of need.

Built for the **Industrial Knowledge Intelligence** challenge: asset-intensive plants run across 7–12 disconnected document systems, ~35% of working hours are lost searching for information, and a retiring generation is taking decades of undocumented knowledge with them. ATLAS attacks all three at once.

---

## What it does (mapped to the challenge)

| Challenge area | ATLAS module | Where to see it |
|---|---|---|
| **Universal Ingestion & Knowledge Graph** | **Real PDF / CSV / Excel / email / text parsing + OCR for scanned forms & image-only PDFs** → entity extraction → one unified graph that re-links automatically on every new file | **Knowledge Graph** page + live multi-format upload on **Documents** |
| **Expert Knowledge Copilot** | Hybrid RAG (BM25 + graph expansion) with **source citations, confidence scores, and links to originating documents** — **fully responsive and installable on a field technician's phone** (PWA manifest, add-to-home-screen) | **Knowledge Copilot** page |
| **Maintenance Intelligence & RCA** | Fuses work orders + failures + OEM manuals + inspections + a **live real-time operating-conditions feed** → recurring-failure RCA and an **optimised, risk-ranked PM schedule** | **Maintenance Intelligence** page (Assets · **Live Conditions** · **PM Schedule** tabs) |
| **Quality & Regulatory Compliance** | Maps OISD / Factories Act / PESO / environmental requirements → gap detection + **one-click auto-generated audit evidence package**, plus a **quality/process-deviation flagging** engine | **Compliance** page (Requirements · **Quality Deviations** tabs) |
| **Lessons Learned & Failure Intelligence** | Dedicated engine: fleet-wide failure patterns, systemic themes, **reference library of industry failure signatures**, and proactive warnings — all **routed to responsible teams as acknowledgeable alerts, including real outbound webhook delivery** | **Lessons Learned** + **Alerts** (push-to-teams) pages |
| **Agentic AI for maintenance & compliance** | A real LLM tool-calling loop (Google Gemini) that **plans for itself** which of six live engines to call (compliance, asset health, ROI, PM schedule, fleet patterns, document search) and in what order, then answers from what it found — distinct from the Copilot's fixed retrieve-then-generate pipeline | **Planning Agent** page |

### The "connect the dots no one team can" moment
The seed corpus hides a real causal chain across **six document types**: a 2024 crude-slate change email → skipped Management-of-Change → two repeat seal failures → a near-miss → the same wax pattern later found on the standby pump → a retiring engineer's handover note that explains all of it. Ask the Copilot *"Why does P-101A keep failing?"* and it reaches the OEM manual and the handover memo **even though they never share a keyword with the question** — because the knowledge graph links them through the shared failure mode.

---

## Architecture at a glance

```
                    ┌───────────────── React SPA (Vite + Tailwind) ──────────────────┐
                    │  Overview · Copilot · Graph · Maintenance · Compliance · Docs   │
                    └───────────────────────┬─────────────────────┬───────────────────┘
                                    JSON / REST          SSE (live alerts, telemetry,
                                             │             streamed Copilot answers)
        ┌────────────────────────────────────┴─────────────────────┴──────────────────────────┐
        │                       FastAPI service (auth/rate-limit/metrics middleware)            │
        ├──────────────┬──────────────┬──────────────┬──────────────────┬──────────────────────┤
        │  Ingestion    │  Entity      │  Knowledge   │  Hybrid retrieval │  Compliance &        │
        │  + bounded/   │  extraction  │  graph       │  BM25 + FAISS-HNSW│  Maintenance         │
        │  overlapping  │  (rule-based │  (adjacency, │  ANN semantic +   │  intelligence        │
        │  chunking     │  + typed     │  typed cause │  graph + lexical/ │  engines (date-      │
        │               │  cause-rels) │  edges)      │  positional rerank│  driven, cited)      │
        └───────┬───────┴──────────────┴──────┬───────┴─────────┬────────┴──────────────────────┘
                │                              │                 │
          Document corpus            In-memory graph      Inverted index + FAISS index
          (18 seed docs, 10 types)   (V=57, E=186)         (disk-cached warm start)
```

---

## Quick start

Two processes: a Python API and a Vite dev server.

### 1. Backend (FastAPI)
```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate     macOS/Linux: source .venv/bin/activate
.venv/Scripts/python -m pip install -r requirements.txt
.venv/Scripts/python -m uvicorn app.main:app --port 8100
```
The API comes up on `http://127.0.0.1:8100`. It reads the corpus in `backend/data/corpus/` on startup and builds the graph + indexes in-memory (~100 ms, including the LSA/SVD fit — see `docs/ARCHITECTURE.md` §6 for the up-to-date breakdown at scale).

`opencv-python-headless` and `Pillow` (P&ID vision) and `rapidocr-onnxruntime` (OCR) are all in `requirements.txt`. OCR is the one genuinely optional piece — everything else degrades cleanly (`ocr_available: false` in `/api/health`) if it's ever removed, but vision/CV is required for the Drawings page to function.

**Run the test suite:**
```bash
.venv/Scripts/python -m pip install -r requirements-dev.txt
.venv/Scripts/python -m pytest tests/ -v
```

**Before exposing this beyond localhost**, set at minimum:
```bash
export ATLAS_API_KEY=some-shared-secret        # required to POST /api/ingest or ack/unack alerts
export ATLAS_CORS_ORIGINS=https://your-frontend.example.com
```
Without any API key set, ingestion and alert-acknowledgement are open to anyone who can reach the API — fine for a local demo, not for a shared deployment. `ATLAS_MAX_UPLOAD_MB` (default 15) and `ATLAS_INGEST_RATE_PER_MIN` (default 12) are also configurable. `GET /api/health` reports whether auth is enabled, and `GET /api/metrics` exposes Prometheus-format counters (request counts/latency by route, corpus size, compliance score).

**Role-based access** — two roles, viewer (read) and operator (write), via named keys:
```bash
export ATLAS_API_KEYS="reader-key:viewer,writer-key:operator"
export ATLAS_REQUIRE_READ_AUTH=true   # optional — also locks down GET endpoints, not just writes
```

**Audit trail** — give a key a display name (`key:role:name`) and every mutating action it takes is attributed to that name in `GET /api/audit` (operator-only) instead of a masked key prefix:
```bash
export ATLAS_API_KEYS="writer-key:operator:jsmith,reader-key:viewer"
```
This is a named-key audit log, not a full per-user identity/SSO system — keys are still shared secrets per role, so it answers "which named key did this," not "prove which human held it." Covers ingest, vision-parse absorb, and alert ack/unack/webhook-test; persisted append-only to `data/audit_log.jsonl`.

**Alert webhook delivery** — the "push to teams" queue is real routing + acknowledgement, but delivery *outside* the app is opt-in:
```bash
export ATLAS_ALERT_WEBHOOK_URL=https://hooks.slack.com/services/...   # any Slack-compatible incoming webhook
```
Without this set, `GET /api/health` reports `alert_webhook_configured: false` and the Alerts page shows "Webhook not configured" — an honest no-op, not a silent failure. With it set, every new active alert is POSTed once (disk-persisted dedup, so restarts and repeated polling never re-send) as `{"text": "..."}`; `POST /api/alerts/webhook-test` (operator key required) sends one synthetic payload on demand to confirm connectivity, and the Alerts page exposes a "Send test" button that calls it.

### Docker
```bash
docker compose up --build
```
Backend on `:8100`, frontend (nginx, proxying `/api` to the backend, SSE-aware) on `:8080`. A named volume persists the semantic-index disk cache and the alert-acknowledgement store across restarts.

### 2. Frontend (React)
```bash
cd frontend
npm install
npm run dev
```
Open the URL Vite prints (e.g. `http://localhost:5173`). The dev server proxies `/api` to port 8100 — no CORS setup needed.

> **Note on ports:** the demo backend runs on **8100** (port 8000 was occupied on the build machine). If you change it, update the proxy target in `frontend/vite.config.js`.

**Run the frontend test suite** (Vitest + React Testing Library):
```bash
npm run lint    # no-undef — the specific check that would have caught the Alerts.jsx bug below
npm run test    # component tests: ErrorBoundary, Alerts, Agent, Copilot, Dashboard
```
Both run in CI on every push/PR (`.github/workflows/ci.yml`). `Alerts.test.jsx` is a real regression test: this page previously referenced an undefined `load` identifier in its Refresh button, which threw on first render and — with no error boundary anywhere in the app — blanked the entire SPA. Both the bug and the missing boundary are fixed (`components/ui.jsx`'s `ErrorBoundary`, wrapped around the router in `App.jsx`); `ErrorBoundary.test.jsx` and `Alerts.test.jsx` are what stand between a regression like that and it shipping again.

### 3. Optional — LLM-synthesized answers (pluggable provider; free via Groq)
The Copilot works fully offline in **extractive mode**. To upgrade answer synthesis to an LLM while keeping identical retrieval + citation grounding, set one API key. The provider is auto-detected from the key prefix and sits behind a small adapter (`app/llm.py`) — swapping providers is configuration, not a code change.

```bash
export GROQ_API_KEY=gsk_...     # PowerShell: $env:GROQ_API_KEY="gsk_..."
# (optional) pick a model — defaults to llama-3.3-70b-versatile:
#   export LLM_MODEL=llama-3.1-8b-instant
# restart the backend
```

The Copilot header flips to "AI synthesis on" automatically. **Retrieval and citations are identical in both modes** — only the prose composition changes, so the demo is honest either way.

The Copilot streams — `GET /api/ask/stream` (Server-Sent Events) delivers the model's answer token-by-token as it's generated. In extractive mode (no LLM configured, or the retrieval isn't grounded), the answer is already computed instantly, so it arrives as a single event rather than a fake incremental replay of a complete string — the honesty of "this mode doesn't hallucinate a stream" matters as much as "this mode doesn't hallucinate an answer."

### 4. Optional — the Planning Agent
The same key also enables `POST /api/agent/ask` and the **Planning Agent** page — a genuine tool-calling loop (see `app/agent.py`) where the model itself decides which of `get_compliance_gaps` / `get_asset_health` / `search_documents` / `get_fleet_patterns` / `get_roi_summary` / `get_pm_schedule` to call, how many times, and in what order, before answering. Unlike the Copilot, there is **no offline fallback** for this one — planning across signals has no honest non-LLM equivalent, so without a key the page reports itself unavailable rather than faking a plan. `GET /api/health` exposes `agent_available` so the frontend never has to guess.

---

## Try these queries in the Copilot
- `Why does P-101A keep failing?` — recurring seal failure, root-caused to the crude-slate change
- `Which relief valves are overdue for testing?` — pulls the overdue PSV-1104 from the inspection register
- `What are the pre-start checks for the crude charge pumps?` — cites SOP-012
- `What did the crude slate change cause?` — traces the email → incident chain

## Try these in the Planning Agent
- `What should Rotating Equipment prioritise this week?` — calls `get_asset_health` and `get_compliance_gaps`, cross-references them
- `Where is the biggest avoidable-cost opportunity right now, and what would close it?` — calls `get_roi_summary`, then `get_pm_schedule` for the fix
- `Which asset has the worst combination of health and open compliance gaps?` — needs both engines before it can answer, so the trace shows two tool calls in sequence

Open the **plan trace** on any answer to see exactly which tools it called, in what order, and what each one returned — the same explainability standard the Copilot's retrieval trace already holds itself to.

## Live-ingest demo
On the **Documents** page, drop a `.md`/`.txt` file. Watch the entity extraction, graph node/edge counts, and re-index time update in <50 ms — then re-run a Copilot query and see the new document participate immediately. **No re-training, no re-embedding.**

---

## Repository layout
```
atlas/
├── backend/
│   ├── app/
│   │   ├── main.py          # FastAPI routes, in-memory state, auth/RBAC/rate-limit/metrics middleware
│   │   ├── formats.py       # multi-format readers (PDF, CSV, XLSX, .eml, image, text)
│   │   ├── ocr.py           # OCR for scanned forms / image-only PDFs (RapidOCR, isolated subprocess)
│   │   ├── ocr_worker.py    # the isolated OCR subprocess entrypoint
│   │   ├── vision.py        # classical-CV P&ID/drawing digitisation (OpenCV), feeds graph.py
│   │   ├── sample_pid.py    # generates the sample P&ID used by the Drawings demo
│   │   ├── ingest.py        # frontmatter parsing, bounded/overlapping chunking
│   │   ├── extract.py       # rule-based entity extraction
│   │   ├── graph.py         # knowledge graph — mention/co-occurrence edges + typed caused_by/
│   │   │                    #   root_cause_condition edges; add_document() folds in one new doc
│   │   │                    #   incrementally, no existing document re-processed (ARCHITECTURE.md §12b)
│   │   ├── search.py        # BM25 + FAISS-ANN semantic + graph, plus a lexical/positional reranker;
│   │   │                    #   add_document() extends the live index without a full rebuild
│   │   ├── embeddings.py    # LSA (TF-IDF → SVD) served through a FAISS HNSW index, disk-cached
│   │   ├── rag.py           # extractive + optional LLM synthesis (blocking + streaming), confidence gating
│   │   ├── llm.py           # provider adapter (Google Gemini free tier) behind a minimal Messages-shaped seam
│   │   ├── agent.py         # real LLM tool-calling loop over the read-only engines below — genuine
│   │   │                    #   planning, distinct from rag.py's fixed retrieve-then-generate pipeline
│   │   ├── compliance.py    # date-driven regulatory gap engine (pattern-based, not doc-id-bound)
│   │   ├── evidence.py      # printable audit evidence-pack generator
│   │   ├── qms.py           # QMS non-conformance-record export contract
│   │   ├── ontology.py      # ISO 14224 / ISA-95 equipment classification
│   │   ├── quality.py       # quality / process-deviation flagging
│   │   ├── maintenance.py   # asset health, MTBF, RCA (quotes cause from source text), recommendations
│   │   ├── telemetry.py     # real-time operating-conditions simulator
│   │   ├── schedule.py      # optimised, risk-ranked PM schedule
│   │   ├── lessons.py       # fleet-wide failure patterns + proactive warnings (data-derived, not authored)
│   │   ├── alerts.py        # alert aggregation + team routing + disk-persisted ack + real webhook delivery
│   │   ├── roi.py           # avoidable-downtime / cost (business impact)
│   │   ├── evaluation.py    # gold-set retrieval/extraction accuracy harness — tuning set + a held-out split
│   │   └── bench.py         # synthetic-scale benchmark (scalability evidence)
│   ├── tests/                # pytest suite — ingestion, extraction, graph, RAG, streaming, compliance, API
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── requirements-dev.txt  # + pytest/httpx for running tests
│   └── data/corpus/          # 18 cross-linked industrial documents, 10 document types
├── frontend/
│   ├── Dockerfile / nginx.conf   # multi-stage build; nginx proxies /api (SSE-aware) to the backend
│   └── src/
│       ├── pages/           # Dashboard, Copilot (streaming), Agent (tool-call trace), Graph, Documents,
│       │                    #   Assets (live SSE), Compliance, Lessons, Alerts (live SSE + webhook status), Drawings
│       └── components/      # reusable UI system (responsive, mobile drawer nav)
├── docker-compose.yml
├── .github/workflows/ci.yml # backend pytest suite + frontend build, on every push/PR
└── docs/
    ├── ARCHITECTURE.md      # system-design deep dive
```

## Design choices worth calling out
- **No black-box embeddings for the demo.** Retrieval is explainable end-to-end — you can open the "retrieval trace" on any answer and see the exact terms, matched graph entities, and which documents were surfaced *through the graph* rather than by keyword. That transparency is a feature for a safety-critical industrial buyer. The semantic signal is real ANN (FAISS HNSW), not a stand-in — see below.
- **Every number is computed from the documents, not hardcoded.** Compliance statuses come from dates parsed out of any inspection register / SOP frontmatter matching the expected shape (not one specific document id — see `compliance.py`, verified in `tests/test_compliance.py` by renaming every document id in the corpus and confirming the same findings still fire). Asset health, RCA recommendations, and fleet-wide warnings all quote a "Root cause: …" clause or a computed timeline straight out of the cited records rather than asserting one. Change a date or a finding in a source file, restart, and the whole system reflects it.
- **Grounded, not hallucinated.** In both extractive and LLM modes, answers are constrained to retrieved passages and every claim carries a citation to a real document. Confidence is gated on actual evidence — term coverage of the retrieved text, a recognised equipment/standard/failure-mode entity, and the retrieval score — not on how many citations happened to be assembled, so an out-of-corpus question reports near-zero confidence instead of a false-confident answer.
- **The semantic signal actually measures a lift.** LSA's rank was previously left uncapped relative to corpus size (k≈48 for 49 chunks — SVD had almost nothing to compress, so it contributed nothing measurable over lexical search). Capping k relative to corpus size fixed this: semantic search alone now matches the graph's contribution on keyword-poor queries (`docs/ARCHITECTURE.md` §6.2) — reported with the same honesty as when it measured zero.
- **A real ANN index, not a bigger dependency for its own sake.** The LSA embeddings are served through FAISS's `IndexHNSWFlat` — genuine approximate nearest-neighbour search, not a dense scan relabelled — with the fitted index persisted to disk and content-addressed, so a restart on an unchanged corpus warm-starts instead of re-running SVD (measured 84× faster at 2,000 chunks; `tests/test_embeddings.py`).
- **Relationships beyond co-occurrence.** Most graph edges are "these two things were mentioned in the same document." `graph.py` additionally parses "root cause / due to / caused by" clauses and links the *specific* entity named in the clause — a directed `caused_by` relation, not just proximity.
- **Real-time is actually real-time.** Alerts and live telemetry are pushed over Server-Sent Events (`/api/stream/alerts`, `/api/stream/telemetry`) — one held-open connection the server writes into — not the frontend polling on a `setInterval`. The Copilot streams token-by-token from the LLM the same way.
- **Ingest is genuinely incremental, not a full rebuild with a fast corpus.** `/api/ingest` used to re-tokenise and re-embed every existing document on every single upload — fine at 18 docs (<50ms), the actual bottleneck at scale. `graph.KnowledgeGraph.add_document()` / `search.HybridIndex.add_document()` / `embeddings.SemanticIndex.add_chunks()` now fold in one new document without re-processing any existing one — proven equivalent to a full rebuild (same nodes, same edges, same BM25 rankings) in `tests/test_incremental.py` and `tests/test_state_incremental.py`, including a real bug those tests found and fixed: an already-ingested document referencing a not-yet-ingested one by id needs its "references" edge created retroactively when that document arrives, not silently dropped. Measured live: ~9ms per ingest now, vs. ~187ms for a full rebuild of the same 18-document corpus — and unlike the old path, that number doesn't grow with corpus size for the part that dominates at scale. See `docs/ARCHITECTURE.md` §12b.
- **The agent is a genuine new layer, not the engines renamed again.** `docs/ARCHITECTURE.md` §9 explicitly renamed this codebase's "agents" to "engines" because nothing planned or acted autonomously — and said that if real orchestration were ever built, it would be "a genuine new architecture layer on top of this one, not a renaming exercise." `app/agent.py` is that layer: the LLM itself chooses which read-only tool to call and in what order, per question, over exactly the engines the rest of the app already ships and tests. See `docs/ARCHITECTURE.md` §9a.
- **"Held-out" is named as exactly that, not oversold as "independent."** The original 16-question, 7-document gold set was authored *while* the extraction/retrieval code was being built — a fitted score. `evaluation.py` now also carries an 18-question, 11-document set authored *after* that code was frozen, with nothing changed to improve it — the strongest claim of generalisation this team can make without a third-party reviewer, and it's reported as such (`/api/evaluation`'s `validation` block, and the Dashboard's Tuning/Held-out toggle) rather than blended into one flattering number. The held-out numbers are honestly lower (72% hit@1 vs. 81%) — that drop is the point of measuring it.
- **"Mobile" means installable, not just responsive.** The brief asks for something that works "on mobile for field technicians, not just desktops" — Tailwind responsive layout and the drawer nav already covered *works*; `frontend/public/manifest.json` + the `apple-mobile-web-app-*` tags in `index.html` cover *installable*, so a technician can add ATLAS to their home screen like a native app rather than bookmarking a website. No native mobile API exists, and none is claimed to — "mobile API" would overstate what's here; this is a PWA-installable responsive web app, named as exactly that.
- **Push-to-teams now actually leaves the app.** `alerts.py`'s webhook dispatcher POSTs a real Slack-compatible payload to `ATLAS_ALERT_WEBHOOK_URL` for every alert the instant it first becomes active — disk-persisted so a restart or a 3-second SSE re-poll never re-sends one already delivered. Unconfigured, it's a documented no-op (`alert_webhook_configured: false`), not a silent gap dressed up as a feature.
