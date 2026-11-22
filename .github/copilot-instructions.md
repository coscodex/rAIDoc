# Copilot Instructions — rAIdoc

rAIdoc is an AI-powered vehicle inspection assistant. A user uploads an image of a component; a vision model classifies the defect; RAG retrieves technical documentation; an LLM writes a grounded diagnostic report. It is an **inspection aid, not a replacement for a professional mechanic.**

Full design: `docs/PROJECT_BLUEPRINT.md`. Read it before making architectural changes.

## Scope (do not expand without being asked)

- Components: **motorcycle chain** (rust, wear) and **vehicle tyre** (cracks, tread wear). Nothing else.
- No agents, no LangGraph, no user accounts, no payments, no marketplace.
- Do not add LangChain unless explicitly requested. Write the RAG pipeline as plain, readable, modular Python.

## Stack

- Backend: Python 3.11+, FastAPI, Pydantic v2, uvicorn
- Vision: PyTorch + torchvision / Hugging Face `transformers`, small backbone (MobileNet / ResNet-18 class), CPU-friendly
- Embeddings: `sentence-transformers`
- Vector store: ChromaDB (persistent, local)
- Structured facts: SQLite or YAML/JSON file for per-defect facts (severity, action, interval). Postgres only if requested.
- LLM: accessed ONLY through `app/rag/llm_client.py` (see below)
- Frontend: React (Vite), plain CSS or Tailwind, no heavy UI libraries
- Tests: pytest

## Folder structure

```
backend/app/{main.py, config.py, routes/, vision/, rag/, schemas/, services/}
backend/knowledge_base/   # source docs + structured facts
backend/tests/
frontend/
docs/                     # PROJECT_BLUEPRINT.md, CHANGELOG.md, DECISIONS.md
notebooks/                # vision experiments only
```

Follow this layout. Inspect the existing repo first and preserve working code; do not restructure or rewrite files unless asked.

## Coding rules

1. Define or reuse Pydantic schemas in `schemas/` **before** writing routes or services. Never invent response shapes inline.
2. One module = one responsibility. Keep functions small, typed, and documented with short docstrings.
3. No secrets in code. Read config from environment variables via `config.py`; keep `.env.example` updated; never commit `.env`.
4. Do not add a new dependency without saying why. Prefer the standard library and existing dependencies.
5. Every new module gets at least one pytest test, runnable without network access (mock the LLM and any downloads).
6. Handle failures explicitly: bad file type, oversized image, unreadable image, empty retrieval, LLM timeout. Return clear HTTP errors; never crash silently.
7. Log with the `logging` module, not `print`.
8. Explain non-obvious code in comments. The developer must be able to explain every line in an interview.

## Vision conventions

- Model output is a single combined label (e.g. `chain_rust`, `chain_wear`, `tyre_crack`, `tyre_wear`, plus `unknown`/`unsupported`) with a confidence score.
- Apply a configurable confidence threshold (`CONFIDENCE_THRESHOLD`, default 0.6). Below it, return a low-confidence result asking for a clearer image instead of a diagnosis.
- Training and evaluation scripts live in `notebooks/` or `backend/app/vision/training/`; report precision, recall, F1 and a confusion matrix on a held-out test set.

## RAG conventions

- Pipeline: extract → clean → chunk → embed → store → retrieve → generate.
- Every chunk keeps metadata: `document_name`, `component`, `page`, `doc_type`, `chunk_id`.
- Filter retrieval by component. Return top-k chunks with similarity scores.
- If retrieval returns nothing relevant (score below `MIN_RETRIEVAL_SCORE`), the report must say there is insufficient documentation. Do not fall back to model memory for specifications.
- Ingestion is a script (`python -m app.rag.ingestion`), not a public endpoint, unless requested.

## LLM client (provider-agnostic)

- All LLM calls go through `app/rag/llm_client.py` using an **OpenAI-compatible chat API**, configured by env vars: `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`.
- Default: local Ollama (`http://localhost:11434/v1`). Switching to a free-tier hosted API must need only env changes, no code changes.
- Small local models are weaker: keep prompts short and explicit, request JSON output matching a Pydantic schema, validate it, and retry once on invalid output.
- The prompt must include: component, predicted defect, confidence, retrieved context with source ids, and instructions to separate confirmed documentation from uncertain observations and to cite sources.

## Safety and wording rules

- Never present an AI prediction as a confirmed mechanical diagnosis. Use language such as "appears to show", "possible", "consistent with".
- For safety-critical findings (e.g. tyre cracks, severe chain wear), always recommend professional inspection.
- Never invent technical specifications (torque values, wear limits, pressures). Only state values found in retrieved sources, and cite them.
- Always include a short disclaimer field in the `/inspect` response.

## API

- `GET /health`, `POST /inspect` (image upload → full diagnostic), `GET /sources`.
- Validate file type (jpg/png/webp) and size limit on upload.
- Response schema lives in `schemas/diagnostic.py`; keep the frontend types in sync.

## Workflow

- Work on one small task at a time. Before coding a larger task, state a short plan and the files you will touch.
- After changes, tell me how to run and test them.
- Record architecture decisions in `docs/DECISIONS.md` and notable changes in `docs/CHANGELOG.md`.
- If a requirement is ambiguous or conflicts with these rules, ask instead of guessing.