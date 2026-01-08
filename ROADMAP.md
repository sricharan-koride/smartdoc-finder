# SmartDoc Finder: Engineering Roadmap

## 📊 Current Architecture

| Service | Technology | Purpose |
|---------|-----------|---------|
| Backend | Spring Boot + Lucene | Keyword search, orchestration |
| Embedding | FastAPI + FAISS | Semantic vector search |
| Generator | FastAPI + Ollama (Qwen2) | RAG answer generation |
| Reranker | FastAPI + Cross-Encoder | Result re-ranking |
| Infrastructure | PostgreSQL, RabbitMQ, Docker Compose | Data & messaging |

---

## 🏗️ Roadmap Philosophy

> **Architectural changes FIRST, features SECOND.**

Features are categorized as:
- **🔄 Architecture-Agnostic** - Work on any backend, do anytime
- **🏛️ Foundation** - Must be decided/built before dependent features
- **🏗️ Build Phase** - Features that depend on the new architecture

---

# Phase 1: Architecture-Agnostic Improvements
*These work regardless of backend choice. Start here while deciding on architecture.*

### 1. Streaming RAG Responses ⭐⭐
**Time**: 1-2 days | **Impact**: 🔥🔥🔥

Add SSE streaming from Ollama → Backend → Frontend for token-by-token responses.

### 2. Observability Stack ⭐⭐⭐
**Time**: 2-3 days | **Impact**: 🔥🔥🔥🔥

Add OpenTelemetry, Prometheus, Grafana, Jaeger. Works with any search backend.

### 3. CI/CD Pipeline ⭐⭐⭐
**Time**: 3-5 days | **Impact**: 🔥🔥🔥

GitHub Actions, integration tests with Testcontainers. Backend-agnostic.

### 4. Authentication & Multi-Tenancy ⭐⭐⭐⭐
**Time**: 1-2 weeks | **Impact**: 🔥🔥🔥

JWT auth, RBAC, tenant isolation. Lives in backend/frontend, not search layer.

### 5. Document Processing Pipeline ⭐⭐⭐
**Time**: 1 week | **Impact**: 🔥🔥🔥

OCR, table extraction, semantic chunking. Preprocessing is backend-agnostic.

---

# Phase 2: Architectural Foundation
*Make these decisions BEFORE building search-dependent features.*

### 🚨 Decision Point: Keep Lucene/FAISS or Migrate?

| Keep Current | Migrate to Professional Tooling |
|--------------|--------------------------------|
| Simpler, educational value | Production-grade features |
| Full control over internals | Better query DSL, aggregations |
| Good for learning | Easier scaling, managed indices |

**If migrating**, do this FIRST:

### 6. Migrate to OpenSearch + Qdrant ⭐⭐⭐⭐
**Time**: 2-3 weeks | **Impact**: 🔥🔥🔥🔥🔥

```yaml
# Replace Lucene with OpenSearch
opensearch:
  image: opensearchproject/opensearch:latest
  ports:
    - "9200:9200"

# Replace FAISS with Qdrant  
qdrant:
  image: qdrant/qdrant:latest
  ports:
    - "6333:6333"
```

**Migration Steps**:
1. Add new services to docker-compose
2. Create abstraction layer in backend (SearchProvider interface)
3. Implement OpenSearch + Qdrant adapters
4. Migrate indexing pipeline
5. Update search queries
6. Remove old Lucene/FAISS code

> ⚠️ **Once you decide to migrate, skip any Lucene/FAISS-specific optimizations.**

---

# Phase 3: Features on New Architecture
*Build these AFTER architectural foundation is complete.*

### 7. Hybrid Search with Reciprocal Rank Fusion ⭐⭐⭐
**Time**: 3-5 days | **Impact**: 🔥🔥🔥🔥

Combine keyword + vector scores. Both OpenSearch and Qdrant support this natively.

### 8. Learned Ranking Model ⭐⭐⭐⭐⭐
**Time**: 2 weeks | **Impact**: 🔥🔥🔥🔥🔥

Train XGBoost/LightGBM on features from the new search stack. OpenSearch has built-in LTR plugin support.

### 9. Query Understanding Pipeline ⭐⭐⭐⭐
**Time**: 1 week | **Impact**: 🔥🔥🔥🔥🔥

Spell check, intent classification, query expansion. Build on top of final search layer.

---

# Phase 4: Advanced AI Features
*These build on top of all previous phases.*

### 10. Interactive Chat Interface ⭐⭐⭐
**Time**: 1 week | **Impact**: 🔥🔥🔥🔥

Conversational search with memory. Independent of search backend.

### 11. Source Citations & Highlighting ⭐⭐⭐
**Time**: 3-5 days | **Impact**: 🔥🔥🔥🔥

Clickable citations linking to source passages.

### 12. Fact-Checking with NLI ⭐⭐⭐⭐⭐
**Time**: 2 weeks | **Impact**: 🔥🔥🔥🔥🔥

Deploy NLI model to verify LLM claims against sources.

---

# Phase 5: Production Deployment
*Final hardening for production.*

### 13. Kubernetes + Helm Charts ⭐⭐⭐⭐
**Time**: 1-2 weeks | **Impact**: 🔥🔥🔥🔥

Helm charts, HPA, PVCs, health probes.

---

## 📋 Summary: Correct Order

```
Week 1-2:   Streaming RAG + Observability (architecture-agnostic)
Week 3-4:   CI/CD + Document Processing (architecture-agnostic)
Week 5-7:   🏛️ OpenSearch/Qdrant Migration (FOUNDATION)
Week 8-9:   Hybrid Search + Query Understanding (on new arch)
Week 10-11: Chat Interface + Citations (on new arch)
Week 12+:   Learned Ranking + Fact-Checking + Kubernetes
```

---

## ❌ Features to SKIP if Migrating

If you decide to migrate to OpenSearch/Qdrant, **do NOT work on**:
- ~~FAISS IVF-PQ/HNSW optimization~~
- ~~Lucene analyzer tuning~~
- ~~Custom FAISS sharding~~

These become obsolete once you adopt professional tooling.
