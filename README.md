# Brand Guardian AI — Compliance QA Pipeline

An AI-powered video compliance auditing pipeline that ingests YouTube videos, extracts their spoken and on-screen content, and audits them against regulatory rulebooks (FTC influencer guidelines, YouTube ad specs) using Retrieval-Augmented Generation (RAG).

The system answers a single question for marketing, legal, and brand teams: **"Does this video violate any brand or regulatory rules — and if so, where and how?"**

---

## The Problem

Brands, agencies, and creators publish thousands of video assets every month. Each one is subject to a growing thicket of rules:

- **FTC disclosure requirements** for sponsored/influencer content
- **Platform-specific advertising policies** (YouTube ad specs, claim restrictions)
- **Brand-specific claim guidelines** (no absolute guarantees, required disclaimers, etc.)

Manually reviewing every video against every rulebook is slow, expensive, and inconsistent. A human reviewer must:

1. Watch the entire video
2. Transcribe the spoken claims
3. Read the on-screen text
4. Cross-reference each claim against multiple PDFs of regulations
5. Write up a report with violations, severity, and citations

**Brand Guardian automates this end-to-end.** Given a YouTube URL, it returns a structured compliance report — flagged violations, severity ratings, and a natural-language summary — in a single API call.

---

## What It Does

Submit a YouTube URL → the system:

1. **Downloads** the video locally using `yt-dlp`
2. **Uploads** it to Azure Video Indexer
3. **Extracts** the full transcript (speech-to-text) and on-screen text (OCR)
4. **Retrieves** the most relevant regulatory rules from a vectorized knowledge base built from regulatory PDFs
5. **Audits** the transcript + OCR against the retrieved rules using an Azure OpenAI chat model
6. **Returns** a structured JSON report:
   - `status`: `PASS` or `FAIL`
   - `compliance_results`: list of violations with `category`, `severity`, and `description`
   - `final_report`: human-readable summary

---

## Architecture

The pipeline is a two-node LangGraph DAG. State flows linearly from ingestion → auditing.

```
            ┌──────────────┐         ┌──────────────┐
   START ─► │   Indexer    │ ──────► │   Auditor    │ ──► END
            │  (Node 1)    │         │  (Node 2)    │
            └──────────────┘         └──────────────┘
                  │                         │
                  ▼                         ▼
        ┌─────────────────────┐   ┌──────────────────────┐
        │  yt-dlp             │   │  Azure AI Search     │
        │  Azure Video        │   │  (vector retrieval)  │
        │  Indexer (STT+OCR)  │   │  Azure OpenAI Chat   │
        └─────────────────────┘   └──────────────────────┘
```

### Node 1 — Indexer ([nodes.py:23](backend/src/graph/nodes.py#L23))

Handles ingestion and content extraction.

- Downloads the YouTube video to a local temp file via `yt-dlp` (with Android client and custom user-agent to avoid bot detection)
- Authenticates with Azure using `DefaultAzureCredential` and exchanges an ARM token for a Video Indexer account token
- Uploads the local file to Azure Video Indexer
- Polls Azure every 30s until processing reaches `Processed` (or fails with `Failed` / `Quarantined`)
- Parses the raw VI JSON into a normalized state object: `transcript` (joined string), `ocr_text` (list of strings), `video_metadata`

The ingestion logic lives in [VideoIndexerService](backend/src/services/video_indexer.py).

### Node 2 — Auditor ([nodes.py:70](backend/src/graph/nodes.py#L70))

Performs RAG-based compliance reasoning.

- Builds a query from `transcript + ocr_text`
- Issues a `similarity_search(k=3)` against the Azure AI Search vector store to retrieve the top-3 most relevant rule chunks
- Constructs a system prompt that embeds the retrieved rules and a strict JSON output schema
- Calls Azure OpenAI Chat (`temperature=0.0` for deterministic auditing)
- Strips markdown code-fences if the model wraps its JSON, parses it, and writes `compliance_results`, `final_status`, and `final_report` back into the graph state

### Graph State ([state.py](backend/src/graph/state.py))

The shared state object that flows through nodes:

| Field | Purpose |
|---|---|
| `video_url`, `video_id` | Inputs |
| `transcript`, `ocr_text`, `video_metadata` | Populated by Indexer |
| `compliance_results` | Append-only list of `ComplianceIssue` dicts (LangGraph `operator.add`) |
| `final_status`, `final_report` | Populated by Auditor |
| `errors` | Append-only error log for observability |

---

## The Knowledge Base

The audit is only as good as the rules it retrieves. The knowledge base is built offline via a separate ETL script ([index_documents.py](backend/scripts/index_documents.py)) that runs once (or whenever rulebooks change).

**ETL flow:**

1. **Extract** — Reads PDFs from [backend/data/](backend/data/) using `PyPDFLoader`. Currently indexed:
   - `1001a-influencer-guide-508_1.pdf` (FTC Influencer Disclosure Guide)
   - `youtube-ad-specs.pdf` (YouTube advertising specifications)
2. **Transform** — Chunks each PDF into 1000-character segments with 200-character overlap (`RecursiveCharacterTextSplitter`), preserving context across cut points. Each chunk is tagged with its source filename for traceability.
3. **Load** — Embeds chunks using Azure OpenAI `text-embedding-3-small` and writes them to Azure AI Search.

The current index holds ~37 chunks across both PDFs. Adding a new rulebook is as simple as dropping the PDF into [backend/data/](backend/data/) and re-running the script.

---

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | **LangGraph** | Stateful DAG with typed schema; easy to extend with new nodes (e.g., a human-review step) |
| LLM | **Azure OpenAI** (chat + `text-embedding-3-small`) | Enterprise compliance, regional data residency |
| Vector store | **Azure AI Search** | Native managed service; integrates with LangChain via `AzureSearch` |
| Video AI | **Azure Video Indexer** | Provides both speech-to-text and OCR in a single call |
| Video download | **yt-dlp** | Robust YouTube extraction with anti-bot workarounds |
| API | **FastAPI + Uvicorn** | Async, auto-generated OpenAPI docs |
| Observability | **Azure Monitor / OpenTelemetry** + **LangSmith** | Auto-instrumented FastAPI request tracing; LLM trace inspection |
| Package mgmt | **uv** | Fast, deterministic dependency resolution |

---

## Project Structure

```
ComplianceQAPipeline/
├── main.py                          # CLI entry point — runs a single audit and prints the report
├── pyproject.toml                   # Dependencies (managed by uv)
├── .env.example                     # Required environment variables
└── backend/
    ├── data/                        # Source PDFs that form the knowledge base
    │   ├── 1001a-influencer-guide-508_1.pdf
    │   └── youtube-ad-specs.pdf
    ├── scripts/
    │   └── index_documents.py       # One-time ETL: PDFs → embeddings → Azure AI Search
    └── src/
        ├── api/
        │   ├── server.py            # FastAPI app exposing POST /audit and GET /health
        │   └── telemetry.py         # Azure Monitor OpenTelemetry setup
        ├── graph/
        │   ├── workflow.py          # LangGraph DAG: indexer → auditor
        │   ├── nodes.py             # Indexer and Auditor node implementations
        │   └── state.py             # VideoAuditState TypedDict schema
        └── services/
            └── video_indexer.py     # Azure Video Indexer client + yt-dlp wrapper
```

---

## Getting Started

### Prerequisites

- Python 3.12+
- `uv` package manager
- An Azure subscription with the following resources provisioned:
  - Azure OpenAI (with a chat deployment + `text-embedding-3-small` deployment)
  - Azure AI Search instance with an index created
  - Azure Video Indexer account (ARM-based)
  - (Optional) Azure Monitor / Application Insights for telemetry

### 1. Install dependencies

```bash
uv sync
```

### 2. Configure environment

Copy [.env.example](.env.example) to `.env` and fill in your Azure credentials:

```bash
cp .env.example .env
```

Required variables include `AZURE_OPENAI_*`, `AZURE_SEARCH_*`, `AZURE_VI_*`, and `AZURE_SUBSCRIPTION_ID` / `AZURE_RESOURCE_GROUP` for Video Indexer ARM auth.

### 3. Build the knowledge base (one-time)

Drop your regulatory PDFs into [backend/data/](backend/data/), then run:

```bash
uv run python backend/scripts/index_documents.py
```

This chunks, embeds, and uploads the PDFs to Azure AI Search. Re-run whenever rulebooks change.

### 4. Run a single audit (CLI)

```bash
uv run python main.py
```

This audits the hard-coded test URL in [main.py:58](main.py#L58) and prints the report to stdout. Replace the URL to audit a different video.

### 5. Run as an API

```bash
uv run uvicorn backend.src.api.server:app --reload
```

Then:

- **Interactive docs:** http://localhost:8000/docs
- **Health check:** `GET http://localhost:8000/health`
- **Audit endpoint:** `POST http://localhost:8000/audit`

Example request:

```bash
curl -X POST http://localhost:8000/audit \
  -H "Content-Type: application/json" \
  -d '{"video_url": "https://youtu.be/dT7S75eYhcQ"}'
```

Example response:

```json
{
  "session_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "video_id": "vid_ce6c43bb",
  "status": "FAIL",
  "final_report": "Video contains 2 critical violations related to undisclosed sponsorship and absolute claims.",
  "compliance_results": [
    {
      "category": "FTC Disclosure",
      "severity": "CRITICAL",
      "description": "Sponsored content disclosure is missing at the start of the video."
    },
    {
      "category": "Misleading Claims",
      "severity": "CRITICAL",
      "description": "Use of an absolute guarantee ('100% effective') without supporting evidence."
    }
  ]
}
```

---

## Observability

Two layers of tracing are wired in:

- **Azure Monitor / OpenTelemetry** ([telemetry.py](backend/src/api/telemetry.py)) — auto-instruments every FastAPI request, captures response times, error rates, and downstream Azure dependency calls. Activated only if `APPLICATIONINSIGHTS_CONNECTION_STRING` is set.
- **LangSmith** — set `LANGCHAIN_TRACING_V2=true` and provide `LANGCHAIN_API_KEY` to capture every LLM call, prompt, and graph node transition in the LangSmith UI.
---

## Roadmap / Where to Extend

The two-node graph is intentionally minimal. Natural next nodes:

- **Human-in-the-loop** — add a conditional edge that routes ambiguous `WARNING`-severity findings to a human reviewer.
- **Citation tracking** — surface which retrieved chunk justifies each violation, so reviewers can audit the auditor.