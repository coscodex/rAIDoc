# rAIdoc — Technical Blueprint & Development Roadmap

**Version:** 1.2
**Last Updated:** 24 November 2026
**Project Type:** AI-powered two-wheeler inspection and diagnostic assistant
**Status:** Skeleton done (`/health` works) → vision dataset search

---

## 1. Project Overview

rAIdoc is an AI-powered two-wheeler inspection and diagnostic assistant combining computer vision, Retrieval-Augmented Generation (RAG) and a Large Language Model (LLM).

It has two ways in:

1. **Inspect (image):** upload a photo of a component → vision model finds a defect → RAG retrieves documentation → LLM writes a grounded report.
2. **Ask (text):** pick a bike model, describe a symptom → system retrieves model-specific known issues and maintenance documents → LLM lists possible causes, what to check first, and what is safety-critical.

**Important:** rAIdoc is an inspection assistance tool, not a replacement for a professional mechanic. It never presents an AI prediction as a confirmed diagnosis.

### What makes it different from uploading a photo to a general chatbot

1. **Grounded, cited answers** from a curated knowledge base, with "insufficient information" when the documents do not cover something.
2. **Measured behaviour:** precision/recall, confusion matrix, retrieval and groundedness metrics, and a comparison against a general multimodal model on the same test images.
3. **Consistent structured output** (same schema every time).
4. **Calibrated confidence and abstention:** rejects blurry or out-of-scope images instead of guessing.
5. **Local and private option:** runs with a local Ollama model; no images leave the machine.
6. **Model-specific knowledge** for four popular Indian motorcycles.

---

## 2. MVP Scope

### Components (image inspection)

| Component | Initial defects |
|---|---|
| Motorcycle chain | Rust/corrosion, visible wear |
| Vehicle tyre | Cracks, tread wear |

A third component (candidates: sprocket wear, brake pad/disc wear, battery terminal corrosion) is added **only after** both components above have a working, evaluated baseline, and only if a usable dataset exists.

### Bike models (ask feature)

Four models, chosen to cover different kinds of motorcycle:

1. Hero Splendor (commuter, best-selling motorcycle in India)
2. Bajaj Pulsar (sporty, high-selling)
3. Royal Enfield Classic 350 (single-cylinder cruiser)
4. Royal Enfield Classic 650 (parallel-twin cruiser, launched in India in March 2025, so long-term issue history is limited)

**Build order:** Splendor, Pulsar and Classic 350 first; Classic 650 once the knowledge-base format is proven.

**Pin the exact variant and model years for each** (for example, which Pulsar variant and which Classic 350 generation) and record them in `docs/DECISIONS.md`. Issues, service intervals and manuals differ between variants, so the knowledge base must not mix them.

Scooters are out of scope (they use belt/CVT drives, not chains). More models are added only after these are well covered.

### Explicitly out of scope

Mechanic marketplace, payments, ratings, video consultation, maps, user accounts, more than a handful of bike models, autonomous agents.

### MVP output

**/inspect:** uploaded image, component, detected defect, severity (if available), confidence, defect location (box or heatmap, if available), explanation, retrieved technical context, recommendations, sources, warnings, disclaimer.

**/ask:** bike model, symptom, ranked possible causes, what to inspect first, safety-critical flag, sources with evidence type, warnings, disclaimer.

---

## 3. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React (Vite) | User interface |
| Backend | Python + FastAPI | API and orchestration |
| Environment | `uv`, virtual environment at `backend/.venv` | Dependency management |
| Computer Vision | PyTorch + torchvision / Hugging Face | Defect classification (and localisation if data allows) |
| LLM | Local Ollama model by default; free-tier hosted API as fallback | Diagnostic explanation |
| LLM access | One OpenAI-compatible client (`rag/llm_client.py`) configured by env vars | Switch providers without code changes |
| Embeddings | Sentence Transformers | Document embeddings |
| Vector Database | ChromaDB | Semantic retrieval |
| Structured model facts | YAML/JSON files (SQLite optional) | Known issues per bike model |
| RAG Orchestration | Plain modular Python | Retrieval and generation |
| Testing | pytest, Swagger (`/docs`) | Module and endpoint tests |
| Version Control | Git + GitHub | Source management |

### Implementation decisions

- Build the RAG pipeline in plain Python first, to learn and explain every step. LangChain may be added later for a specific benefit.
- No agents or LangGraph in the MVP.
- PostgreSQL is dropped for the MVP; structured model facts live in YAML/JSON. Revisit later.

---

## 4. System Architecture

**User → React → FastAPI → AI pipeline → report**

### Branch A: Computer vision (`/inspect`)

1. Receive and validate the image (type, size, readability).
2. Quality gate (blur, brightness, framing); ask for a retake if poor.
3. Preprocess.
4. Run the vision model → component/defect label, confidence, severity and location where supported.
5. If confidence is below `CONFIDENCE_THRESHOLD` (default 0.6) or the image is out of scope, return a low-confidence result asking for a clearer image.
6. Return the structured prediction.

### Branch B: RAG (shared by `/inspect` and `/ask`)

1. Collect documents (owner's manuals, service schedules, maintenance guides, curated notes).
2. Extract, clean, chunk.
3. Embed and store in ChromaDB with metadata.
4. Retrieve with component and bike-model filters.
5. Pass retrieved context to the LLM.
6. Generate a grounded answer with source references.

### Branch C: Model-specific facts (`/ask`)

Structured entries per bike model are looked up exactly, in parallel with RAG, and both feed the LLM.

### LLM input

- Component or symptom, bike model, vision result and confidence (if any).
- Retrieved chunks with source ids and the model-specific facts.
- Instructions to separate documented information from uncertain observations, to cite sources, and to say "insufficient information" when unsupported.

---

## 5. Knowledge Base Design

### Sources, in priority order

1. Manufacturer owner's manuals and service schedules (official, public).
2. Component maintenance guides and engineering references.
3. Curated notes written by the developer, each marked with its source and evidence type.
4. Owner-reported issues, summarised by hand into structured entries and tagged as lower-confidence. Raw reviews or forum posts are **not** scraped or used as the core source.

### Chunk metadata

`document_name`, `bike_model` (or `general`), `component`, `page`, `doc_type`, `chunk_id`.

### Model-specific issue entry (example format)

```yaml
- model: <bike model>
  component: chain
  issue: <short description>
  symptoms: [<symptom>, <symptom>]
  severity: low | medium | high | safety_critical
  recommended_action: <text>
  source: <document or link>
  evidence_type: manufacturer | workshop_guidance | owner_reported
```

### Rules

- The LLM must not invent specifications or model-specific problems that are not in the knowledge base.
- If the model or symptom is not covered, say so and suggest a mechanic.
- Brake, tyre and chain issues always recommend professional inspection when severity is high.

---

## 6. Backend Structure

```text
rAIdoc/
├── .github/copilot-instructions.md
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── routes/          # health, inspect, ask
│   │   ├── vision/          # preprocessing, quality gate, classifier
│   │   ├── rag/             # ingestion, chunking, embeddings, retrieval, generation, llm_client
│   │   ├── schemas/         # diagnostic.py, ask.py
│   │   └── services/        # orchestration
│   ├── knowledge_base/      # documents + model facts (YAML)
│   ├── tests/
│   ├── requirements.txt
│   └── .env.example
├── frontend/
├── notebooks/               # vision experiments
├── docs/                    # PROJECT_BLUEPRINT.md, DECISIONS.md, CHANGELOG.md
└── README.md
```

---

## 7. API Design

| Method | Endpoint | Responsibility |
|---|---|---|
| GET | `/health` | Backend health check (done) |
| POST | `/inspect` | Image upload → full diagnostic |
| POST | `/ask` | Bike model + symptom → possible causes with sources |
| GET | `/sources` | Knowledge-base source information |

Ingestion is a script (`python -m app.rag.ingestion`), not a public endpoint, for the MVP.

### `/inspect` response (illustrative)

```json
{
  "component": "motorcycle_chain",
  "predicted_defect": "rust",
  "severity": "moderate",
  "confidence": 0.91,
  "low_confidence": false,
  "location": null,
  "explanation": "...",
  "technical_context": [],
  "recommendations": [],
  "sources": [],
  "warnings": [],
  "disclaimer": "..."
}
```

### `/ask` request and response (illustrative)

```json
{ "bike_model": "<model>", "symptom": "chain noise at low speed" }
```

```json
{
  "bike_model": "<model>",
  "possible_causes": [
    { "cause": "...", "likelihood": "possible", "check_first": "...", "severity": "medium", "sources": [] }
  ],
  "safety_critical": false,
  "insufficient_information": false,
  "warnings": [],
  "disclaimer": "..."
}
```

Schemas are defined in `schemas/` before routes are written.

---

## 8. Frontend Requirements

Simple and functional.

- Two tabs: **Inspect** (upload, preview, analyze, loading state) and **Ask** (bike model dropdown, symptom box).
- Results: image, component, defect, confidence indicator, severity, explanation, retrieved information, recommendations, source references, warnings.
- If the symptom relates to a supported component, suggest uploading a photo.

---

## 9. Roadmap (no fixed deadline; one small task at a time)

1. **Repo setup — done.** Blueprint, Copilot instructions, `.gitignore`, backend skeleton, `/health`.
2. **Vision dataset search.** Chain (rust, wear), tyre (cracks, tread wear), and candidates for a third component. Prefer datasets with bounding boxes, masks or severity labels. Record licence, class counts, quality and verdict in `docs/DECISIONS.md`.
3. **Vision baseline notebook.** Transfer learning on a small backbone; report precision, recall, F1 and a confusion matrix on a held-out test set.
4. **Schemas.** `schemas/diagnostic.py` and `schemas/ask.py` with tests.
5. **Knowledge base.** Collect owner's manuals and service schedules for the chosen bikes (Classic 650 last); write the structured issue entries.
6. **RAG pipeline.** Extraction, chunking, embeddings, ChromaDB, retrieval; verify retrieved chunks by hand.
7. **LLM integration.** `llm_client.py` with Ollama; grounded prompts; JSON output validation.
8. **Endpoints.** `/inspect`, then `/ask`.
9. **Frontend.**
10. **Evaluation and comparison.** Vision metrics, retrieval relevance on 15–20 hand-written questions, groundedness, latency, and a comparison against a general multimodal model.
11. **Finalisation.** README, setup instructions, known limitations, future work.

---

## 10. Evaluation Plan

| Area | Measure |
|---|---|
| Vision | Precision, recall, F1 per class, confusion matrix, held-out test set |
| Retrieval | Whether the correct chunk appears in the top-k for 15–20 hand-written questions |
| Groundedness | Share of claims supported by retrieved sources; count of invented specifications |
| Abstention | Behaviour on blurry, out-of-scope and uncovered-model inputs |
| System | Latency and failure cases |
| Comparison | Same test images and questions run through a general multimodal model; compare label accuracy, consistency, citations and invented specifications |

---

## 11. Definition of Done

- [ ] Frontend accepts image uploads and symptom queries.
- [ ] Vision model returns predictions with confidence; low-confidence and out-of-scope inputs are handled.
- [ ] Knowledge base ingested with metadata for the chosen bike models (variants pinned).
- [ ] Relevant documents are retrieved and cited.
- [ ] LLM output is grounded and validated against the schema.
- [ ] `/inspect` and `/ask` work end-to-end.
- [ ] Evaluation results are recorded in the README.

A working pipeline does not mean the vision model is accurate; accuracy is evaluated separately.

---

## 12. Known Risks and Limitations

| Risk | Mitigation |
|---|---|
| No suitable chain dataset | Use a generic rust dataset, collect own images, or start tyre-first |
| Small local LLM follows instructions poorly | Short explicit prompts, JSON schema validation, one retry, hosted fallback |
| Thin or unreliable model-specific data (especially the newer Classic 650) | Few models; official sources first; tag evidence type; say so when issue history is limited |
| Poor retrieval quality | Tune chunking and metadata filters; evaluate on test questions |
| LLM hallucination | Ground in retrieved context; abstain when unsupported |
| Copyright and terms-of-use issues with documents and reviews | Use official public manuals; write own summaries; no scraping |
| Safety | Possible-cause wording, professional inspection for safety-critical findings, disclaimer in every response |

---

## 13. Development Principles

1. Do not rewrite working code without a reason.
2. Build and test one component at a time; commit after every working step.
3. Understand every module; do not blindly copy generated code.
4. Define schemas before routes.
5. Avoid unnecessary frameworks and dependencies.
6. Keep secrets in environment variables.
7. Update this document and `DECISIONS.md` when the design changes.
8. Never present uncertain AI predictions as confirmed mechanical diagnoses.
9. Never state a specification or model-specific fault that is not in the knowledge base.

---

## 14. Change Log

| Version | Date | Change |
|---|---|---|
| 1.0 | 23 Nov 2026 | Initial architecture and 48-hour roadmap |
| 1.1 | 24 Nov 2026 | Removed fixed deadline; vision-first order; added `/ask` feature and model-specific facts; Ollama via OpenAI-compatible client; PostgreSQL deferred; added differentiation, evaluation plan and quality gating |
| 1.2 | 24 Nov 2026 | Bike models set to Splendor, Pulsar, Classic 350 and Classic 650 (Classic 650 added after the first three); variants to be pinned |