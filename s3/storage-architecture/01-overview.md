# 01-overview.md (V1)

# Purpose
This document provides a high-level overview of the Amazon S3 storage architecture used by Agora Fiscal Ai

## S3 is responsible for storing from raw PDF/DOCX documents to structured documents (XML/JSON) that represent:
- Mexican fiscal laws on raw documents
- Extracted normalized documents
- Parsed canonical fragments (for future chunking and vectorization)
- Metadata files or pre-processed inputs
  
These docuemnts serve as source content for RDS-based vector embeddings using pgvector

## Role of S3 in the System
S3 Provides:
- Durable and scalable storage for legal source documents
- A clean separation between raw, processed, and normalized data
- A stable source of truth for the vector databse pipeline
- A way to reference documents inside the pgvector schema using document_s3_url.
  
Unlike storing raw documents directly in Postgres:
- S3 keeps data cheap and durable
- Postgres keeps only metadata + vector embeddings
- The system avoids unnecesary DB bloat
- Mantaining modular and scalable architecture.

## Data Flow Overview
1. Raw legal documents (PDF, XML, or JSON) are uploaded manually or programmatically to S3.
2. A processing service normalizes or transforms these files into XML and then transforms into canonical JSON.
3. The JSON files are stored in processed/ or normalized/ S3 folders.
4. Chunking logic ingests the canonical JSON from S3.
5. Chunks with metadata are stored in RDS (pgvector), referencing their origin via:
´´´ XML
s3://<bucket>/normalized/<law_id>/<file>.json
´´´
6. The vector embeddings pipeline loads these chunks and inserts the vectors into PostgreSQL.
   
## Example Core Use Cases for S3
- Store all legal documents un a hierarchical, predictable structure
- Maintain canonical normalized data for downstream indexing
- Provide links for every vectorized chunk in the pgvector table.
- Enable lifecycle pilicies for automatic retention (raw PDFs)
### Serve as a staging area for:
- incoming documents(raw)
- Preprocessed documents
- Final normalzed JSON ready for embeddings
  
## Relationship with RDS pgvector Tables
S3 integrates with RDS in the following way:
- Every chunk in table law_chunks includes:
- - source_s3_url -> JSON/normalized document
- - chunk_s3_url(optional future) -> chunk serialized JSON
- The laws table includes:
- - canonical_json_url referencing the full normalized law document
This ensures:
- Traceability
- Reproducibility
- Auditable data lineage
- Easy re-embedding in the future


## What’s NOT covered yet
Security and advanced topics will be done later:
- Bucket Policies
- IAM Roles
- Encryption (SSE-S3 / SSE-KMS)
- Cross-account or VPC endpoint access
- CloudFront public access rules
- Static website hosting (not required right now)
  

