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

## 2.  Schema

### **Table: document_chunks**

This is the ONLY required table for the MVP.

```sql

CREATE TABLE laws (
    id UUID PRIMARY KEY,
    file_hash_sha256 CHAR(26)
    title TEXT NOT NULL,
    jurisdiction TEXT NOT NULL,
    category TEXT NOT NULL,
    version TEXT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    effective_date DATE,
    expiration_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    json_s3_url TEXT,
    source_pdf_s3_url TEXT
);


CREATE TABLE law_chunks (
    id UUID PRIMARY KEY,
    law_id UUID REFERENCES laws(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    text TEXT NOT NULL,
    embedding VECTOR(1536),
    metadata JSONB,
    json_s3_url TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);