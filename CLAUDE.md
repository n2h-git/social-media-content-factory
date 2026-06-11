# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a Windows workstation home directory containing multiple ArkCybr projects under `src/`. Each subdirectory in `src/` is an independent git repository (not submodules). All repos are under the `n2h-git` GitHub organization except `CKL_Checker` which is under `optima-gs`.

## Projects

| Project | Stack | Purpose |
|---------|-------|---------|
| `src/besure` | React 18 + Base44 (Deno serverless) | Cybersecurity education platform (courses, games, XP, multi-tenant) |
| `src/daf-rules-check` | React 18 + Base44 | DAFMAN 90-161 compliance checker with Claude AI rule evaluation |
| `src/dafman-compliance-agent-v2` | Python FastAPI + React Base44 | DAFMAN 90-161 compliance agent with RAG pipeline (LlamaIndex + ChromaDB) |
| `src/arkcybr-soc-workflows` | n8n JSON workflows | SOC threat intelligence automation (VirusTotal, Shodan, AbuseIPDB, MISP) |
| `src/social-media-content-factory` | n8n JSON workflow | AI-powered multi-platform social media posting with approval gate |
| `src/CKL_Checker` | PowerShell (Windows Forms) | STIG checklist (.ckl/.cklb) validation and repair tool |

## Build/Dev Commands

### besure (React + Base44)
```bash
cd src/besure
npm run dev           # Vite dev server
npm run build         # Production build
npm run lint          # ESLint check
npm run lint:fix      # ESLint auto-fix
npm run typecheck     # Type checking
```
Has its own `CLAUDE.md` with Base44 platform rules — **read it before editing besure**.

### daf-rules-check (React + Base44)
```bash
cd src/daf-rules-check
npm run dev           # Vite dev server
npm run build         # Production build
npm run lint          # ESLint check (quiet mode)
npm run lint:fix      # ESLint auto-fix
```

### dafman-compliance-agent-v2 (Python FastAPI)
```bash
cd src/dafman-compliance-agent-v2/backend
pip install -r requirements.txt
python scripts/ingest_dafman.py        # One-time: embed DAFMAN PDF into ChromaDB
uvicorn main:app --reload --port 8000  # Dev server
```
Also has a `frontend-base44/` subdirectory following Base44 conventions.

### CKL_Checker (PowerShell)
```powershell
.\CKL_Checker_V24.ps1   # Run directly (Windows Forms GUI)
```

### n8n Workflows (arkcybr-soc-workflows, social-media-content-factory)
No build step. Import JSON files into n8n via UI or Docker CLI.

## Architecture: Base44 Platform (besure, daf-rules-check)

Two projects use the Base44 full-stack platform. Key constraints:

- **`pages/` must be flat** — no subdirectories, each file is a route
- **`functions/` must be flat** — each Deno function deploys independently, no local imports between functions
- **`entities/` are pure JSON** — database schemas, no JavaScript
- Functions reference each other without `.ts` extension internally; GitHub export adds `.ts` automatically
- Functions must use `Deno.serve()` + `Response.json()` pattern with `createClientFromRequest(req)` from `npm:@base44/sdk@0.8.6`
- `base44.asServiceRole` bypasses Row Level Security for admin operations
- Changes pushed to GitHub auto-sync to Base44 within ~60 seconds
- Frontend SDK: `base44.auth.*`, `base44.entities.*`, `base44.functions.invoke()`

## Architecture: dafman-compliance-agent-v2

RAG pipeline: PDF/DOCX upload → document parsing (PyMuPDF/python-docx) → ChromaDB vector store with DAFMAN reference text → Claude/GPT-4o evaluation → structured findings → human review workflow.

API routes are modular under `backend/api/routes/` (upload, check, findings, review, report).

## DAFMAN 90-161 Compliance Rules (shared by daf-rules-check and dafman-compliance-agent-v2)

14 rules across 5 categories: Header (HDR-001 through HDR-006), Directive Language (DIR-001, DIR-002), Waiver (WAI-001, WAI-002), Records (REC-001), Structure (STR-001), Privacy (PRI-001, PRI-002). Results are PASS/FAIL/WARNING/NOT_APPLICABLE with DAFMAN citations.

## Git Workflow

Each project has its own git remote. Work within each project's directory for git operations:
```bash
git -C src/besure pull origin main
git -C src/besure add -A && git -C src/besure commit -m "message" && git -C src/besure push origin main
```

For besure specifically, always `git fetch` and check for upstream changes before editing — the Base44 AI builder also pushes to the same repo.

## Security & Credentials

- n8n workflows: all API keys stored in n8n Credential Manager, never hardcoded in JSON
- Base44 apps: secrets stored in Base44 Dashboard → Secrets (STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, etc.)
- dafman-compliance-agent-v2: `.env` file for OPENAI_API_KEY, CHROMA_DB_PATH, SMTP/Slack config
- The `aaAntiRansomElastic-*` and `zzAntiRansomElastic-*` directories are Elastic Security canary files — do not touch
