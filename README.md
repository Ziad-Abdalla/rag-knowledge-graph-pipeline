# RAG Knowledge Graph Pipeline

A production **Retrieval-Augmented Generation** system that transforms PDF documents into a queryable knowledge graph. Built at [AiCentive GmbH](https://aicentive.com) as a multi-tenant document intelligence platform for enterprise customers.

> **Note:** This is an architecture showcase of a private production system. Source code is proprietary.

---

## What It Does

Upload a PDF → the system extracts entities and relationships using LLMs → builds a knowledge graph → answers natural language questions using hybrid search over the graph.

**Example:** Upload a 50-page company report. Ask "Who founded TechNova and where are they located?" → The system finds the relevant entities and relationships in the graph and generates a sourced answer.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATA INGESTION PIPELINE                    │
│                                                                 │
│  PDF  ──→  Docling  ──→  Chunking  ──→  LLM Extraction  ──→   │
│            (PDF→MD)      (semantic)     (entities + rels)       │
│                                                                 │
│  ──→  FAISS Dedup  ──→  Graph Insertion  ──→  Vector Index     │
│       (clustering)      (UPSERT)              (FAISS/Neo4j)    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │      Graph Database      │
              │                          │
              │  Neo4j → NebulaGraph     │
              │  (migration completed)   │
              │                          │
              │  Per-tenant graph spaces  │
              │  + Per-tenant FAISS index │
              └────────────┬─────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     QUERY & RETRIEVAL                           │
│                                                                 │
│  Question  ──→  Key Term Extraction (Mistral Nemo 12B)         │
│            ──→  Hybrid Search (keyword + vector)                │
│            ──→  Subgraph Expansion (3 levels)                   │
│            ──→  Answer Generation (LLM)                         │
│            ──→  Natural language response                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      REST API (FastAPI)                         │
│                                                                 │
│  POST /ask          → Question answering via LangGraph          │
│  POST /upload       → PDF upload + background processing        │
│  GET  /documents    → List documents by tenant                  │
│  DELETE /documents  → Remove doc + extracted entities            │
│                                                                 │
│  API key auth · Rate limiting · Document isolation              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND (Angular 17)                        │
│                                                                 │
│  Chat tab: Ask questions about the knowledge base               │
│  Admin tab: Upload PDFs, configure pipeline, manage documents   │
│  Auth: API key stored in browser, auto-attached via interceptor │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Deep Dive

### 1. PDF Processing
**Docling** converts PDFs to Markdown, preserving document structure. For PDFs with complex tables and images, an optional **local vision model** (Qwen2-VL 7B via Ollama) augments the text extraction — images never leave the machine (GDPR advantage).

### 2. Chunking
The original system split only on `##` headers — if a document had no headers, the entire text became one chunk. Upgraded to **semantic chunking** that detects topic shifts using embeddings, producing meaningful segments regardless of document formatting.

### 3. Entity & Relationship Extraction
Each chunk is sent to an LLM (Llama 3.3 70B on IONOS AI Hub) which extracts structured entities and relationships. Output is validated through **Pydantic schemas** with retry logic — malformed JSON triggers up to 3 retries before gracefully skipping.

Chunks are processed in parallel using `asyncio.gather()` with configurable concurrency via semaphore.

### 4. Deduplication
Extracted entities are deduplicated using **FAISS nearest-neighbor search + Union-Find clustering**. Entities with embedding similarity above a configurable threshold (default 0.85) are merged, keeping the richest description.

### 5. Graph Storage
Entities become nodes, relationships become edges. The schema uses dynamic types — the LLM decides labels at extraction time (Person, Organization, Regulation, etc.).

**Multi-tenancy:** Each customer gets an isolated **NebulaGraph graph space** — no data mixing possible, no query filtering needed. Migrated from Neo4j's shared-database approach which didn't scale (151-node subgraph expansions exceeded LLM token limits with just 3 documents).

### 6. Retrieval
Questions go through a **LangGraph state machine**:
1. **Key term extraction** — A smaller LLM (Mistral Nemo 12B) pulls search terms
2. **Hybrid search** — Keyword matching on node properties + vector similarity via FAISS
3. **Subgraph expansion** — 3-level graph traversal finds connected context
4. **Answer generation** — LLM synthesizes an answer from the graph context

---

## Key Technical Decisions

| Decision | Why |
|----------|-----|
| **Knowledge graph over vector store** | Entity relationships are first-class citizens — "how is X connected to Y?" is a single graph traversal, not a similarity guess |
| **IONOS AI Hub (European LLM hosting)** | All data stays on European servers. OpenAI-compatible API, so LangChain's `ChatOpenAI` works with a custom base URL |
| **NebulaGraph over Neo4j** | Native multi-tenancy via graph spaces. Neo4j's `projectId` filtering scanned the entire graph on every query |
| **Per-tenant FAISS indexes** | NebulaGraph Community lacks vector search (Enterprise-only). FAISS is free, fast, and already in the project |
| **Database abstraction layer** | `GraphService` interface lets the system swap databases by writing a new implementation. Neo4j kept as fallback during migration |
| **API keys over JWT** | Primary consumers are other applications integrating the API, not humans logging in. Each key maps to a `project_id` for automatic tenant scoping |
| **PydanticOutputParser over structured output** | Tested both — structured output was slower (1m 15s vs 1m 3s) and extracted fewer entities (32 vs 43) on Llama 70B |
| **Source-based cleanup over incremental** | Deduplication merges entities across chunks, making chunk-level differential processing impractical. Full re-process takes ~1 min anyway |

---

## Performance & Metrics

### Pipeline Speed Optimization: 87x Improvement

| Configuration | Time | Entities | Retries |
|--------------|------|----------|---------|
| Llama 405B, sequential | 87 min | 46 | 2 |
| Llama 70B, sequential | 2m 4s | 43 | 0 |
| **Llama 70B, parallel (c=3)** | **1m 3s** | **43** | **0** |
| Llama 70B, parallel + structured output | 1m 15s | 32 | 0 |

**Key findings:**
- Switching from 405B → 70B: nearly identical entity counts, zero retries, 42x faster
- Adding parallel processing: wall time halved (2m 4s → 1m 3s)
- 405B timing was unpredictable (~600s timeouts from IONOS queue) while 70B was consistent (4–35s per call)
- Structured output extracted fewer entities — the constrained JSON schema was too restrictive

### Code Quality: 32 Issues Fixed

Inherited a codebase with 32 issues across 4 repositories. All resolved:

- **10 HIGH severity** — Hardcoded paths, vector dimension mismatch (384 vs 1024), deduplication threshold set to impossible value (>1.0), no error handling in extraction, retired model references, unpinned dependencies, embeddings leaking into LLM prompts (314K chars)
- **11 MEDIUM severity** — Cypher injection risk, Neo4j password in git, no logging framework, misleading file names, module-level initialization crashes
- **4 LOW severity** — Dead code, confusing folder names, missing documentation

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **PDF Processing** | Docling, optional Qwen2-VL 7B (local vision via Ollama) |
| **LLM Provider** | IONOS AI Hub (Llama 3.3 70B extraction, Mistral Nemo 12B key terms) |
| **Orchestration** | LangChain, LangGraph (state machine) |
| **Graph Database** | NebulaGraph (migrated from Neo4j) — per-tenant graph spaces |
| **Vector Search** | FAISS (per-tenant indexes) + SentenceTransformer embeddings |
| **Deduplication** | FAISS nearest-neighbor + Union-Find clustering |
| **API** | FastAPI, API key auth, slowapi rate limiting |
| **Frontend** | Angular 17, Angular Material, HttpClient + interceptors |
| **Infrastructure** | Docker Compose, planned Kubernetes deployment |
| **Language** | Python (backend), TypeScript (frontend) |

---

## Architecture Highlights

### Multi-Tenancy Evolution
Started with `projectId` property filtering on a shared Neo4j graph → migrated to **NebulaGraph with isolated graph spaces per tenant**. Each tenant also gets their own FAISS vector index. The `GraphService` abstraction layer made the swap possible without changing business logic.

### Database Abstraction
```
GraphService (abstract interface)
  ├── Neo4jGraphService    — Cypher queries, APOC procedures
  └── NebulaGraphService   — nGQL queries, per-tenant FAISS
```
Factory pattern reads `GRAPH_BACKEND` env var → creates the right implementation. Both coexist during migration.

### Security & Isolation
- API keys → automatic tenant scoping (key maps to `project_id`)
- Document isolation: entities tagged with source, queries filtered by tenant
- Rate limiting: per-API-key (not per-IP) because integrating backends share IPs
- All LLM calls stay on European infrastructure (IONOS AI Hub)
- Local vision processing: images never leave the machine

### Pipeline as Background Service
PDF upload triggers asynchronous processing. The API returns immediately with a document ID for status polling. SHA-256 hashing prevents reprocessing identical files. Re-uploads trigger source-based cleanup (delete old entities, then re-extract).

---

## Project Scope

I inherited a partially working codebase and turned it into a production-ready multi-tenant system:

- Assessed and fixed 32 code quality issues across 4 repositories
- Built the complete REST API layer (FastAPI) with auth, rate limiting, and document management
- Rewired the frontend from Socket.IO to REST, added admin interface
- Ran controlled experiments comparing extraction models, parallelism, and output strategies
- Migrated from Neo4j to NebulaGraph for native multi-tenancy
- Implemented database abstraction layer for future-proof database swaps
- Achieved 87x pipeline speed improvement through model optimization + parallelism

---

## Repositories

| Repository | Purpose |
|-----------|---------|
| `rag-pipeline-guc` | Data ingestion — PDF processing, entity extraction, graph insertion |
| `rag-chatbot-guc` | Query engine — FastAPI server, hybrid search, LangGraph orchestration |
| `sio-chatbot-widget` | Frontend — Angular 17 chat + admin interface |

All repositories are private (proprietary to AiCentive GmbH).
