# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Agentic AI system that checks draft DAF publications against DAFMAN 90-161 compliance requirements using a RAG pipeline, then routes structured findings to publications managers for human review. Python FastAPI backend with a React Base44 frontend.

## Tech Stack

- **Backend:** Python 3.11+ / FastAPI / Uvicorn
- **RAG:** LlamaIndex + ChromaDB (vector database) + OpenAI `text-embedding-3-small`
- **LLM:** OpenAI GPT-4o (structured JSON output, temperature 0.1)
- **Document Parsing:** PyMuPDF (PDF), python-docx (DOCX)
- **Report Generation:** ReportLab (PDF export)
- **Notifications:** SMTP email + Slack webhooks
- **Frontend:** React on Base44 platform (Deno serverless)
- **Deployment:** Docker + docker-compose

## Build/Dev Commands

```bash
# Backend setup
cd backend
pip install -r requirements.txt

# One-time: ingest DAFMAN 90-161 PDF into ChromaDB
python ../scripts/ingest_dafman.py [--pdf path/to/dafman.pdf] [--reset]

# Run dev server
uvicorn main:app --reload --port 8000
# Swagger docs at http://localhost:8000/docs

# Docker
docker-compose up --build
```

## API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/upload` | Upload PDF/DOCX, returns `doc_id` |
| POST | `/check/{doc_id}` | Start async compliance check, returns `check_id` |
| GET | `/findings/{check_id}` | Get structured findings |
| GET | `/findings/by-doc/{doc_id}` | Get findings by document ID |
| POST | `/review/{check_id}` | Submit review decision (APPROVE/REQUEST_REVISION/ESCALATE) |
| GET | `/report/{check_id}/pdf` | Download PDF compliance report |
| GET | `/health` | Health check |

## RAG Pipeline Architecture

```
DAFMAN 90-161 PDF → chunk by section headers → OpenAI text-embedding-3-small
  → ChromaDB (cosine similarity, collection: dafman90161)

Per rule check:
  Rule requirement → embed query → retrieve top-5 DAFMAN chunks
  → format as context → GPT-4o structured JSON evaluation
  → CheckResult {status, finding, detail, dafman_citation, recommendation, evidence}
```

**Ingestion** (`scripts/ingest_dafman.py`):
- Chunks PDF by section headers (regex: `\d+\.\d+`, `Chapter \d+`, `ATTACHMENT \d+`)
- Max 1500 chars per chunk
- Batches embeddings (100 at a time)
- Stores in ChromaDB at `rag-data/vectorstore/`

**Retrieval** (`backend/rag/retriever.py`):
- Embeds query, searches ChromaDB for top-N similar chunks
- Returns `RetrievedChunk` objects with text, section, page, similarity score
- Formats chunks as `[DAFMAN Reference N — Section: X.X, Page: N]` blocks for LLM context

## 14 Compliance Rules

Defined in `backend/agent/checklist.py`. Each rule has: `id`, `category`, `title`, `description`, `patterns` (regex for pre-check), `rag_query` (vector search query).

**Categories:** Header (HDR-001–006), Directive Language (DIR-001–002), Waiver (WAI-001–002), Records (REC-001), Structure (STR-001–002), Privacy (PRI-001–002).

Each rule is evaluated with: regex pre-check → RAG retrieval → GPT-4o analysis → structured JSON result (PASS/FAIL/WARNING/NOT_APPLICABLE).

## Project Structure

```
backend/
├── main.py                    # FastAPI app with lifespan, CORS, route registration
├── config.py                  # Pydantic settings (loads from .env)
├── requirements.txt
├── Dockerfile
├── api/routes/                # Modular route files
│   ├── upload.py              # POST /upload
│   ├── check.py               # POST /check/{doc_id} (async background task)
│   ├── findings.py            # GET /findings/*
│   ├── review.py              # POST /review/{check_id}
│   └── report.py              # GET /report/{check_id}/pdf
├── agent/
│   ├── compliance_agent.py    # Main orchestration: run_compliance_check()
│   ├── checklist.py           # 14 rule definitions with patterns + RAG queries
│   └── prompts.py             # LLM system prompts
├── rag/
│   ├── vectorstore.py         # ChromaDB client singleton
│   └── retriever.py           # Embedding + similarity search + context formatting
├── ingestion/
│   └── document_parser.py     # PDF (PyMuPDF) + DOCX (python-docx) parsing
└── review/
    ├── findings_model.py      # Pydantic models (ComplianceReport, CheckResult, etc.)
    ├── notifier.py            # Email (SMTP) + Slack webhook notifications
    └── report_generator.py    # ReportLab PDF report generation

frontend-base44/               # React Base44 frontend
├── api/backendClient.js       # HTTP client for FastAPI (uses VITE_BACKEND_URL)
├── entities/ComplianceCheck.json  # Base44 entity schema
├── pages/
│   ├── Upload.jsx             # File upload + check trigger
│   ├── Results.jsx            # Real-time polling + findings display
│   └── ReviewQueue.jsx        # Manager review interface
└── Layout.js                  # Navigation

scripts/
└── ingest_dafman.py           # One-time ChromaDB ingestion

rag-data/
├── raw/dafman90-161.pdf       # Source PDF (you must provide)
└── vectorstore/               # ChromaDB persistence (auto-created)

files/                         # Test documents (compliant + non-compliant samples)
```

## Environment Variables

**Required (.env):**
```
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
CHROMA_PERSIST_DIR=../rag-data/vectorstore
CHROMA_COLLECTION_NAME=dafman90161
SECRET_KEY=random-string
CORS_ORIGINS=http://localhost:3000,https://your-base44-app.base44.app
```

**Optional (notifications):**
```
SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASSWORD
NOTIFICATIONS_FROM, PUBLICATIONS_MANAGER_EMAIL
SLACK_WEBHOOK_URL, SLACK_CHANNEL
```

## Human Review Workflow

1. User uploads document via Base44 frontend
2. Frontend calls POST /check/{doc_id}, backend runs async compliance check
3. Frontend polls GET /findings/by-doc/{doc_id} every 5 seconds
4. On completion: email + Slack notifications sent to publications manager
5. Manager reviews in ReviewQueue, submits decision: APPROVE / REQUEST_REVISION / ESCALATE
6. All decisions logged with reviewer name, email, timestamp, notes

## Important Notes

- **In-memory storage (MVP):** `_reports` dict in `check.py` — replace with database for production
- **No auth on API endpoints (MVP)** — add JWT/OAuth2 before production use
- **ChromaDB only stores DAFMAN 90-161** (public, unclassified reference text)
- **Do not upload classified/CUI documents** to this MVP (no encryption at rest)
- Frontend Base44 pages must remain flat (no subdirectories) per Base44 platform rules
