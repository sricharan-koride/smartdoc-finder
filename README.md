# 🚀 SmartDoc Finder

A self-hosted, privacy-first, Retrieval-Augmented Generation (RAG) engine designed to answer questions using your own documents. This project combines advanced search techniques with large language models to provide accurate, context-aware answers.

## ✅ Features

- ✅ **Hybrid Search**: Combines keyword (OpenSearch) and semantic (Qdrant) vector search for initial document retrieval.
- ✅ **Re-ranking**: Uses a sophisticated cross-encoder model to re-rank the initial results for maximum relevance.
- ✅ **Generative Answers**: Leverages a local LLM (via Ollama) to generate natural language answers based on the top-ranked documents.
- ✅ **Fully Containerized**: The entire stack is containerized with Docker Compose for easy setup and deployment.
- ✅ **Privacy First**: All data stays local — no external API calls for search or generation.

## 🏛️ Architecture Overview

The system follows a multi-stage RAG pipeline to ensure high-quality, factual answers:

### Indexing Pipeline

Documents are uploaded, and their text is extracted. The backend:
1. Saves the document to **PostgreSQL**
2. Indexes the text to **OpenSearch** for keyword search
3. Generates embeddings and stores vectors in **Qdrant** for semantic search

### Search & Generation Pipeline (Real-time)

- **Stage 1: Retrieval**  
  A user's query is sent to the backend, which performs a hybrid search (keyword + semantic) to retrieve candidate documents using RRF (Reciprocal Rank Fusion) scoring.

- **Stage 2: Re-ranking**  
  Candidates are sent to the reranker service, which uses a cross-encoder model (`ms-marco-MiniLM-L-6-v2`) to re-order and select the top documents.

- **Stage 3: Generation**  
  The query and top documents are sent to the generator service, which uses a local LLM (Ollama) to generate a final answer.

## 📂 Monorepo Structure (with Git Submodules)

This repository serves as the orchestration layer for the project, linking several independent services, each as a Git submodule.

| Service Folder              | Description                                               |
|----------------------------|-----------------------------------------------------------|
| `smartdoc-finder-backend`  | Spring Boot backend (APIs, OpenSearch, Qdrant, Orchestration) |
| `smartdoc-finder-frontend` | Next.js frontend (Web Interface)                          |
| `smartdoc-finder-embedding`| FastAPI service for generating embeddings                 |
| `smartdoc-finder-reranker` | FastAPI service for re-ranking search results             |
| `smartdoc-finder-generator`| FastAPI service that calls the LLM to generate answers    |
| `smartdoc-finder-ollama`   | Docker configuration for the self-hosted Ollama service   |

> 📝 **Note**: Each submodule points to its own Git repository. To clone the project correctly, you must include the submodules.

## 🐳 Getting Started

### Prerequisites

- Docker & Docker Compose  
- Git  
- An `.env` file in the project root (see `.env.example` for the required variables)

### 1. Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/sricharan-koride/smartdoc-finder.git
cd smartdoc-finder
```

### 2. Configure Your Environment

Create a `.env` file in the project root by copying the `.env.example` file.  
The most important variable to set is `USER_DATA_DIRECTORY`, which should point to the folder on your local machine containing the documents you want to index.

---

## 🚀 Quick Start (Recommended)

Use pre-built Docker images for the fastest startup (~30 seconds):

```bash
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up
```

---

## 🛠️ Development Mode

Build from source (for development or contributing):

```bash
docker compose up --build
```

This will build all service images locally, which takes longer on first run.

---

### Download AI Models (First-Time Setup)

The first time you run the application, the AI services need to download their models. This can take several minutes. You can monitor the progress by watching the logs.

- **Ollama (Generator)**: The `ollama` service will automatically start downloading the model specified in its `entrypoint.sh` script (e.g., `qwen2:0.5b`).
- **Embedding & Reranker**: These services will download their models (`all-mpnet-base-v2` and `ms-marco-MiniLM-L-6-v2`) when they first start up.

The application will be ready to use once all services are running and the models are downloaded.

---

## 🏗️ Tech Stack

| Component | Technology |
|-----------|------------|
| Backend | Spring Boot 3.4, Java 21 |
| Frontend | Next.js 15, React |
| Keyword Search | OpenSearch 2.11 |
| Vector Search | Qdrant |
| Embeddings | Sentence Transformers (all-mpnet-base-v2) |
| Reranker | Cross-Encoder (ms-marco-MiniLM-L-6-v2) |
| LLM | Ollama (qwen2:0.5b) |
| Database | PostgreSQL 16 |
| Containerization | Docker Compose |
