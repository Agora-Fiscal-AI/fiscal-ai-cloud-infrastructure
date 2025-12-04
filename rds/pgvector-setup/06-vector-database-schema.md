# 06 — Vector Database Schema (PostgreSQL + pgvector)

## Overview
This document defines the vector database schema for Fiscal AI Platform, built on PostgreSQL + pgvector.
The database stores legal documents and their semantic embeddings to enable fast and accurate Retrieval-Augmented Generation (RAG) pipelines.

The schema is structured around:
- A mastertable for legal documents(laws)
- A chunk table containing text segments + embeddings(law chunks)
- Vector idexes using pgvector
- Metadata and references to structured documents stored in **Amzon S3**

This design emphasizes:
- Scalability
- Clean separation of concerns
- High-performance vector search
- Flexibility for versioning and document lineage

S3 is used only for.
- Raw documents (PDF,DOCX,XML, JSON)
- Processed full-text JSON representations

The intellgence layer (metadata + chunk text + embeddings) lives in PostgreSQL

## Why split into two tables?
- laws stores the document-level information.
- law_chunks stores chunk-level vector data.

**Benefits:**
- Each document may contain thousands of chunks → horizontal scalability.
- Clean separation between document metadata and vector data.
- Reduces duplication and storage overhead.
- Enables versioning without embedding conflicts.
- Allows efficient retrieval for RAG using chunk-level granularity.

## Table: laws (Master Legal Documents)
This table represents a complete legal document(law,code,regulation,guideline...).

### Fields
| **Field**            | **Type**      | **Description**                                   |
|----------------------|---------------|---------------------------------------------------|
| `id`                 | UUID          | Unique identifier                                 |
| `title`              | TEXT          | Official name of the law                          |
| `jurisdiction`       | TEXT          | `federal` / `state` / `municipal`                 |
| `category`           | TEXT          | e.g., `fiscal`, `administrative`, etc.            |
| `version`            | TEXT          | Version label, e.g., `"2024-revision"`            |
| `is_active`          | BOOLEAN       | Indicates if the law is currently valid           |
| `effective_date`     | DATE          | When the law becomes effective                    |
| `expiration_date`    | DATE          | When it expires *(optional)*                      |
| `created_at`         | TIMESTAMP     | Timestamp when the record was stored              |
| `json_s3_url`        | TEXT          | S3 URL of the **processed JSON** document         |
| `source_pdf_s3_url`  | TEXT          | S3 URL of the **original PDF**                    |

```sql

CREATE TABLE laws (
    id UUID PRIMARY KEY,
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

```
## Table: law_chunks (Vector Store)
This is where embeddings and chunk text are stored

### Fields
| **Field**        | **Type**         | **Description**                                   |
|------------------|------------------|---------------------------------------------------|
| `id`             | UUID             | Chunk ID                                          |
| `law_id`         | UUID             | Foreign key referencing `laws.id`                 |
| `chunk_index`    | INTEGER          | Order of the chunk inside the parent document     |
| `text`           | TEXT             | Text content of the chunk                         |
| `embedding`      | VECTOR(1536)     | Titan Embeddings G1 vector                        |
| `metadata`       | JSONB            | Chapter, section, pagination, or extra metadata   |
| `json_s3_url`    | TEXT             | S3 URL of the per-chunk processed JSON            |
| `created_at`     | TIMESTAMP        | Storage timestamp                                 |


```sql
CREATE EXTENSION IF NOT EXISTS vector;

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

CREATE INDEX law_chunks_embedding_idx
ON law_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```
- 1536 corresponds to Amazon Bedrock Titan Embeddings G1 dimension.

## Processing Pipeline Overview (Simplified)

### Inputs
- PDF Legal docuemnts
- Docx Word documents
- XML structured documents
- Pre-extracted text

### Steps
1. Upload raw file -> S3
2. Extract and clean text
3. Chunk into meaningful segments
4. Generate embeddings using Titan of Bedrock
5. Insert into:
- laws
- law_chunks
6. pgvector indexes embeddings for fast similarity search

## RAG Query Example:
- LangChain executes queries like:
```sql
    SELECT text, metadata
    FROM law_chunks
    ORDER BY embedding <-> $QUERY_VECTOR
    LIMIT 5;
```

## Example Entry
### Law
```json
{
  "id": "2d8a1e26-845d-4e37-8b90-7a52af3efc54",
  "title": "Income Tax Law",
  "jurisdiction": "federal",
  "category": "fiscal",
  "version": "2024-01",
  "json_s3_url": "s3://fiscalai/laws/isr_2024.json"
}
```
### Chunk
```json
{
  "id": "fe98a5a8-3116-4e47-98da-07dd1c82643f",
  "law_id": "2d8a1e26-845d-4e37-8b90-7a52af3efc54",
  "chunk_index": 12,
  "text": "Article 25 — Authorized deductions in this section include...",
  "metadata": {
    "chapter": "Title II",
    "article": 25,
    "page": 7
  },
  "embedding_vector":[ ...1536 values... ],
  "json_s3_url": "s3://fiscalai/chunks/isr_2024/chunk12.json"
}
