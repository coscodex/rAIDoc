# rAIdoc — Technical Blueprint & Development Roadmap

**Version:** 1.0  
**Last Updated:** 23 November 2026  
**Development Target:** 48 Hours  
**Project Type:** AI-powered Automotive Inspection and Diagnostic Assistant  
**Status:** Planning → Implementation

---

## 1. Project Overview

### What is rAIdoc?

rAIdoc is an AI-powered automotive inspection and diagnostic assistant that combines computer vision, Retrieval-Augmented Generation (RAG), and Large Language Models (LLMs).

The application accepts an image of a vehicle component, identifies potential defects using a computer vision model, retrieves relevant technical information from automotive documentation, and generates an understandable diagnostic report.

### Problem Statement

Traditional vehicle inspections depend heavily on manual inspection and the availability of experienced technicians.

rAIdoc aims to assist this process by:

- Identifying visible defects in automotive components.
- Retrieving relevant information from technical manuals.
- Generating context-aware explanations.
- Suggesting appropriate inspection or maintenance actions.
- Providing traceable supporting technical information.

**Important:** rAIdoc is an inspection assistance tool, not a replacement for professional vehicle inspection.

---

## 2. Initial MVP Scope

We will deliberately limit the first version to two automotive components.

| Component | Initial defects |
|---|---|
| Motorcycle chain | Rust, visible wear |
| Vehicle tyre | Cracks, visible wear |

### Core functionality

1. User uploads a component image.
2. Vision model processes the image.
3. System identifies the component and potential defect.
4. Backend constructs a diagnostic query.
5. RAG retrieves relevant technical documentation.
6. LLM generates a grounded diagnostic explanation.
7. Frontend displays the results and supporting references.

### MVP output

- Uploaded image.
- Component identification.
- Detected defect.
- Model confidence score.
- Explanation of the defect.
- Relevant technical information.
- Recommended inspection or maintenance action.
- Source references.
- Uncertainty warnings where necessary.

---

## 3. Technology Stack

The following is our working stack. Existing repository dependencies and implementation constraints will take precedence when inspected.

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React | User interface |
| Backend | Python + FastAPI | API and orchestration |
| Computer Vision | Hugging Face / PyTorch | Image classification |
| LLM | External LLM API | Diagnostic explanation |
| Embeddings | Sentence Transformers | Document embeddings |
| Vector Database | ChromaDB (provisional) | Semantic retrieval |
| Structured Database | PostgreSQL (planned) | Structured diagnostic data |
| RAG Orchestration | Python (modular implementation) | Retrieval and generation |
| API Testing | Postman / Swagger | Endpoint testing |
| Version Control | Git + GitHub | Source management |

### Important implementation decision

We will avoid unnecessary frameworks and abstractions.

The initial RAG pipeline will be implemented in understandable, modular Python code. We can introduce LangChain later if it provides a concrete advantage.

We will also avoid introducing LangGraph or autonomous agents into the MVP.

---

## 4. System Architecture

### High-level workflow

**User → React → FastAPI → AI Pipeline → Diagnostic Report**

The AI pipeline contains two primary branches.

### Branch A: Computer Vision

1. Receive uploaded image.
2. Validate image format and dimensions.
3. Preprocess image.
4. Run vision model.
5. Obtain predicted component/defect and confidence.
6. Return structured prediction.

### Branch B: RAG Pipeline

1. Collect automotive manuals and technical documents.
2. Extract text from documents.
3. Clean and preprocess text.
4. Split documents into meaningful chunks.
5. Generate embeddings.
6. Store embeddings in vector database.
7. Retrieve relevant chunks using the diagnostic query.
8. Pass retrieved context to the LLM.
9. Generate a grounded response with source references.

### Final orchestration

The FastAPI backend combines the vision prediction and RAG output.

The LLM should receive:
- Component identification.
- Predicted defect.
- Vision confidence.
- Relevant retrieved documentation.
- Instructions to distinguish confirmed information from uncertain observations.

---

## 5. Knowledge Base Design

Our RAG system must use reliable automotive knowledge.

### Initial document sources

- Automotive maintenance manuals.
- Manufacturer technical documentation.
- Component maintenance guides.
- Curated engineering reference material.

### Document processing pipeline

PDF / DOCX → Text Extraction → Cleaning → Chunking → Embeddings → ChromaDB

Each chunk should retain metadata:

- Document name.
- Component category.
- Source location or page number.
- Document type.
- Chunk identifier.

### Retrieval requirements

- Semantic similarity search.
- Component-specific filtering where appropriate.
- Relevant document references.
- Retrieval quality testing.

We will not allow the LLM to invent technical specifications when the knowledge base does not contain supporting information.

---

## 6. Backend Structure

Proposed modular structure:

```text
rAidoc/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── routes/
│   │   │   ├── inspection.py
│   │   │   └── health.py
│   │   │
│   │   ├── vision/
│   │   │   ├── classifier.py
│   │   │   └── preprocessing.py
│   │   │
│   │   ├── rag/
│   │   │   ├── ingestion.py
│   │   │   ├── chunking.py
│   │   │   ├── embeddings.py
│   │   │   ├── retrieval.py
│   │   │   └── generation.py
│   │   │
│   │   ├── services/
│   │   │   └── diagnostic_service.py
│   │   │
│   │   ├── schemas/
│   │   │   └── diagnostic.py
│   │   │
│   │   └── config.py
│   │
│   ├── knowledge_base/
│   ├── tests/
│   ├── requirements.txt
│   └── .env.example
│
├── frontend/
│
├── docs/
│   ├── PROJECT_BLUEPRINT.md
│   └── CHANGELOG.md
│
└── README.md
```

This is a proposed structure, not an instruction to recreate the existing repository. We will adapt it after inspecting the current codebase.

---

## 7. API Design

### Initial endpoints

| Method | Endpoint | Responsibility |
|---|---|---|
| GET | `/health` | Backend health check |
| POST | `/inspect` | Upload image and run complete diagnostic pipeline |
| POST | `/ingest` | Add technical documents to knowledge base |
| GET | `/sources` | Retrieve available knowledge-base source information |

### Expected `/inspect` response

```json
{
  "component": "motorcycle_chain",
  "predicted_defect": "rust",
  "confidence": 0.91,
  "explanation": "Visible corrosion detected...",
  "technical_context": [],
  "recommendations": [],
  "sources": [],
  "warnings": []
}
```

The response structure is illustrative and will be refined during implementation.

---

## 8. Frontend Requirements

The initial interface should be simple and functional.

### Main screen

- rAIdoc branding.
- Component image upload.
- Image preview.
- Analyze button.
- Loading indicator.

### Diagnostic results screen

- Uploaded image.
- Identified component.
- Defect classification.
- Confidence indicator.
- AI-generated explanation.
- Technical information retrieved from documentation.
- Recommendations.
- Source references.
- Uncertainty warnings.

We will prioritize functionality and usability over complex UI animations.

---

## 9. 48-Hour Development Roadmap

### DAY 1 — Backend and AI Intelligence

**Phase 1: Repository inspection (1 hour)**

- Inspect existing repository.
- Identify implemented components.
- Check dependencies.
- Preserve working code.
- Finalize implementation architecture.

**Phase 2: Backend foundation (1–2 hours)**

- Configure FastAPI.
- Establish API routes.
- Implement image upload.
- Validate API using Swagger.

**Phase 3: Computer vision (2 hours)**

- Select an appropriate pretrained model.
- Implement image preprocessing.
- Integrate classification.
- Test component and defect predictions.
- Evaluate whether the model supports our actual defect categories.

**Phase 4: RAG implementation (3 hours)**

- Collect initial technical documents.
- Implement text extraction.
- Implement chunking.
- Generate embeddings.
- Set up ChromaDB.
- Implement semantic retrieval.
- Verify retrieved chunks.

**Phase 5: LLM integration (2 hours)**

- Integrate LLM API.
- Construct diagnostic prompt.
- Pass retrieved context.
- Generate grounded explanations.
- Return references and uncertainty warnings.

**Day 1 deliverable:** Working backend with vision and RAG pipelines.

### DAY 2 — Integration and Application

**Phase 6: Pipeline integration (2 hours)**

- Connect vision output to RAG.
- Build diagnostic orchestration.
- Standardize response schema.
- Test complete backend workflow.

**Phase 7: Frontend development (3 hours)**

- Build image upload interface.
- Connect FastAPI.
- Display diagnostic results.
- Display sources and recommendations.

**Phase 8: Testing and debugging (2 hours)**

- Test valid images.
- Test unsupported images.
- Test low-confidence predictions.
- Test missing retrieval results.
- Test API failures.
- Verify source references.

**Phase 9: Finalization (1 hour)**

- Update README.
- Document architecture.
- Add setup instructions.
- Update GitHub repository.
- Record known limitations and future improvements.

**Day 2 deliverable:** Integrated rAIdoc MVP.

---

## 10. Definition of Done

The MVP will be considered complete when:

- [ ] Frontend accepts image uploads.
- [ ] Backend processes images successfully.
- [ ] Vision model returns predictions.
- [ ] Technical documents can be ingested.
- [ ] Embeddings are stored in vector database.
- [ ] Relevant documents are retrieved.
- [ ] LLM generates context-grounded explanations.
- [ ] Sources are displayed in the frontend.
- [ ] Confidence and uncertainty are handled.
- [ ] Complete application works end-to-end.

A working pipeline does not automatically mean the vision model is diagnostically accurate. We will separately evaluate its predictions using appropriate test images.

---

## 11. Known Risks and Limitations

| Risk | Mitigation |
|---|---|
| Pretrained model does not recognize our defect classes | Evaluate model suitability and use a suitable classifier or dataset |
| Insufficient technical documentation | Start with a small, curated knowledge base |
| Poor retrieval quality | Tune chunking, metadata and retrieval |
| LLM hallucinations | Ground generation in retrieved context |
| API latency | Keep initial pipeline lightweight |
| Limited development time | Prioritize the end-to-end workflow |
| GPU memory constraints | Prefer smaller models or CPU-compatible alternatives |

---

## 12. Development Principles

These rules will guide our implementation.

1. Do not rewrite existing working code without a reason.
2. Build one component at a time.
3. Test every module independently before integration.
4. Understand the implementation rather than blindly copying code.
5. Avoid unnecessary frameworks and dependencies.
6. Keep secrets and API keys in environment variables.
7. Update this document whenever architecture changes.
8. Record bugs, decisions and unresolved questions.
9. Prioritize working functionality over theoretical completeness.
10. Never present uncertain AI predictions as confirmed mechanical diagnoses.

---

## 13. Change Log

| Version | Date | Change |
|---|---|---|
| 1.0 | 23 Nov 2026 | Initial architecture and 48-hour MVP roadmap |

### Future updates

Every significant change will be documented using:

- Date.
- Previous implementation.
- New implementation.
- Reason for change.
- Impact on other modules.

---

## 14. Immediate Next Step

**Before writing new code, inspect the existing GitHub repository.**

We need to establish:

1. What code already exists?
2. What dependencies are installed?
3. Which components can be reused?
4. What needs modification?
5. What is the first executable task?

Once the repository is inspected, we will convert this blueprint into an implementation-specific development plan.

**Our objective: A working, demonstrable rAIdoc MVP within 48 hours, with a modular architecture that we can extend beyond the initial release.**