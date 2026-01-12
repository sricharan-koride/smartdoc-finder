# Migration Plan: Lucene/FAISS → OpenSearch/Qdrant

## Overview

Migrate from manual Lucene + FAISS implementation to professional, self-hosted OpenSearch + Qdrant services.

| Current | New |
|---------|-----|
| Apache Lucene (embedded Java) | OpenSearch (containerized) |
| FAISS (Python service) | Qdrant (containerized) |

> **Legacy Branch**: `lucene-faiss-legacy` preserves the current implementation

---

## Phase 1: Infrastructure Changes

### 1.1 Add OpenSearch & Qdrant to Docker Compose

**File**: `docker-compose.yaml`

Add these services:

```yaml
# --- OpenSearch (Keyword Search) ---
opensearch:
  image: opensearchproject/opensearch:2.11.0
  environment:
    - discovery.type=single-node
    - plugins.security.disabled=true
    - OPENSEARCH_INITIAL_ADMIN_PASSWORD=admin
  ports:
    - "9200:9200"
  volumes:
    - opensearch_data:/usr/share/opensearch/data
  healthcheck:
    test: ["CMD-SHELL", "curl -s http://localhost:9200 | grep -q 'cluster_name'"]
    interval: 10s
    timeout: 5s
    retries: 10

# --- Qdrant (Vector Search) ---
qdrant:
  image: qdrant/qdrant:latest
  ports:
    - "6333:6333"
    - "6334:6334"
  volumes:
    - qdrant_data:/qdrant/storage
```

Add volumes:
```yaml
volumes:
  opensearch_data:
  qdrant_data:
```

### 1.2 Remove Obsolete Services

**Remove from docker-compose.yaml**:
- `embedding-worker` (replaced by direct Qdrant indexing)

**Keep**:
- `embedding-api` (still needed for generating embeddings)

---

## Phase 2: Backend Refactoring (Java/Spring Boot)

### 2.1 Add Dependencies

**File**: `smartdoc-finder-backend/pom.xml`

Add OpenSearch client:
```xml
<dependency>
    <groupId>org.opensearch.client</groupId>
    <artifactId>opensearch-java</artifactId>
    <version>2.8.0</version>
</dependency>
```

Add Qdrant client:
```xml
<dependency>
    <groupId>io.qdrant</groupId>
    <artifactId>client</artifactId>
    <version>1.7.0</version>
</dependency>
```

---

### 2.2 Create Search Provider Interface

**File**: [NEW] `SearchProvider.java`

Create an abstraction layer:
```java
public interface KeywordSearchProvider {
    void indexDocument(Long id, String filename, String content);
    List<SearchResult> search(String query, int maxResults);
}

public interface VectorSearchProvider {
    void indexVector(Long docId, float[] vector);
    List<VectorSearchHit> search(float[] queryVector, int maxResults);
}
```

---

### 2.3 Refactor LuceneService → OpenSearchService

**File**: `LuceneService.java` → **Rename to** `OpenSearchService.java`

**Current methods to refactor**:

| Method | Current (Lucene) | New (OpenSearch) |
|--------|------------------|------------------|
| `init()` | Creates IndexWriter | Creates OpenSearch client |
| `indexDocument()` | Writes to Lucene index | HTTP PUT to OpenSearch |
| `search()` | Lucene Query + TopDocs | OpenSearch Query DSL |
| `buildLuceneQuery()` | BooleanQuery builder | OpenSearch match query JSON |
| `createSnippet()` | Lucene Highlighter | OpenSearch highlight API |
| `cleanup()` | Closes IndexWriter | Closes client |

**Key changes**:
- Replace `Directory`, `IndexWriter`, `IndexSearcher` with OpenSearch `OpenSearchClient`
- Replace Lucene `Query` with OpenSearch query DSL
- Replace `TopDocs` iteration with OpenSearch `SearchResponse` parsing

---

### 2.4 Refactor SemanticSearchService → QdrantService

**File**: `SemanticSearchService.java` → **Rename to** `QdrantService.java`

**Current**: Calls embedding API's `/semantic-search` endpoint (which uses FAISS internally)

**New**: 
- Call embedding API's `/embed` to get vector
- Call Qdrant HTTP API directly for vector search

**Changes**:
- Add Qdrant REST client
- Replace `/semantic-search` call with:
  1. `/embed` to get vector
  2. Direct Qdrant `/collections/{name}/points/search` call

---

### 2.5 Update LuceneIndexListener → OpenSearchIndexListener

**File**: `LuceneIndexListener.java`

**Current**: Listens to RabbitMQ and indexes to Lucene
**New**: Index to OpenSearch instead

---

### 2.6 Update EmbeddingMessageListener → QdrantIndexListener  

**File**: `EmbeddingMessageListener.java`

**Current**: Sends messages to RabbitMQ for FAISS worker
**New**: 
- Get embedding from embedding API
- Index directly to Qdrant

---

### 2.7 Remove/Update Config Files

**Remove**: `LuceneConfig.java` (no longer needed)

**Add**: 
- `OpenSearchConfig.java` (client configuration)
- `QdrantConfig.java` (client configuration)

---

## Phase 3: Embedding Service Refactoring (Python)

### 3.1 Simplify main.py

**Remove**:
- All FAISS code (`faiss.read_index`, `index.search`, etc.)
- `/semantic-search` endpoint (no longer needed)

**Keep**:
- `/multi-embed` endpoint (still needed for generating embeddings)

### 3.2 Delete Files

**Delete**:
- `build_faiss_index.py` (replaced by Qdrant)
- `index_corpus.py` (if exists)

---

## Phase 4: Environment Configuration

### 4.1 Add Environment Variables

**File**: `.env`

```env
# OpenSearch
OPENSEARCH_URL=http://opensearch:9200

# Qdrant
QDRANT_URL=http://qdrant:6333
QDRANT_COLLECTION=documents
```

### 4.2 Update Backend Service

**File**: `docker-compose.yaml` - backend service

Add environment variables:
```yaml
backend:
  environment:
    OPENSEARCH_URL: "http://opensearch:9200"
    QDRANT_URL: "http://qdrant:6333"
  depends_on:
    opensearch:
      condition: service_healthy
    qdrant:
      condition: service_started
```

---

## Phase 5: Data Migration

### 5.1 Create OpenSearch Index

Run once after OpenSearch starts:

```bash
curl -X PUT "http://localhost:9200/documents" -H "Content-Type: application/json" -d '{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "filename": { "type": "text", "analyzer": "standard" },
      "content": { "type": "text", "analyzer": "standard" }
    }
  }
}'
```

### 5.2 Create Qdrant Collection

```bash
curl -X PUT "http://localhost:6333/collections/documents" -H "Content-Type: application/json" -d '{
  "vectors": {
    "size": 768,
    "distance": "Cosine"
  }
}'
```

---

## Files Summary

### Files to MODIFY

| File | Changes |
|------|---------|
| `docker-compose.yaml` | Add OpenSearch, Qdrant; remove embedding-worker |
| `pom.xml` | Add OpenSearch + Qdrant client dependencies |
| `LuceneService.java` | Rewrite as `OpenSearchService.java` |
| `SemanticSearchService.java` | Rewrite as `QdrantService.java` |
| `LuceneIndexListener.java` | Update for OpenSearch |
| `EmbeddingMessageListener.java` | Update for Qdrant |
| `embedding/main.py` | Remove FAISS, keep only embedding |

### Files to DELETE

| File | Reason |
|------|--------|
| `LuceneConfig.java` | No longer needed |
| `build_faiss_index.py` | Replaced by Qdrant |

### Files to CREATE

| File | Purpose |
|------|---------|
| `OpenSearchConfig.java` | OpenSearch client bean |
| `QdrantConfig.java` | Qdrant client bean |
| `KeywordSearchProvider.java` | Abstraction interface |
| `VectorSearchProvider.java` | Abstraction interface |

---

## Verification Plan

### After Phase 1 (Infrastructure)
- [ ] `curl http://localhost:9200` returns OpenSearch cluster info
- [ ] `curl http://localhost:6333/collections` returns empty list

### After Phase 2-3 (Code Refactoring)
- [ ] Backend starts without errors
- [ ] Documents index to both OpenSearch and Qdrant
- [ ] Search returns results from both sources

### After Phase 5 (Data Migration)
- [ ] All existing documents re-indexed
- [ ] Search quality comparable to previous implementation

---

## Estimated Effort

| Phase | Effort |
|-------|--------|
| Phase 1: Infrastructure | 1-2 hours |
| Phase 2: Backend refactoring | 2-3 days |
| Phase 3: Embedding service | 1-2 hours |
| Phase 4: Configuration | 1 hour |
| Phase 5: Data migration | 2-4 hours |
| **Total** | **3-4 days** |
