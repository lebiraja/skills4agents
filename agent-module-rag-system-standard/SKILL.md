---
name: agent-module-rag-system-standard
description: "Module: Production-Grade RAG System Architecture and Implementation Standard"
---
# Module: Production-Grade RAG System Architecture and Implementation Standard

## 1. Module Metadata
- Module ID: `agent.module.rag-system-standard`
- Version: `1.0.0`
- Maturity: `production`
- Scope: End-to-end RAG (Retrieval-Augmented Generation) architecture, embedding model selection, vector database deployment, pipeline orchestration, and production readiness for knowledge-grounded LLM applications.
- Primary outcomes:
  - Deterministic embedding and retrieval pipeline with measurable quality metrics.
  - Production-grade vector database deployment (local and cloud).
  - Multi-tier embedding model selection strategy (cost vs. quality).
  - Full RAG pipeline with chunking, indexing, retrieval, and re-ranking.
  - Operational readiness with monitoring, caching, and failover.

## 2. Mission and Applicability
Use this module to architect and deploy RAG systems that augment LLM responses with domain knowledge, documents, or proprietary data.

Apply when:
- LLM needs grounding in external knowledge (documents, code, data).
- System requires up-to-date information retrieval (not in training data).
- Domain-specific accuracy and citation are critical.
- Multi-tenant or compliance-heavy retrieval is required.

Do not apply directly when:
- Knowledge fits in LLM context window (< 16K tokens).
- System is pure generation without retrieval (no external data).
- Real-time retrieval latency is incompatible with SLA (< 50ms).

## 3. RAG Architecture Pattern
- Pattern: `Chunked retrieval + semantic ranking + LLM augmentation`
- Core design rules:
  - Separate indexing pipeline (offline) from retrieval pipeline (online).
  - Embedding model quality drives retrieval quality; never compromise for cost alone.
  - Vector database must support metadata filtering, exact match fallback, and HNSW indexing.
  - Retrieval results ranked by semantic similarity + business rules (recency, authority, domain).
  - LLM prompt includes retrieved context + confidence scores + source attribution.
  - Monitor embedding drift, retrieval latency, and hallucination rates independently.

**Core RAG Flow:**
```
User Query
    ↓
[Embedding Model] (embed query)
    ↓
[Vector DB] (semantic search)
    ↓
[Re-ranker] (rank & filter results)
    ↓
[Context Assembly] (format retrieved docs + metadata)
    ↓
[LLM Prompt] (augment with context)
    ↓
[LLM Output] (generate response with attribution)
```

---

## 4. Implementation Workflow

### Phase A: Data Preparation and Chunking Strategy
1. **Document Ingestion**: Collect and normalize source documents (PDFs, web, databases, code).
2. **Chunking Strategy**:
   - Chunk size: 256–512 tokens (semantic unit, not overlap).
   - Overlap: 50–100 tokens (preserve context boundaries).
   - Metadata preservation: source file, chunk index, creation date, authority score.
3. **Cleaning**: Remove boilerplate (headers, footers), normalize whitespace, detect language.
4. **Versioning**: Track document versions; expire stale chunks.

Exit criteria:
- All documents chunked with consistent metadata.
- Chunk size tuned to embedding model's optimal range.
- Version tracking in place.

### Phase B: Embedding Model Selection and Deployment

**Embedding Model Decision Matrix:**

| Criteria | Local (Ollama) | Cloud (AWS/OpenAI/Cohere) | Enterprise (Proprietary) |
|----------|---|---|---|
| **Latency (single embed)** | 50–200ms | 100–500ms | <100ms |
| **Throughput (batch)** | 1K/sec | 10K–100K/sec | 100K+/sec |
| **Cost** | ~$0 (hardware) | $0.02–1.00 per 1K embeds | Custom |
| **Privacy** | ✅ Full | ❌ Cloud-stored | ✅ Full |
| **Dimensionality** | 768–4096 | 1536–3072 | 1024–10000 |
| **Quality (MTEB)** | 65–75 | 75–85 | 80–95 |

#### **Tier 1: Open-Source Local Models (Ollama)**

**Best for:** Privacy-first, cost-sensitive, offline capability.

1. **BAAI/bge-small-en-v1.5** (384 dims)
   - MTEB Score: 63.18
   - Size: 125M parameters
   - Latency: 30–50ms (CPU), 10–15ms (GPU)
   - Use when: High-volume indexing, limited budget, CPU-only infrastructure
   - Installation: `ollama pull bge-small-en`
   - Cost: $0 (your compute)

2. **BAAI/bge-base-en-v1.5** (768 dims)
   - MTEB Score: 63.55
   - Size: 209M parameters
   - Latency: 50–100ms (CPU), 20–30ms (GPU)
   - Use when: Balanced quality and speed, standard RAG workloads
   - Installation: `ollama pull bge-base-en`
   - Cost: $0 (your compute)

3. **BAAI/bge-large-en-v1.5** (1024 dims)
   - MTEB Score: 64.23
   - Size: 335M parameters
   - Latency: 100–200ms (CPU), 40–60ms (GPU)
   - Use when: Premium quality required, GPU available
   - Installation: `ollama pull bge-large-en`
   - Cost: $0 (your compute)

4. **Sentence-Transformers/all-mpnet-base-v2** (768 dims)
   - MTEB Score: 60.89
   - Size: 109M parameters
   - Use when: Legacy systems, fine-tuning capability needed
   - Cost: $0 (your compute)

#### **Tier 2: Cloud-Hosted Models (Cost-Quality Balance)**

**Best for:** Scalability, managed infrastructure, multi-tenant systems.

1. **OpenAI text-embedding-3-small** (1536 dims)
   - MTEB Score: ~75
   - Cost: $0.02 per 1M tokens (~$0.02 per 1K embeds)
   - Latency: 100–300ms
   - Batch support: Yes (10K+ embeds/sec)
   - Use when: Need high quality + OpenAI ecosystem
   - Provider: OpenAI API
   - Max tokens: 8191

2. **OpenAI text-embedding-3-large** (3072 dims)
   - MTEB Score: ~80
   - Cost: $0.13 per 1M tokens (~$0.13 per 1K embeds)
   - Latency: 100–400ms
   - Use when: Premium quality, budget allows
   - Provider: OpenAI API

3. **AWS Bedrock Amazon Titan Embeddings** (1536 dims)
   - MTEB Score: ~70
   - Cost: $0.00013 per 1K embeds (+ Bedrock service fee)
   - Latency: 200–500ms
   - Regional: Any AWS region
   - Use when: AWS-native, cost optimization
   - Provider: AWS Bedrock
   - IAM: bedrock:InvokeModel

4. **Cohere Embed API** (1024 dims)
   - MTEB Score: ~72
   - Cost: $0.10 per 1M tokens (~$0.10 per 1K embeds)
   - Latency: 100–250ms
   - Input limit: 512 tokens per doc
   - Use when: Multi-language support needed
   - Provider: Cohere API

#### **Tier 3: Enterprise Models (Maximum Quality)**

1. **Anthropic Claude Embeddings** (TBD dims)
   - Use when: Full Claude ecosystem integration
   - Provider: Claude API (via bedrock or direct)
   - MTEB: Estimated 80+

2. **Proprietary Fine-Tuned Models**
   - Use when: Domain-specific vocabulary (medical, legal, finance)
   - Training: Fine-tune on domain corpus (1000+ examples recommended)
   - Cost: Development effort + inference cost

**Decision Rule:**
```
IF latency < 50ms AND offline required:
  → BAAI bge-large-en-v1.5 (local GPU)
ELIF latency < 100ms AND cost < $0.05:
  → BAAI bge-base-en-v1.5 (local CPU)
ELIF quality > cost (MTEB > 75):
  → OpenAI text-embedding-3-small
ELIF multi-cloud required:
  → AWS Bedrock Titan
ELIF domain-specific (medical/legal):
  → Fine-tuned proprietary model
ELSE:
  → BAAI bge-base-en-v1.5 (default)
```

Exit criteria:
- Embedding model selected with documented trade-offs.
- Local or cloud deployment tested end-to-end.
- Latency and cost measured.

### Phase C: Vector Database Selection and Deployment

**Vector DB Decision Matrix:**

| Database | Deployment | Latency (search) | Scalability | Cost | Best For |
|----------|---|---|---|---|---|
| **Pinecone** | Cloud | 50–100ms | Auto-scale (100M+) | $0.25/unit | Managed, multi-tenant |
| **Weaviate** | Self-hosted | 10–50ms | Up to 1B vectors | Self-hosted | Privacy, control |
| **Milvus** | Self-hosted | 10–50ms | Distributed (100M+) | Self-hosted | High-volume, on-prem |
| **Qdrant** | Self-hosted | 10–50ms | Up to 1B vectors | Self-hosted | Fast, minimal deps |
| **Chroma** | Local/Docker | 20–100ms | Up to 10M | Open-source | Development, small RAG |
| **FAISS** | Local library | 5–20ms | Batch-only (no live update) | Open-source | Offline, batch retrieval |
| **Elasticsearch** | Self-hosted | 50–200ms | 1B+ (with tuning) | Self-hosted | Full-text + vector hybrid |

#### **Tier 1: Self-Hosted (Full Control)**

1. **Qdrant** (Recommended for most RAG systems)
   - **Deployment**: Docker container or binary
   - **Latency**: 10–30ms (search 10M vectors)
   - **Storage**: Persistent disk, snapshots
   - **Install (Docker)**:
     ```bash
     docker run -p 6333:6333 qdrant/qdrant:latest
     ```
   - **Install (Local)**:
     ```bash
     curl -L https://github.com/qdrant/qdrant/releases/download/v1.x.x/qdrant-x86_64-unknown-linux-gnu \
       -o qdrant && chmod +x qdrant && ./qdrant
     ```
   - **Python Client**:
     ```python
     from qdrant_client import QdrantClient
     client = QdrantClient("http://localhost:6333")
     ```
   - **Pros**: Fast, minimal dependencies, HNSW indexing, snapshot/restore
   - **Cons**: Self-managed scaling
   - **Use when**: < 100M vectors, full control needed, privacy critical

2. **Milvus** (Enterprise-scale self-hosted)
   - **Deployment**: Kubernetes or Docker Compose
   - **Latency**: 10–50ms
   - **Scalability**: Distributed (sharding across nodes)
   - **Storage**: Persistent storage (MinIO, S3)
   - **Install (Kubernetes)**:
     ```bash
     helm repo add milvus https://milvus-io.github.io/milvus-helm/
     helm install milvus milvus/milvus
     ```
   - **Python Client**:
     ```python
     from pymilvus import Collection, connections
     connections.connect("default", host="localhost", port=19530)
     ```
   - **Pros**: Highly scalable, HNSW + IVF indexing, cloud-native
   - **Cons**: Complex deployment, resource-heavy
   - **Use when**: > 100M vectors, horizontal scaling needed

3. **Weaviate** (GraphQL + Vector hybrid)
   - **Deployment**: Docker or Kubernetes
   - **Latency**: 20–50ms
   - **GraphQL API**: Native support
   - **Install (Docker)**:
     ```bash
     docker run -p 8080:8080 semitechnologies/weaviate:latest
     ```
   - **Python Client**:
     ```python
     import weaviate
     client = weaviate.Client("http://localhost:8080")
     ```
   - **Pros**: GraphQL queries, metadata filtering, hybrid search
   - **Cons**: Learning curve, slightly slower than Qdrant
   - **Use when**: Complex queries, graph relationships needed

#### **Tier 2: Cloud-Managed (Minimal Ops)**

1. **Pinecone** (Zero-ops vector DB)
   - **Setup**: API key, no infrastructure
   - **Latency**: 50–100ms (search)
   - **Scaling**: Automatic (pay per "pod" capacity)
   - **Cost**: $0.25–1.00 per pod/day (~$75–300/month for standard)
   - **Install**:
     ```bash
     pip install pinecone-client
     ```
   - **Python**:
     ```python
     import pinecone
     pinecone.init(api_key="YOUR_KEY", environment="us-west1-gcp")
     index = pinecone.Index("my-index")
     index.upsert(vectors=[...])
     ```
   - **Pros**: Fully managed, auto-scaling, live updates
   - **Cons**: Vendor lock-in, higher latency, data residency concerns
   - **Use when**: Managed service preferred, multi-tenant SaaS

2. **AWS OpenSearch with Vector Support**
   - **Setup**: AWS console or Infrastructure as Code
   - **Latency**: 50–200ms
   - **Cost**: $0.25–1.00/hour for instances (~$180–750/month)
   - **Pros**: Integrated with AWS ecosystem, hybrid full-text + vector
   - **Cons**: Complex tuning, higher cost for small workloads
   - **Use when**: AWS-native, existing OpenSearch deployment

3. **Azure Cognitive Search + Vector Search**
   - **Setup**: Azure portal
   - **Latency**: 50–200ms
   - **Cost**: Starts at ~$200/month
   - **Pros**: Azure integration, RBAC, compliance
   - **Use when**: Azure-first organization

#### **Tier 3: Lightweight Local (Development)**

1. **Chroma** (Development-focused)
   - **Install**: `pip install chromadb`
   - **Latency**: 20–100ms
   - **In-memory or persistent**:
     ```python
     import chromadb
     chroma_client = chromadb.Client()
     collection = chroma_client.create_collection(name="my_collection")
     collection.add(ids=[...], embeddings=[...], documents=[...])
     ```
   - **Use when**: Prototyping, < 1M vectors, single-user

2. **FAISS** (Facebook AI Similarity Search - batch only)
   - **Install**: `pip install faiss-cpu`
   - **Latency**: 5–20ms
   - **Limitation**: No live updates, offline indexing only
   - **Use when**: Batch indexing, no live queries

**Decision Rule:**
```
IF self-hosted AND < 100M vectors:
  → Qdrant (simplest, fastest)
ELIF self-hosted AND > 100M vectors:
  → Milvus (distributed)
ELIF managed service AND cost not critical:
  → Pinecone (zero-ops)
ELIF AWS ecosystem:
  → OpenSearch Vector
ELIF development/prototyping:
  → Chroma (local)
ELIF batch-only use case:
  → FAISS (fastest)
ELSE:
  → Qdrant (default)
```

Exit criteria:
- Vector DB selected and deployed.
- Latency and throughput benchmarked.
- Backup/restore tested.

### Phase D: Complete RAG Indexing Pipeline

1. **Document Ingestion**:
   - Extract text from PDF/web/database
   - Normalize encoding (UTF-8)
   - Clean whitespace and boilerplate
   - Parse metadata (title, author, date, URL)

2. **Chunking**:
   - Split by semantic boundaries (paragraphs, sentences)
   - 256–512 tokens per chunk
   - 50–100 token overlap
   - Preserve chunk ordering

3. **Embedding**:
   - Batch embedding (1000+ at once)
   - Monitor for failed embeds
   - Cache embeddings to disk
   - Track embedding version (model + date)

4. **Indexing**:
   - Upsert to vector DB with metadata
   - Create composite index (semantic + exact match)
   - Verify all chunks indexed
   - Monitor index size and latency

5. **Metadata Storage**:
   - Store in relational DB alongside vector DB
   - Source document, chunk ID, offset, date
   - Authority score, access control
   - Version and expiration

**Example Indexing Pipeline (Python)**:
```python
from sentence_transformers import SentenceTransformer
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct
import hashlib
import time

# Load embedding model
model = SentenceTransformer("BAAI/bge-base-en-v1.5")

# Connect to vector DB
client = QdrantClient("http://localhost:6333")

# Create collection
client.recreate_collection(
    collection_name="documents",
    vectors_config={"size": 768, "distance": "Cosine"},
)

# Index documents
def index_documents(documents: list[dict]):
    """
    Args:
        documents: [{"id": "doc1", "content": "...", "source": "..."}, ...]
    """
    for doc_id, doc in enumerate(documents):
        # Chunk document
        chunks = chunk_text(doc["content"], chunk_size=512, overlap=100)
        
        # Embed chunks
        embeddings = model.encode(chunks, batch_size=32)
        
        # Prepare points for vector DB
        points = []
        for chunk_idx, (chunk_text, embedding) in enumerate(zip(chunks, embeddings)):
            point_id = int(hashlib.md5(f"{doc['source']}_{chunk_idx}".encode()).hexdigest(), 16) % (10**8)
            points.append(PointStruct(
                id=point_id,
                vector=embedding.tolist(),
                payload={
                    "source": doc["source"],
                    "chunk_index": chunk_idx,
                    "chunk_text": chunk_text,
                    "timestamp": int(time.time()),
                }
            ))
        
        # Upsert to vector DB
        client.upsert(collection_name="documents", points=points)
        print(f"Indexed {len(points)} chunks from {doc['source']}")

def chunk_text(text: str, chunk_size: int = 512, overlap: int = 100) -> list[str]:
    """Simple chunking by tokens (approximate)."""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        if chunk.strip():
            chunks.append(chunk)
    return chunks
```

Exit criteria:
- All documents indexed with embeddings
- Metadata stored and queryable
- Index latency < 1s per document
- Integrity checks pass

### Phase E: Retrieval Pipeline with Re-ranking

1. **Query Embedding**:
   - Embed user query with same model as documents
   - Cache query embeddings (by hash)
   - Monitor embedding latency

2. **Vector Search**:
   - Top-k retrieval from vector DB (k=20–50)
   - Apply metadata filters (date, source, authority)
   - Track search latency

3. **Re-ranking**:
   - Option A: Semantic re-ranker (cross-encoder, slower but more accurate)
   - Option B: BM25 hybrid (full-text re-rank)
   - Option C: Business rules (recency, source authority)
   - Return top-3 to top-5 results

4. **Context Assembly**:
   - Format retrieved chunks with metadata
   - Add source attribution + confidence scores
   - Enforce context window limits (max 4K tokens)

**Example Retrieval (Python)**:
```python
from sentence_transformers import CrossEncoder

# Load re-ranker
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")

def retrieve_context(query: str, top_k: int = 20, rerank_k: int = 3):
    """Retrieve and re-rank documents."""
    
    # Embed query
    query_embedding = model.encode(query)
    
    # Vector search
    results = client.search(
        collection_name="documents",
        query_vector=query_embedding.tolist(),
        limit=top_k,
    )
    
    # Extract documents and scores
    documents = []
    scores = []
    for hit in results:
        documents.append(hit.payload["chunk_text"])
        scores.append(hit.score)
    
    # Re-rank using cross-encoder
    pairs = [[query, doc] for doc in documents]
    rerank_scores = reranker.predict(pairs)
    
    # Sort by re-rank scores
    ranked = sorted(zip(documents, rerank_scores, scores), key=lambda x: x[1], reverse=True)
    
    # Return top reranked results
    return [
        {
            "content": doc,
            "semantic_score": semantic_score,
            "rerank_score": rerank_score,
        }
        for doc, rerank_score, semantic_score in ranked[:rerank_k]
    ]

def assemble_context(retrieved: list[dict], max_tokens: int = 4000) -> str:
    """Format retrieved context for LLM."""
    context_parts = []
    token_count = 0
    
    for i, item in enumerate(retrieved):
        # Estimate tokens
        tokens = len(item["content"].split()) * 1.3
        if token_count + tokens > max_tokens:
            break
        
        context_parts.append(
            f"[Source {i+1}] {item['content']}\n"
            f"(Confidence: {item['rerank_score']:.2f})\n"
        )
        token_count += tokens
    
    return "".join(context_parts)
```

Exit criteria:
- Retrieval latency < 500ms
- Re-ranking working and improving quality
- Context assembly deterministic

### Phase F: LLM Augmentation and Response Generation

1. **Prompt Engineering**:
   - Include retrieved context + citations
   - Define confidence thresholds (accept/reject retrieved docs)
   - Instruct LLM to decline when context insufficient
   - Require inline citations

2. **Generation**:
   - Stream response to client
   - Track token usage and cost
   - Monitor hallucination rates

3. **Attribution**:
   - Append source citations at end
   - Confidence scores
   - Links to original documents

**Example RAG Prompt**:
```python
def build_rag_prompt(query: str, context: str) -> str:
    return f"""You are a helpful assistant answering questions based on provided documents.

Retrieved Context:
{context}

Instructions:
1. Answer the user's question using ONLY the provided context.
2. If the context doesn't contain enough information to answer, say "I don't have enough information."
3. Always cite your sources inline: [Source 1], [Source 2], etc.
4. Be accurate and concise.

User Question: {query}

Answer:"""

def generate_response(query: str, llm_client):
    """Generate RAG response."""
    # Retrieve context
    retrieved = retrieve_context(query)
    context = assemble_context(retrieved)
    
    # Build prompt
    prompt = build_rag_prompt(query, context)
    
    # Generate response
    response = llm_client.generate(prompt, stream=True)
    
    # Append sources
    sources = [f"- {r['content'][:100]}..." for r in retrieved]
    full_response = f"{response}\n\nSources:\n" + "\n".join(sources)
    
    return full_response
```

Exit criteria:
- End-to-end RAG pipeline working
- Responses include citations
- Latency < 2 seconds (retrieval + LLM)

### Phase G: Production Monitoring and Optimization

1. **Metrics to Track**:
   - Retrieval latency (p50, p95)
   - Embedding model performance (cache hit rate)
   - Re-ranking quality (NDCG)
   - Hallucination rate (manual review, ~100 samples)
   - User feedback (thumbs up/down)

2. **Alerting**:
   - Retrieval latency > 1s (degradation)
   - Vector DB unavailable
   - Embedding model errors
   - High hallucination rate (> 5%)

3. **Optimization**:
   - Cache query embeddings (24h TTL)
   - Batch embedding updates
   - Tune chunk size based on retrieval quality
   - A/B test re-rankers

Exit criteria:
- Monitoring dashboard deployed
- Alerts configured and tested
- Optimization strategy documented

---

## 5. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Embedding Model | BAAI bge-base-en-v1.5 (local) | OpenAI text-embedding-3-small (cloud) | Use local for privacy/cost; cloud for scalability/quality |
| Vector Database | Qdrant (< 100M vectors) | Milvus (> 100M) or Pinecone (managed) | Scale and ops tolerance determine choice |
| Chunking Strategy | 256–512 tokens with 50–100 overlap | Fixed 1K token chunks | Semantic boundaries outperform fixed sizes |
| Re-ranking | Cross-encoder (slow, accurate) | BM25 hybrid (fast) | Accuracy vs latency tradeoff |
| Context Assembly | Semantic + business rules ranking | Random sampling | Deterministic ranking ensures reproducibility |
| Caching Strategy | Query embedding cache (24h) | No cache | Cache layer reduces load 60–80% |

---

## 6. Validation Strategy

### Functional Validation
- Unit tests for chunking (boundaries, overlap, token count).
- Integration tests for embedding (model consistency, cache hits).
- End-to-end tests: document → embed → search → retrieve → generate.
- Retrieval quality tests (ground truth evaluation set).

### Failure Validation
- Embedding model unavailable (fallback to cached embeddings).
- Vector DB timeout (circuit breaker, cached results).
- No relevant documents found (graceful degradation).
- Hallucination under poor context (test with incomplete docs).

### Quality Validation
- Retrieval NDCG (Normalized Discounted Cumulative Gain) >= 0.7
- Hallucination rate <= 5% (manual review)
- Citation accuracy (retrieved docs match claims) >= 95%
- Latency p95 <= 2 seconds (retrieval + generation)

**Example Evaluation**:
```python
from sklearn.metrics import ndcg_score
import numpy as np

def evaluate_retrieval(queries, ground_truth_docs, retrieved_results):
    """Evaluate retrieval quality using NDCG."""
    scores = []
    for query, true_docs, retrieved in zip(queries, ground_truth_docs, retrieved_results):
        # Binary relevance (0 = not relevant, 1 = relevant)
        y_true = [1 if doc in true_docs else 0 for doc in retrieved]
        y_score = list(range(len(retrieved), 0, -1))  # Rank positions
        
        ndcg = ndcg_score([y_true], [y_score], k=5)
        scores.append(ndcg)
    
    avg_ndcg = np.mean(scores)
    print(f"NDCG@5: {avg_ndcg:.3f}")
    return avg_ndcg >= 0.7
```

---

## 7. Benchmarks and SLO Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Retrieval Latency (p95)** | <= 500ms | Includes embedding + search |
| **Generation Latency (p95)** | <= 1.5s | LLM streaming to first token |
| **Total RAG Latency (p95)** | <= 2.0s | End-to-end user experience |
| **Embedding Model Throughput** | >= 1K/sec (batched) | Indexing throughput |
| **Vector Search QPS** | >= 100 queries/sec | Per instance |
| **Retrieval NDCG@5** | >= 0.7 | Relevance metric |
| **Hallucination Rate** | <= 5% | Manual review sample |
| **Citation Accuracy** | >= 95% | Source attribution correctness |
| **Index Freshness** | <= 1 day | Document update latency |
| **Availability** | >= 99.5% | Vector DB + embedding service |
| **Vector DB Storage Efficiency** | 10–15 bytes per vector | Compression + metadata |

---

## 8. Risks and Controls

- **Risk:** Embedding model hallucination (poor retrieval quality).
  - **Control:** Use re-ranker; validate NDCG >= 0.7; monitor hallucination rate.
  
- **Risk:** Vector DB scale (> 1B vectors, latency degradation).
  - **Control:** Implement sharding; migrate to Milvus; monitor query latency.
  
- **Risk:** Stale documents (outdated information indexed).
  - **Control:** Track document version; expire old chunks; re-index on schedule.
  
- **Risk:** Embedding drift (model updates break similarity scores).
  - **Control:** Version embedding model; re-embed on major updates; A/B test new models.
  
- **Risk:** Privacy leakage (sensitive data in context).
  - **Control:** Implement document access control; audit retrieval logs; filter by user permissions.
  
- **Risk:** Cost explosion (large indexing, frequent searches).
  - **Control:** Batch embeddings; cache query embeddings; use local models for high-volume.
  
- **Risk:** Latency SLA violation (search > 500ms).
  - **Control:** Tune chunk size; add caching layer; optimize vector DB indexes.

---

## 9. Agent Execution Checklist

- [ ] Embedding model selected (local vs cloud) and tested for latency/quality.
- [ ] Vector database deployed (self-hosted or cloud) with persistence.
- [ ] Document chunking pipeline implemented with versioning.
- [ ] Indexing pipeline complete (embed → upsert) with error handling.
- [ ] Retrieval pipeline with semantic search working.
- [ ] Re-ranker integrated (cross-encoder or BM25).
- [ ] Context assembly deterministic and enforces token limits.
- [ ] LLM augmentation with citations implemented.
- [ ] Monitoring dashboard deployed (latency, NDCG, hallucination).
- [ ] Evaluation framework in place (NDCG, citation accuracy, manual review).
- [ ] Caching layer added (query embeddings, search results).
- [ ] Backup/restore tested for vector DB and metadata.
- [ ] Documentation complete (architecture, runbooks, troubleshooting).
- [ ] Load testing passed (target QPS, latency SLOs).
- [ ] Production readiness checklist signed off.

---

## 10. Reuse Notes

This module is reusable across any RAG system. Adapt only:
- **Embedding model**: Choose based on latency/quality/cost constraints.
- **Vector database**: Scale and ops tolerance (self-hosted vs managed).
- **Chunk size**: Tune based on document type and domain.
- **Re-ranker**: Swap for domain-specific cross-encoder if needed.
- **Context window limit**: Adjust for your LLM's context limit.
- **SLO targets**: Scale based on user tier (e.g., premium users get < 1s latency).
- **Re-indexing frequency**: Document-dependent (news daily, docs weekly).

---

## 11. Production Readiness Rubric

A RAG system is production-ready only if all are true:
- ✅ Embedding model selected with documented latency/cost/quality tradeoffs.
- ✅ Vector database deployed, backed up, and scaled for peak load.
- ✅ Retrieval quality validated (NDCG >= 0.7, hallucination <= 5%).
- ✅ End-to-end latency SLO met (< 2s).
- ✅ Monitoring and alerting configured.
- ✅ Runbooks written for common incidents (DB unavailable, high latency).
- ✅ Access control and privacy audited.
- ✅ Capacity planning for 3–6 month growth documented.

---

## 12. Reference Implementation Examples

### Example 1: Local RAG with Qdrant + BAAI bge-base-en-v1.5
```python
# docker-compose.yml
version: '3.8'
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - ./qdrant_storage:/qdrant/storage

# main.py
from sentence_transformers import SentenceTransformer
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

# Initialize
model = SentenceTransformer("BAAI/bge-base-en-v1.5")
client = QdrantClient("http://localhost:6333")

# Create collection
client.recreate_collection(
    collection_name="my_docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
)

# Index documents
documents = ["Machine learning is...", "LLMs can..."]
embeddings = model.encode(documents)
client.upsert(
    collection_name="my_docs",
    points=[...],  # PointStruct with embeddings
)

# Search
query_embedding = model.encode("What is machine learning?")
results = client.search(
    collection_name="my_docs",
    query_vector=query_embedding,
    limit=5,
)
```

### Example 2: Cloud RAG with Pinecone + OpenAI Embeddings
```python
import pinecone
from openai import OpenAI

# Initialize
pinecone.init(api_key="YOUR_KEY", environment="us-west1-gcp")
index = pinecone.Index("my-index")
client = OpenAI()

# Create index
pinecone.create_index(
    name="my-index",
    dimension=1536,
    metric="cosine",
)

# Index documents
documents = ["Machine learning is...", "LLMs can..."]
response = client.embeddings.create(
    input=documents,
    model="text-embedding-3-small",
)
embeddings = [emb.embedding for emb in response.data]

# Upsert to Pinecone
index.upsert(vectors=[(f"doc-{i}", emb, {}) for i, emb in enumerate(embeddings)])

# Search
query_embedding = client.embeddings.create(
    input="What is machine learning?",
    model="text-embedding-3-small",
).data[0].embedding

results = index.query(vector=query_embedding, top_k=5)
```

### Example 3: Hybrid RAG with Re-ranking
```python
from sentence_transformers import CrossEncoder

# Re-ranker
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")

# Retrieve candidates
candidates = [...]  # From vector DB

# Re-rank
pairs = [[query, doc] for doc in candidates]
scores = reranker.predict(pairs)
ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)

# Return top-k
return [doc for doc, _ in ranked[:3]]
```

---

## 13. Troubleshooting Runbook

### Issue: High Retrieval Latency (> 1s)
1. Check vector DB health: `client.get_collection_info()`
2. Profile query latency: `search_start = time.time()`
3. If latency in embedding: reduce batch size or use smaller model
4. If latency in search: add caching, reduce top-k, tune indexes
5. If latency in re-ranking: use BM25 instead of cross-encoder

### Issue: Low Retrieval Quality (NDCG < 0.6)
1. Verify embedding model: test with known good queries
2. Check chunk size: too large (> 512) loses specificity
3. Try larger embedding model (bge-large)
4. Add re-ranker if not present
5. Evaluate on ground truth set to identify patterns

### Issue: High Hallucination Rate (> 10%)
1. Reduce context window (truncate to top-2 results)
2. Increase confidence threshold for accepted documents
3. Test with stronger LLM model
4. Add instruction: "Only use provided context"
5. Implement fact-checking layer

### Issue: Vector DB Unavailable
1. Check DB health: `client.get_collection_info()`
2. Check logs: `docker logs qdrant`
3. Restart if crashed: `docker-compose restart qdrant`
4. If disk full: expand storage and re-index
5. Failover to cached results (if available)

---

## 14. Cost Optimization Strategies

1. **Batch Embedding**: Embed 1000+ documents at once (reduce API calls 100×)
2. **Query Caching**: Cache embeddings for repeated queries (reduce embedding costs 50–80%)
3. **Local Models**: Use BAAI bge-base (free) instead of OpenAI ($0.02 per query)
4. **Chunk Optimization**: Right-size chunks to reduce total embeddings needed
5. **Vector DB**: Self-hosted Qdrant ($0 ops) vs Pinecone ($75/month minimum)
6. **Re-ranker Sampling**: Only re-rank top 20 instead of all results

**Cost Comparison (1M queries/month)**:
- OpenAI Embeddings: $0.02 × 1M = $20,000/month
- Local BAAI + Qdrant: $0 (your GPU)
- Pinecone: $75/month + $0.02 embedding cost = ~$20,075/month
- AWS Bedrock Titan: $0.0013 × 1M = $1,300/month

---

## 15. Advanced Topics (Optional)

### Multi-Tenant RAG with Access Control
```python
# Metadata-based filtering
results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[FieldCondition(key="tenant_id", match=MatchValue(value=user_tenant_id))]
    ),
    limit=5,
)
```

### Hybrid Search (Vector + Full-Text)
```python
# Elasticsearch with vector support
es = Elasticsearch(["http://localhost:9200"])

# Vector + BM25 hybrid
results = es.search(
    index="documents",
    body={
        "query": {
            "bool": {
                "must": [
                    {"knn": {"embedding": {"vector": query_embedding}}},
                    {"match": {"content": query_text}},
                ]
            }
        },
    },
)
```

### Domain-Specific Fine-Tuning
```python
# Fine-tune embedding model on domain corpus
from sentence_transformers import SentenceTransformer, losses
from torch.utils.data import DataLoader

model = SentenceTransformer("BAAI/bge-base-en-v1.5")
train_examples = [InputExample(texts=[query, pos], label=1)]
train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=32)
train_loss = losses.CosineSimilarityLoss(model)

model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=1,
    warmup_steps=100,
)
```

