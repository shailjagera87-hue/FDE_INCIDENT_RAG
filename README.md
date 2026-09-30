# FDE_INCIDENT_RAG
# CloudOps Sentinel — Enterprise Incident Response Self-RAG Copilot

A production-oriented **Self-RAG application** for cloud operations and incident response. It searches private operational knowledge in Pinecone first, self-grades retrieved evidence, corrects weak retrieval and generation, and uses internet search only when the private knowledge base is insufficient.

The project also demonstrates **LangGraph SQLite persistence memory**, allowing follow-up questions to reuse the same incident context through a stable `thread_id`.

## Architecture

```text
User Incident / Follow-up
        ↓
LangGraph SQLite Memory
        ↓
Contextualize Follow-up
        ↓
Decide Retrieval
        ↓
Private Pinecone Knowledge Base
        ↓
Grade Retrieved Documents
   ┌────┴───────────────┐
Relevant             Weak / Missing
   ↓                      ↓
Generate              Rewrite Query
   ↓                      ↓
IsSUP              Retry Private KB
   ↓                      ↓
Revise if needed   Internet Search Fallback
   ↓                      ↓
IsUSE              Grade Web Evidence
   ↓                      ↓
Final Answer ← Generate → IsSUP → IsUSE
        ↓
SQLite Checkpoint / Memory
```

## Tech Stack

* **LangGraph** — Self-RAG workflow, conditional routing, and persistent thread state
* **OpenAI `gpt-5-mini`** — routing, query rewriting, generation, relevance grading, IsSUP, and IsUSE
* **OpenAI `text-embedding-3-large`** — embeddings for private operational documents
* **Pinecone** — private runbooks, SOPs, and postmortem knowledge base
* **Tavily** — controlled internet-search fallback
* **SQLite** — LangGraph persistence memory and audit database
* **FastAPI** — application/API backend
* **HTML/CSS/JavaScript** — incident-command-center UI with document upload
* **Docker** — application containerization and deployment packaging

## Project Structure

```text
CloudOps-Sentinel-Enterprise-Incident-Response-Self-RAG-Copilot/
├── app.py
├── data_ingestion.py          # Single entry point for building the Pinecone KB
├── src/
│   ├── config.py
│   ├── db.py
│   ├── ingestion.py
│   ├── models.py
│   ├── self_rag.py
│   └── vectorstore.py
├── documents/                 # Initial private knowledge documents
│   ├── checkout-api-runbook.md
│   ├── payments-high-cpu-runbook.md
│   └── deployment-rollback-sop.md
├── templates/
│   └── index.html
├── static/
│   ├── styles.css
│   └── app.js
├── uploads/                   # Documents uploaded through the UI
├── data/                      # SQLite persistence files
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

## 1. Setup

Create a Python virtual environment:

```bash
python -m venv venv
```

### Windows

```bash
venv\Scripts\activate
```

### macOS/Linux

```bash
source venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Copy `.env.example` to `.env` and configure:

```env
OPENAI_API_KEY=...
PINECONE_API_KEY=...
TAVILY_API_KEY=...
```

The default configuration is:

```env
OPENAI_MODEL=gpt-5-mini
EMBEDDING_MODEL=text-embedding-3-large
EMBEDDING_DIMENSION=3072
PINECONE_INDEX_NAME=cloudops-sentinel-openai-self-rag
PINECONE_NAMESPACE=incident-runbooks
```

> The project uses a dedicated Pinecone index configuration to ensure that the embedding model and vector dimension remain consistent across ingestion and retrieval.

## 2. Build the Pinecone Knowledge Base

Place your initial PDF, TXT, Markdown, or DOCX documents inside `documents/`, then run:

```bash
python data_ingestion.py
```

The ingestion pipeline performs:

```text
Read .env
  ↓
Create / validate Pinecone index
  ↓
Load ./documents
  ↓
Split documents into chunks
  ↓
Generate OpenAI embeddings
  ↓
Upsert vectors to Pinecone
  ↓
Configured namespace is ready
```

The ingestion pipeline uses stable chunk IDs, allowing repeated ingestion to update matching vectors rather than generating a new random ID for every chunk.

## 3. Run the Application

Run with:

```bash
python app.py
```

or:

```bash
uvicorn app:app --reload
```

Open:

```text
http://127.0.0.1:8000
```

## Add New Documents from the UI

The **Runbook Vault** accepts:

* PDF
* TXT
* Markdown
* DOCX

When **Add to Knowledge Base** is selected, the backend uses the same ingestion layer and OpenAI embedding model as `data_ingestion.py`.

```text
New Document
      ↓
FastAPI /api/upload
      ↓
Document Loader + Chunking
      ↓
OpenAI Embeddings
      ↓
Existing Pinecone Index
      ↓
Existing Namespace
```

New operational knowledge therefore becomes available to the retrieval pipeline without rebuilding the application.

## LangGraph SQLite Persistence Memory

Each browser incident session has a persistent `thread_id`. The LangGraph workflow is compiled with a SQLite checkpointer and stores checkpoints in:

```text
data/langgraph_memory.sqlite
```

Example conversation:

**Question 1**

> Our checkout API is returning 502 errors after deployment. What should I check first?

**Question 2**

> What should I check next if that doesn't work?

For the second message, the `contextualize` node uses the persisted incident context and converts the follow-up into a standalone operational question before continuing through the Self-RAG workflow.

SQLite provides lightweight local persistence for the application. The persistence layer is designed to be replaceable, allowing a production deployment to use PostgreSQL or another supported production database when higher concurrency and distributed application instances are required.

## Recommended Application Flow

### Scenario A — Existing Pinecone Knowledge

Ask:

> Our checkout API is returning 502 errors after deployment. What should the on-call engineer check first?

Expected behavior: **Private Runbooks** are retrieved and used as the primary evidence source.

### Scenario B — Persistent Memory

Immediately ask:

> What should I check next if that does not work?

The same `thread_id` reuses the previous incident context.

### Scenario C — Upload New Operational Knowledge

Upload a new PDF, Markdown, or DOCX document through the UI.

The application reports how many chunks were added to the existing namespace. A question can then be asked against information contained in the newly uploaded document.

### Scenario D — Internet Search Fallback

Ask about a technical issue that is not sufficiently covered by the private runbooks.

Self-RAG first evaluates the private knowledge base, rewrites the query when necessary, and transitions to **Internet Search** when internal evidence remains insufficient.

## Important Configuration Rule

The application and ingestion script both import `src/vectorstore.py`. This ensures that ingestion and runtime retrieval consistently use the same:

* OpenAI embedding model
* Embedding dimension
* Pinecone index
* Pinecone namespace

This prevents a common RAG configuration problem where offline ingestion and runtime retrieval use different embeddings, dimensions, indexes, or namespaces.

## Key Engineering Concepts

This project demonstrates:

* **Self-RAG** with retrieval evaluation and corrective retries
* **Private-first retrieval** for operational knowledge
* **Controlled web-search fallback**
* **Evidence-grounded generation**
* **IsSUP answer verification**
* **IsUSE usefulness verification**
* **LangGraph conditional workflows**
* **Persistent conversational state**
* **Vector search with Pinecone**
* **Document ingestion and dynamic knowledge-base updates**
* **FastAPI API development**
* **Docker-based application packaging**
* **Production-oriented cloud application architecture**
