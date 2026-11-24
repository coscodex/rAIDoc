# Copilot Instructions — rAIdoc

**Instructions version:** v1.2 (matches `docs/PROJECT_BLUEPRINT.md` v1.2, 24 Nov 2026)

rAIdoc is an AI-powered two-wheeler inspection and diagnostic assistant. It has two features:

1. **Inspect (image):** a user uploads a photo of a component; a vision model classifies the defect; RAG retrieves technical documentation; an LLM writes a grounded diagnostic report.
2. **Ask (text):** a user picks a bike model and describes a symptom; the system retrieves model-specific known issues and maintenance documents; an LLM lists possible causes, what to check first, and what is safety-critical.

It is an **inspection aid, not a replacement for a professional mechanic.**

Full design: `docs/PROJECT_BLUEPRINT.md`. Read it before making architectural changes. Keep this file and the blueprint in sync, and bump the version in both when the design changes.

## Scope (do not expand without being asked)

- Components (image inspection): **motorcycle chain** (rust, wear) and **tyre** (cracks, tread wear). A third component is added only after both have an evaluated baseline.
- Bike models (`/ask`): **Hero Splendor, Bajaj Pulsar, Royal Enfield Classic 350, Royal Enfield Classic 650** (Classic 650 is added last). Exact variants and model years are pinned in `docs/DECISIONS.md`; never mix data from different variants.
- Scooters are out of scope.
- No agents, no LangGraph, no user accounts, no payments, no marketplace.
- Do not add LangChain unless explicitly requested. Write the RAG pipeline as plain, readable, modular Python.

## Stack

- Backend: Python 3.11+, FastAPI, Pydantic v2, uvicorn
- Environment: virtual environment at `backend/.venv` managed with **uv** (`uv venv`, `uv pip install -r requirements.txt`). Give uv commands, not plain pip, in instructions.
- Vision: PyTorch + torchvision / Hugging Face `transformers`, small backbone (MobileNet / ResNet-18 class), CPU-friendly
- Embeddings: `sentence-transformers`
- Vector store: ChromaDB (persistent, local)
- Structured model facts: YAML files under `backend/knowledge_base/models/`. No PostgreSQL for now.
- LLM: accessed ONLY through `app/rag/llm_client.py` (see below)
- Frontend: React (Vite), plain CSS or Tailwind, no heavy UI libraries
- Tests: pytest

## Folder structure

```
backend/app/{main.py, config.py, routes/ (health, inspect, ask), vision/, rag/, schemas/ (diagnostic.py, ask.py), services/}
backend/knowledge_base/   # documents/ (manuals, guides) and models/ (YAML issue entries)
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
6. Handle failures explicitly: bad file type, oversized image, unreadable image, empty retrieval, unknown bike model, LLM timeout. Return clear HTTP errors; never crash silently.
7. Log with the `logging` module, not `print`.
8. Explain non-obvious code in comments. The developer must be able to explain every line in an interview.

## Vision conventions

- Model output is a single combined label (e.g. `chain_rust`, `chain_wear`, `tyre_crack`, `tyre_wear`, plus `unknown`/`unsupported`) with a confidence score. Add severity and defect location (box or heatmap) only if the dataset supports it.
- Run an image quality check (blur, brightness, framing) before inference; ask for a retake if the image is poor.
- Apply a configurable confidence threshold (`CONFIDENCE_THRESHOLD`, default 0.6). Below it, or for out-of-scope images, return a low-confidence result asking for a clearer image instead of a diagnosis.
- Training and evaluation scripts live in `notebooks/` or `backend/app/vision/training/`; report precision, recall, F1 and a confusion matrix on a held-out test set.

## RAG conventions

- Pipeline: extract → clean → chunk → embed → store → retrieve → generate.
- Every chunk keeps metadata: `document_name`, `bike_model` (or `general`), `component`, `page`, `doc_type`, `chunk_id`.
- Filter retrieval by bike model and component. Return top-k chunks with similarity scores.
- Model-specific facts live in YAML entries with fields: `model`, `component`, `issue`, `symptoms`, `severity` (low | medium | high | safety_critical), `recommended_action`, `source`, `evidence_type` (manufacturer | workshop_guidance | owner_reported). Look these up exactly, in parallel with RAG, and pass both to the LLM.
- If retrieval returns nothing relevant (score below `MIN_RETRIEVAL_SCORE`), or the bike model or symptom is not covered, say there is insufficient documentation and recommend a mechanic. Do not fall back to model memory for specifications or model-specific faults.
- Do not scrape reviews or forums. Owner-reported issues are summarised by hand into YAML entries and tagged `owner_reported` (lower confidence than manufacturer guidance). Official owner's manuals and service schedules are the backbone.
- Ingestion is a script (`python -m app.rag.ingestion`), not a public endpoint, unless requested.

## LLM client (provider-agnostic)

- All LLM calls go through `app/rag/llm_client.py` using an **OpenAI-compatible chat API**, configured by env vars: `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`.
- Default: local Ollama (`http://localhost:11434/v1`). Switching to a free-tier hosted API must need only env changes, no code changes.
- Small local models are weaker: keep prompts short and explicit, request JSON output matching a Pydantic schema, validate it, and retry once on invalid output.
- The prompt must include: component or symptom, bike model, predicted defect and confidence (if any), retrieved context with source ids, the model-specific facts, and instructions to separate documented information from uncertain observations and to cite sources.

## Safety and wording rules

- Never present an AI prediction as a confirmed mechanical diagnosis. Use language such as "appears to show", "possible", "consistent with".
- For safety-critical findings (e.g. tyre cracks, severe chain wear, brake problems), always recommend professional inspection.
- Never invent technical specifications (torque values, wear limits, pressures, intervals) or model-specific faults. Only state values found in retrieved sources or the YAML entries, and cite them.
- If a bike model has limited issue history (e.g. the recently launched Classic 650), say so instead of guessing.
- Always include a short disclaimer field in the `/inspect` and `/ask` responses.

## API

- `GET /health` (done), `POST /inspect` (image upload → full diagnostic), `POST /ask` (`bike_model` + `symptom` → ranked possible causes with sources), `GET /sources`.
- Validate file type (jpg/png/webp) and size limit on upload. Validate `bike_model` against the supported list.
- Response schemas live in `schemas/diagnostic.py` and `schemas/ask.py`; keep the frontend types in sync.

## Workflow

- Work on one small task at a time. Before coding a larger task, state a short plan and the files you will touch.
- After changes, tell me how to run and test them.
- Record architecture decisions in `docs/DECISIONS.md` and notable changes in `docs/CHANGELOG.md`.
- If a requirement is ambiguous or conflicts with these rules, ask instead of guessing.