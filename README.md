# MetaVault 🏛️
### Enterprise Metadata-as-a-Service (MaaS) & Structure-Aware Knowledge Platform

---

## Table of Contents
- [1. Use Case & Problem Statement](#1-use-case--problem-statement)
- [2. Core Design Principles](#2-core-design-principles)
- [3. Current Features Implemented](#3-current-features-implemented)
- [4. Architecture & Pipeline](#4-architecture--pipeline)
- [5. Technology Stack](#5-technology-stack)
- [6. Project Structure](#6-project-structure)
- [7. Setup & Deployment Instructions](#7-setup--deployment-instructions)
- [8. Usage Guide](#8-usage-guide)
- [9. REST API Reference](#9-rest-api-reference)
- [10. Current Limitations](#10-current-limitations)
- [11. Roadmap & Future Scope](#11-roadmap--future-scope)

---

## 1. Use Case & Problem Statement

### The Problem with Traditional RAG
Modern enterprise search and Retrieval-Augmented Generation (RAG) systems routinely suffer from **semantic fragmentation** and **context loss**. Traditional RAG pipelines ingest PDF or DOCX files by arbitrarily splitting them into fixed-length character chunks (e.g., 500–1,000 characters) with token overlap.

This brute-force chunking strategy introduces significant flaws:
1. **Broken Semantic Boundaries** — Paragraphs, tables, and logical sections are cut in half, leaving isolated chunks devoid of their parent context.
2. **Heading & Hierarchy Blindness** — Chunks lack awareness of document titles, section headers, or structural depth.
3. **Vector Database Bloat** — Storing embeddings for hundreds of micro-chunks per document inflates index sizes, slows down semantic search, and increases memory costs.
4. **Poor Multi-Hop Reasoning** — Retrieving an isolated chunk often fails to answer enterprise queries that depend on document-wide context, departmental metadata, or executive summaries.

### The MetaVault Solution
**MetaVault** is an enterprise-grade **Metadata-as-a-Service (MaaS)** platform that enriches unstructured documents (**PDF** and **DOCX**) with structured metadata, deep semantic understanding, and layout-aware hierarchy extraction.

Unlike standard RAG systems, **MetaVault does not perform document retrieval or question-answering over raw document chunks.** Instead, it transforms raw documents into high-quality, structured knowledge artifacts designed to be consumed by downstream search engines, enterprise knowledge bases, or autonomous AI agents:

- **Relational Metadata** — Stores full structural hierarchies, section-level summaries, keywords, and extracted named entities in PostgreSQL.
- **Single Global Vector** — Stores exactly one high-density semantic embedding per document in Qdrant, generated from synthesized metadata rather than raw text noise.

```
       [ Unstructured Enterprise PDF / DOCX ]
                         │
                         ▼
        ┌──────────────────────────────────┐
        │       MetaVault Platform         │
        │  (Layout Parsing • H1-H3 Engine  │
        │  • GLiNER NLP • GPT-4.1 Synthesis)│
        └─────────────────┬────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
    [ PostgreSQL ]                  [ Qdrant DB ]
  • Structural Trees              • 1 Vector / Document
  • Section Summaries             • Semantic Metadata
  • Extracted Entities              Payload Index
```

---

## 2. Core Design Principles

| Principle | Traditional RAG | MetaVault (MaaS) |
| :--- | :--- | :--- |
| **Primary Goal** | Direct snippet retrieval & QA | Rich metadata generation & structured knowledge extraction |
| **Document Chunking** | Fixed-size character/token windows | Logical structure-aware sections (H1 → H1 boundaries) |
| **Embedding Strategy** | Hundreds of embeddings per document | Exactly 1 embedding per document (synthesized from rich metadata) |
| **AI Architecture** | Basic text-to-embedding models | Hybrid AI: deterministic layout scoring + GLiNER NLP + GPT-4.1 |
| **Index Size & Speed** | Bloated vector index, slower queries | Ultra-compact vector footprint, sub-millisecond retrieval |

1. **Metadata-First Architecture** — Retrieval systems should consume clean, structured metadata rather than raw text.
2. **One Embedding Per Document** — By embedding the aggregated metadata (summary, topic, keywords, key entities, section titles), vector storage requirements drop by 95%+ while semantic recall accuracy increases.
3. **Structure-Aware Processing** — Documents are segmented along natural H1/H2/H3 boundaries. Each section represents a complete logical idea.
4. **Hybrid AI Engine** — Combines deterministic layout parsing (Docling) and feature scoring with state-of-the-art NLP models (GLiNER for zero-shot entity extraction + OpenAI GPT-4.1 for semantic summarization and taxonomy classification).

---

## 3. Current Features Implemented

- **Layout-Preserving Document Parsing** — Full support for `.pdf` and `.docx` enterprise documents using Docling. Captures reading order, typography (boldness, font sizes), whitespace layout, and table-of-contents cues.
- **Intelligent Header Extraction & Candidate Scoring** — Evaluates text blocks against layout features (font scale, numbering patterns, spacing, TOC appearance) to reject false positives (tables, figures, page headers/footers). Dynamically builds an accurate H1 → H2 → H3 structural hierarchy.
- **Logical Section Builder** — Automatically groups content under Level-1 (H1) headers, retaining all nested H2/H3 sub-headings and text within their parent section.
- **Dual-Model Section Analysis**
  - **GLiNER** — Extracts high-precision named entities (Client, Organization, Person, Product, Technology, Location).
  - **OpenAI GPT-4.1** — Generates concise section summaries, topics, keyword tags, and semantic section classifications.
- **Document-Level Metadata Aggregation & Deduplication** — Merges section-level summaries into a coherent executive summary, and classifies document topic, doc type (e.g., Proposal, Handbook, Report), and responsible department.
- **Single-Vector Knowledge Indexing** — Synthesizes document summary, section titles, entities, and keywords into a dense vector embedding, indexed in Qdrant with a lightweight filtering payload. Full section trees, headers, and entity graphs are stored in PostgreSQL.
- **Enterprise REST API (FastAPI)** — Fully asynchronous HTTP API providing document listing, hierarchical tree retrieval, section breakdown, and vector similarity search.
- **UI** — Responsive frontend served via `/app`, featuring real-time document search, hierarchical tree visualizer, entity badges, and semantic agent retrieval previews.
- **Containerized Deployment** — Production-ready `Dockerfile` and `docker-compose.yml` orchestrating API, PostgreSQL, and Qdrant containers with automated health checks.

---

## 4. Architecture & Pipeline

```
                                    PDF / DOCX
                                         │
                                         ▼
                             ┌───────────────────────┐
                             │    Parser (Docling)   │
                             └───────────┬───────────┘
                                         │
                                         ▼
                             ┌───────────────────────┐
                             │   Header Extractor    │
                             │  (Candidate Scoring)  │
                             └───────────┬───────────┘
                                         │
                                         ▼
                             ┌───────────────────────┐
                             │    Section Builder    │
                             │    (H1 Level Units)   │
                             └───────────┬───────────┘
                                         │
                                         ▼
                             ┌───────────────────────┐
                             │    Section Analyzer   │
                             └──────┬─────────┬──────┘
                                    │         │
                 ┌──────────────────▼─┐     ┌─▼──────────────────┐
                 │       GLiNER       │     │     GPT-4.1        │
                 │ (Entity Extractor) │     │ (Summary & Topics) │
                 └──────────────────┬─┘     └─┬──────────────────┘
                                    │         │
                                    ▼         ▼
                             ┌───────────────────────┐
                             │  Metadata Aggregator  │
                             └───────────┬───────────┘
                                         │
                                         ▼
                             ┌───────────────────────┐
                             │  Embedding Generator  │
                             └──────┬─────────┬──────┘
                                    │         │
                   ┌────────────────▼┐       ┌▼────────────────┐
                   │   PostgreSQL    │       │     Qdrant      │
                   │  (Relational &  │       │  (Single Global │
                   │  Hierarchy Tree)│       │   Vector Index) │
                   └─────────────────┘       └─────────────────┘
```

**End-to-End Processing Workflow:**
1. **Parsing** (`processing/parser.py`) — Docling extracts raw text and layout formatting without altering document sequence or losing table/font properties.
2. **Header Extraction** (`processing/header_extractor.py`) — Scans text blocks for typographic signals; scores and validates true section headers (H1, H2, H3).
3. **Section Building** (`processing/section_builder.py`) — Combines validated headings and paragraphs into discrete H1-bounded logical sections.
4. **Section Analysis** (`processing/section_analyzer.py`) — Extracts localized entities via GLiNER and generates section summaries, topics, and keywords via OpenAI.
5. **Metadata Aggregation** (`processing/metadata_aggregator.py`) — Synthesizes section outputs into global document metadata (executive summary, primary topic, department, document type, deduplicated entities).
6. **Storage & Vector Indexing** (`storage/` & `embedding_generator.py`) — Persists normalized hierarchies to PostgreSQL and generates/inserts the single document embedding into Qdrant.

---

## 5. Technology Stack

| Layer | Technologies | Purpose |
| :--- | :--- | :--- |
| **Document Parser** | Docling | High-fidelity PDF/DOCX structural and layout parsing |
| **NLP / Entity Engine** | GLiNER | Zero-shot Named Entity Recognition (NER) |
| **LLM Reasoning** | OpenAI GPT-4.1 / Python OpenAI SDK | Semantic summarization, keyword tagging, and classification |
| **Embeddings** | OpenAI Embeddings API (`text-embedding-3-small` / `large`) | High-density vector generation |
| **Relational Storage** | PostgreSQL 15 (`psycopg2-binary`) | Structured metadata, hierarchical trees, section logs |
| **Vector Database** | Qdrant | Fast similarity search over single-document vector embeddings |
| **API Server** | FastAPI + Uvicorn | Asynchronous HTTP REST API & static frontend server |
| **Web Frontend** | Vanilla HTML5, CSS3, JavaScript | Interactive Document Explorer & Hierarchy Visualizer |
| **DevOps** | Docker & Docker Compose | Containerized multi-service orchestration & deployment |
| **Testing** | PyTest | Automated integration and unit testing suite |

---

## 6. Project Structure

```
metavault/
│
├── .env.example                # Sample environment configuration
├── .gitignore                  # Git exclude rules
├── Dockerfile                  # API service containerization
├── docker-compose.yml          # Multi-container cluster (API + Postgres + Qdrant)
├── requirements.txt            # Python dependencies
├── run_pipeline.py             # CLI entrypoint for batch document ingestion
├── api.py                      # FastAPI REST application & UI server
├── embedding_generator.py      # Vector embedding generator for metadata
├── utils.py                    # Helper utilities (normalization, deduplication)
│
├── processing/                 # Core ingestion & analysis intelligence
│   ├── parser.py               # Docling document parser wrapper
│   ├── header_extractor.py     # Candidate header scoring & hierarchy engine
│   ├── section_builder.py      # H1-level logical section aggregator
│   ├── section_analyzer.py     # GLiNER + GPT section enrichment
│   ├── metadata_aggregator.py  # Document-level synthesis & deduplication
│   └── document_processor.py   # Pipeline orchestration controller
│
├── storage/                    # Persistence layer adapters
│   ├── postgres.py             # Relational schema management & queries
│   └── qdrant.py               # Qdrant collection setup & similarity search
│
├── frontend/                   # Interactive Web Explorer UI
│   ├── index.html              # Main single-page application shell
│   ├── styles.css              # Rich modern UI styling & responsive design
│   └── app.js                  # Frontend state, API clients & tree rendering
│
└── tests/                      # Automated test suite
    ├── test_header_extractor.py
    ├── test_section_builder.py
    └── test_metadata_aggregator.py
```

---

## 7. Setup & Deployment Instructions

### Prerequisites
- **Python:** 3.10 or higher (if running locally without Docker)
- **Docker:** Docker Desktop or Docker Engine + Docker Compose v2 (recommended)
- **GitHub Personal Access Token (PAT)** 

### Option A: One-Command Deployment (Docker Compose) — Recommended

The easiest way to launch the entire MetaVault stack (FastAPI backend, PostgreSQL database, Qdrant vector engine, and interactive Web Explorer UI) is using Docker Compose.

1. **Configure environment variables:**
   Create a `.env` file in the root directory (or copy from `.env.example`):
   ```env
   GITHUB_TOKEN= github_pat_11CGYKYPAxxxxxxxxxzsAhK_WdMW0xxxxxxx0sfUAjd1q
   GITHUB_MODELS_BASE_URL= https://models.inference.ai.azure.com
   DATABASE_URL= postgresql://postgres:postgres@db:5432/metavault
   QDRANT_URL= http://qdrant:6333
   ```

2. **Launch the container cluster:**
   ```bash
   docker-compose up --build -d
   ```
   This will provision a PostgreSQL 15 container (`5432`), a Qdrant vector database container (`6333`), and build/start the MetaVault API server and frontend (`8000`).

3. **Verify deployment:**
   - Interactive Web Explorer UI: `http://localhost:8000/app`
   - Interactive Swagger API Docs: `http://localhost:8000/docs`
   - Qdrant Web Dashboard: `http://localhost:6333/dashboard`

### Option B: Manual Local Installation

If you prefer running the Python services directly on your host machine:

1. **Set up a virtual environment:**
   ```bash
   python -m venv venv
   # On Windows:
   venv\Scripts\activate
   # On macOS/Linux:
   source venv/bin/activate
   ```

2. **Install Python dependencies:**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

3. **Start local PostgreSQL & Qdrant instances:**
   Ensure PostgreSQL is running locally on port `5432` with a database named `metavault`, and Qdrant is running on port `6333` (via Docker or standalone binary):
   ```bash
   # Quick Qdrant Docker launch if needed:
   docker run -p 6333:6333 -d qdrant/qdrant
   ```

4. **Configure local `.env` file:**
   ```env
   OPENAI_API_KEY=sk-your-openai-api-key-here
   DATABASE_URL=postgresql://postgres:postgres@localhost:5432/metavault
   QDRANT_URL=http://localhost:6333
   ```

5. **Start the FastAPI server:**
   ```bash
   python api.py
   ```
   The API will be available at `http://localhost:8000`.

---

## 8. Usage Guide

### Running the Ingestion Pipeline (CLI)
To ingest enterprise PDF or DOCX documents into MetaVault from the command line, use `run_pipeline.py`. You can pass either a single file or a folder containing multiple documents:

```bash
# Ingest a single PDF or DOCX file:
python run_pipeline.py ./path/to/Employee_Handbook_2026.docx

# Ingest an entire directory of documents:
python run_pipeline.py ./documents_folder
```

**What happens during execution?**
1. The document is parsed and layout features are scored.
2. Section headers (H1–H3) are validated and sections are built.
3. GLiNER extracts entities (Person, Organization, Technology, etc.).
4. GPT-4.1 generates summaries, topics, and classifications.
5. Normalized metadata is committed to PostgreSQL and the single semantic vector is stored in Qdrant.

### Accessing the Interactive Web Explorer & UI
After starting the API server (via Docker Compose or locally), navigate to `http://localhost:8000/app` in your web browser.

- **Knowledge Base Sidebar** — Browse all ingested enterprise documents sorted by ingestion time.
- **Hierarchical Structure Visualizer** — Click any document to inspect its inferred H1 → H2 → H3 outline, section summaries, and department/topic badges.
- **Agent Semantic Search** — Use the real-time search bar to test vector similarity queries against your Qdrant index and retrieve precise document sections.

---

## 9. REST API Reference

| HTTP Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/documents` | Lists all processed documents with high-level summaries, departments, and topics. |
| `GET` | `/api/documents/{document_id}/sections` | Returns all H1 logical sections for a specific document with summaries and section types. |
| `GET` | `/api/documents/{document_id}/hierarchy` | Returns a nested JSON tree representing the document's H1–H3 heading hierarchy. |
| `POST` | `/api/search` | Semantic vector similarity search over Qdrant (`top_k` results). |

**Example: Semantic Search Request**
```bash
curl -X POST "http://localhost:8000/api/search" \
     -H "Content-Type: application/json" \
     -d '{"query": "What are the trade secret policies for engineering employees?", "top_k": 3}'
```

**Example Response**
```json
[
  {
    "document_id": "8a2e4b3c-1d2f-4e6a-9b1a-2e3f4a5b6c7d",
    "filename": "Employee_Handbook_2026.docx",
    "section_title": "4. Intellectual Property & Trade Secrets",
    "section_type": "Policy & Compliance",
    "summary": "Details mandatory confidentiality agreements and trade secret protection rules for engineering personnel.",
    "score": 0.912
  }
]
```

---

## 10. Current Limitations

1. **Supported File Formats** — The current parser is optimized specifically for PDF and DOCX files. Other office formats (`.pptx`, `.xlsx`, `.html`) are not yet supported.
2. **Synchronous CLI Execution** — Document ingestion via `run_pipeline.py` processes files sequentially. Very large PDFs (100+ pages) may take 30–60 seconds depending on LLM response times.
3. **External OpenAI Dependency** — The section analyzer and embedding generator currently require an active OpenAI API key and internet connectivity.
4. **OCR Quality Dependence** — For scanned image PDFs, layout parsing fidelity depends on underlying OCR clarity.

---

## 11. Roadmap & Future Scope

- [ ] **Asynchronous Background Job Queue** — Integrate Celery + Redis for distributed, non-blocking batch document processing and webhook notifications.
- [ ] **Multi-Parser & Format Expansion** — Add native support for `.pptx` (presentations), `.xlsx` (financial sheets), and `.html`/`.md` archives.
- [ ] **Enterprise Knowledge Graph Generation** — Automatically extract cross-document relationship triples (Subject → Predicate → Object) from GLiNER entity outputs and store them in Neo4j.
- [ ] **Metadata Versioning & Audit Logs** — Track changes to document revisions and maintain historical metadata diffs in PostgreSQL.
- [ ] **Role-Based Access Control (RBAC)** — Add enterprise authentication (OAuth2 / JWT) and department-level document filtering.
