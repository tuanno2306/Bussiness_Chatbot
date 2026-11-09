---
name: rag-pipeline-expert
description: >-
  Expert guidelines and procedures for building enterprise-grade RAG (Retrieval-Augmented Generation) pipelines.
  Use when designing document ingestion, text chunking, embedding generation, hybrid search (dense + BM25 sparse),
  vector database indexing (Qdrant/pgvector), and re-ranking for Vietnamese and multilingual business knowledge bases.
---

# RAG Pipeline Expert Skill

This skill guides the design and implementation of high-accuracy, low-latency, hallucination-resistant RAG pipelines for enterprise business knowledge.

---

## 1. Document Ingestion & Chunking Strategy

### Supported Formats
* **Unstructured / Semi-structured:** PDF (policies, manuals), DOCX, TXT, Markdown.
* **Structured:** FAQ Sheets (CSV/Excel, JSON), Product catalogs.

### Chunking Rules
1. **Never chunk raw text purely by character count** without respecting semantic boundaries.
2. **Recursive / Markdown Chunking:**
   * Split on Markdown headers (`#`, `##`, `###`), followed by paragraph double newlines (`\n\n`), then sentences.
   * Target chunk size: **400 – 700 tokens** (approx. 1200 - 2200 characters in Vietnamese).
   * Chunk overlap: **10% – 15%** (approx. 50 - 80 tokens) to preserve contextual bridges between sections.
3. **Metadata Enrichment:**
   Attach rich metadata to every chunk:
   ```json
   {
     "doc_id": "chinh-sach-doi-tra-2026",
     "title": "Chính sách Đổi trả & Hoàn tiền",
     "section": "Điều kiện đổi trả sản phẩm lỗi",
     "category": "policy",
     "updated_at": "2026-03-01",
     "source_url": "https://doanhnghiep.vn/chinh-sach-doi-tra"
   }
   ```
4. **FAQ Ingestion:**
   * Ingest FAQ pairs as single atomic chunks: `Question: ... \n Answer: ...`.
   * Embed primarily the **Question** (or Question + Answer) so customer queries match the FAQ question semantics closely.

---

## 2. Embedding Models

| Scenario | Model Name | Dimensions | Why / Notes |
| :--- | :--- | :--- | :--- |
| **Multilingual & Vietnamese (Best Local/Open)** | `BAAI/bge-m3` | 1024 | Exceptional performance on Vietnamese semantic search, supports dense, sparse & multi-vector retrieval. |
| **Cloud API (OpenAI)** | `text-embedding-3-small` | 1536 (or reduced 512) | Low cost, low latency, very solid benchmark performance. |
| **Cloud API (Google)** | `text-embedding-004` | 768 | Native integration with Google ecosystem, fast. |

---

## 3. Hybrid Search (Dense Vector + BM25 Keyword)

Pure vector search struggles with:
* Exact product SKUs (e.g., `AO-SOMI-TRANG-SLIM-01`)
* Hotline phone numbers (e.g., `1900 6868`)
* Specialized industry jargon or acronyms

### Implementation Principle:
Always use **Reciprocal Rank Fusion (RRF)** or weighted score fusion:
$$\text{Score}(d) = \alpha \cdot \text{DenseScore}(d) + (1 - \alpha) \cdot \text{SparseScore}(d)$$
* Typical $\alpha$: `0.6` (dense) / `0.4` (sparse).
* Vector DB recommendation: **Qdrant** (built-in hybrid search with payload filtering) or **pgvector** with PostgreSQL full-text search (`tsvector`).

---

## 4. Cross-Encoder Re-Ranking

Before passing retrieved chunks to the LLM context, always apply a Re-ranker:
* **Recommended Re-ranker:** `BAAI/bge-reranker-v2-m3` or Cohere Rerank API.
* **Workflow:**
  1. Retrieve Top 15 - 20 candidate chunks via Hybrid Search.
  2. Pass `(query, chunk_content)` pairs into the Re-ranker.
  3. Keep only **Top 3 – 5 chunks** with score $> \text{threshold}$ (e.g., score $> 0.35$).
  4. If top score is below threshold, flag as "Low Confidence" $\rightarrow$ trigger fallback response.

---

## 5. Python Implementation Template (Qdrant + FastEmbed)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
from fastembed import TextEmbedding

# 1. Initialize
client = QdrantClient(host="localhost", port=6333)
embedder = TextEmbedding(model_name="BAAI/bge-m3")

COLLECTION_NAME = "enterprise_knowledge"

def index_chunks(chunks: list[dict]):
    texts = [c["content"] for c in chunks]
    embeddings = list(embedder.embed(texts))
    
    points = [
        PointStruct(
            id=idx,
            vector=embedding.tolist(),
            payload=chunk
        )
        for idx, (chunk, embedding) in enumerate(zip(chunks, embeddings))
    ]
    client.upsert(collection_name=COLLECTION_NAME, points=points)

def search_knowledge(query: str, top_k: int = 5):
    query_vector = list(embedder.embed([query]))[0].tolist()
    results = client.search(
        collection_name=COLLECTION_NAME,
        query_vector=query_vector,
        limit=top_k
    )
    return [hit.payload for hit in results]
```
