# icl-rag-api

## Overview

icl-rag-api is a RAG (Retrieval-Augmented Generation) API built with FastAPI, ChromaDB, and Ollama. It answers natural language questions by retrieving relevant context from local documents and generating responses with a local LLM.

The project is set up with GitHub Actions so retrieval quality is tested automatically on every meaningful change. It also includes a mock LLM mode that makes semantic tests deterministic and reliable in CI.

## Features

- FastAPI-based RAG API for question answering
- ChromaDB-backed document retrieval
- Ollama + TinyLlama for local LLM inference
- Mock LLM mode for deterministic CI testing
- Semantic regression tests for retrieval quality
- GitHub Actions workflow for automated CI
- Multi-document ingestion from the `docs/` folder
- Configurable Ollama host through environment variables

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

### Prerequisites

- Python 3.11-3.13
- [Ollama](https://ollama.com/) installed and running locally
- Git and a GitHub account

### Clone the repository

```bash
git clone git@github.com:<your-username>/icl-rag-api.git
cd icl-rag-api
```

### Create and activate a virtual environment

```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

### Install dependencies

```bash
pip install fastapi uvicorn chromadb ollama requests
```

### Pull the TinyLlama model

```bash
ollama pull tinyllama
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `USE_MOCK_LLM` | `0` | Set to `1` to bypass Ollama and return retrieved context directly. Used for deterministic testing in CI. |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama server URL for local or remote inference. |

## Run Locally

### Build embeddings

```bash
python embed_docs.py
```

### Start the API

```bash
# Production mode
uvicorn app:app --reload
```

```bash
# Mock mode
USE_MOCK_LLM=1 uvicorn app:app --reload
```

API runs at:

```text
http://127.0.0.1:8000
```

### Test a query

```bash
# macOS/Linux
curl -X POST "http://127.0.0.1:8000/query" -G --data-urlencode "q=What is Kubernetes?"
```

```powershell
# Windows PowerShell
Invoke-WebRequest -Uri "http://127.0.0.1:8000/query?q=What%20is%20Kubernetes%3F" -Method POST
```

Example response:

```json
{
  "answer": "Kubernetes is a container platform used to manage containers at scale."
}
```

## Scripts

| Script | Purpose |
|--------|---------|
| `embed.py` | Legacy single-document embed script for `k8s.txt`. |
| `embed_docs.py` | Reads all `.txt` files from `docs/`, clears old embeddings, and rebuilds the ChromaDB store. |
| `semantic_test.py` | Runs semantic checks against the live API and validates expected keywords in responses. |

## API / Architecture

### `POST /query`

Queries the RAG system with a natural language question.

| Param | Type | Description |
|-------|------|-------------|
| `q` | `string` | The question to ask. |

Example request:

```bash
POST /query?q=What is Kubernetes?
```

Example response:

```json
{
  "answer": "Kubernetes is a container platform used to manage containers at scale."
}
```

### Request flow

1. The API queries ChromaDB for the most relevant document.
2. If `USE_MOCK_LLM=1`, the API returns the retrieved context directly.
3. Otherwise, the API sends the retrieved context and question to TinyLlama through Ollama.
4. The final answer is returned as JSON.

## Triggering CI and Catching Semantic Regressions

The CI workflow is designed to run when files that affect retrieval behavior change. That includes application code, embedding logic, and files inside the `docs/` directory.

A useful way to test the pipeline is to intentionally remove an expected keyword from a document, commit the change, and push it to GitHub. The workflow will rebuild embeddings, start the API in mock mode, run semantic tests, and fail if the retrieved context no longer contains the required concept.

Example:

```bash
git add docs/k8s.txt
git commit -m "Trigger CI by breaking Kubernetes document"
git push origin main
```

This is the key value of the project: retrieval regressions are caught automatically before they ship.

## Why Mock Mode Exists

LLM output is not deterministic. The same prompt can produce slightly different answers across runs, which makes automated testing unreliable.

Mock mode solves that problem by skipping generation and returning the retrieved context directly. That means CI tests validate retrieval quality only, without randomness from the model.

## Restructuring for Multiple Documents

The project starts with a single-file workflow and then moves to a more scalable multi-document structure.

### What changed

- Knowledge files were moved into a `docs/` directory
- A new `embed_docs.py` script was added to process all `.txt` files
- CI was updated to watch the entire `docs/` folder
- Semantic tests can now cover more than one document

### Why it matters

This structure scales much better than a single root-level file. New knowledge can be added by dropping a text file into `docs/`, rebuilding embeddings, and adding a matching semantic test.

### Example

```text
docs/
├── k8s.txt
└── nextwork.txt
```

Rebuild embeddings with:

```bash
python embed_docs.py
```

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

`db/` and `venv/` are generated locally and should stay out of version control.

## Deployment

If Ollama is running on another machine or service, point the API to it with `OLLAMA_HOST`.

```bash
OLLAMA_HOST=http://<ollama-host>:11434 uvicorn app:app
```

## Contributing

1. Fork the repository
2. Create a branch

```bash
git checkout -b feature/your-change
```

3. Make your changes
4. Update semantic tests when knowledge documents change
5. Open a pull request and let CI validate the change

Keep commits focused and descriptive.

## License

MIT