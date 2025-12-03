# 05 — Schema Design for Vector Search (PostgreSQL + pgvector)

This document defines the schema for the Fiscal AI Platform’s vector database.
The database is used primarily to store:

- Embeddings generated from legislative text chunks
- Minimal metadata required for retrieval, filtering, and traceability
- Links to the actual original structured documents stored in S3

The system intentionally avoids storing full XML or JSON content inside PostgreSQL to ensure:
- Low storage footprint
- Faster vector search performance
- Clear separation between large document storage (S3) and vector retrieval (RDS)
- Simplified future scaling

---

## 1. Overall Design Philosophy

The Fiscal AI Platform uses PostgreSQL + pgvector as a **vector store**, not as a document store.  
The full content (PDF, XML, JSON, normalizations, enriched structures) will live in **Amazon S3**, while PostgreSQL stores only:

### **Minimal metadata + embedding per chunk**

This ensures:

###  Fast similarity search  
Embeddings remain small and indexed efficiently (HNSW / L2 ops).

###  No duplication of large legal documents  
XML and JSON can be 300KB–3MB per law. Storing that in PostgreSQL would slow down queries, backups, and increase storage cost.

###  Clean decoupling between:
- **Document storage** layer (S3)  
- **Structured metadata** (PostgreSQL)
- **Semantic search** (pgvector)

---

## 2. Recommended Schema

### **Table: document_chunks**

This is the ONLY required table for the MVP.

```sql
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Reference to original document in S3
    s3_uri TEXT NOT NULL,

    -- Identifiers to support filtering, grouping, and traceability
    law_name TEXT NOT NULL,
    law_year INT,
    title TEXT,
    chapter TEXT,
    section TEXT,

    -- Chunk-level text content (small excerpts only)
    chunk_text TEXT NOT NULL,

    -- Vector embedding
    embedding vector(1000),

    -- Metadata stored as flexible JSON (optional)
    metadata JSONB,

    created_at TIMESTAMP DEFAULT NOW()
);
