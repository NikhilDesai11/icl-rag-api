# icl-rag-api

## Overview

A production-grade RAG (Retrieval-Augmented Generation) API built with FastAPI and ChromaDB, backed by an automated CI pipeline with GitHub Actions. The API answers natural language questions by retrieving relevant context from a local knowledge base and generating responses through a local LLM with Ollama.

Automated semantic tests catch data-quality regressions on every push, which helps keep retrieval behavior stable as documents and application logic change.

## Features

- RAG pipeline with retrieval plus generation for question answering.
- ChromaDB vector store for semantic search over local text documents.
- Ollama + TinyLlama for local LLM inference.
- Mock LLM mode for deterministic CI testing.
- Semantic tests that validate key concepts in answers.
- GitHub Actions workflow that runs on changes to app and knowledge files.
- Multi-document support through the `docs/` folder and `embed_docs.py` workflow.

## Tech Stack

| Layer | Tool |
|-------|------|
| API Framework | FastAPI |
| Web Server | Uvicorn |
| Vector Store | ChromaDB |
| LLM Runtime | Ollama with TinyLlama |
| CI | GitHub Actions |
| Language | Python 3.11-3.13 |

## Setup

**Prerequisites**
- Python 3.11-3.13
- [Ollama](https://ollama.com/) installed and running locally
- Git and a GitHub account for CI

### 1. Clone the repo

```bash
git clone git@github.com:<your-username>/icl-rag-api.git
cd icl-rag-api
```

### 2. Create and activate a virtual environment

```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install fastapi uvicorn chromadb ollama requests
```

### 4. Pull the TinyLlama model

```bash
ollama pull tinyllama
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `USE_MOCK_LLM` | `0` | Set to `1` to skip Ollama and return retrieved context directly for deterministic CI tests. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama server URL. Override this when Ollama runs on another host. |

## Run Locally

### 1. Build the vector database

```bash
python embed_docs.py
```

### 2. Start the API

```bash
# Production mode
uvicorn app:app --reload

# Mock mode
USE_MOCK_LLM=1 uvicorn app:app --reload
```

API runs at `http://127.0.0.1:8000`.

### 3. Test a query

```bash
# macOS/Linux
curl -X POST "http://127.0.0.1:8000/query" -G --data-urlencode "q=What is Kubernetes?"

# Windows PowerShell
Invoke-WebRequest -Uri "http://127.0.0.1:8000/query?q=What%20is%20Kubernetes%3F" -Method POST
```

Example response:

```json
{"answer": "Kubernetes is a container platform used to manage containers at scale."}
```

## Scripts

| Script | Purpose |
|--------|---------|
| `embed_docs.py` | Reads all `.txt` files in `docs/`, clears old embeddings, and rebuilds the ChromaDB store. |
| `embed.py` | Legacy single-file embedder for `k8s.txt`. |
| `semantic_test.py` | Sends test queries to the running API and checks responses for expected keywords. |

## API / Architecture

### `POST /query`

Queries the RAG system with a natural language question.

| Param | Type | Description |
|-------|------|-------------|
| `q` | `string` | The question to ask. |

**Example request**

```bash
POST /query?q=What is Kubernetes?
```

**Example response**

```json
{
  "answer": "Kubernetes is a container platform used to manage containers at scale."
}
```

**Request flow**
1. The API queries ChromaDB for the top matching document.
2. In production mode, the retrieved context and question are sent to TinyLlama through Ollama.
3. In mock mode, the API returns the retrieved context directly for deterministic testing.

## Folder Structure

```text
icl-rag-api/
├── .github/
│   └── workflows/
│       └── ci.yml
├── docs/
│   ├── k8s.txt
│   └── nextwork.txt
├── app.py
├── embed.py
├── embed_docs.py
├── semantic_test.py
├── .gitignore
└── README.md
```

`db/` and `venv/` are ignored and generated locally.

## Deployment

Set `OLLAMA_HOST` when Ollama is not running on the same machine as the API.

```bash
OLLAMA_HOST=http://<ollama-host>:11434 uvicorn app:app
```

## Contributing

1. Fork the repository.
2. Create a branch: `git checkout -b feature/your-change`
3. Make the change and update `semantic_test.py` when knowledge documents change.
4. Open a pull request and let CI validate it.

Keep commits small and descriptive.

## License

MIT