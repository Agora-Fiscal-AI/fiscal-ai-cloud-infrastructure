# 02- Ingestion Use Cases

## Fiscal AI Cloud Infrastructure - EC2 Worker & Lambda Integration

This document describes the primary ingestion-related use cases supported by the current architecture. Each use case outlines how raw documents are processed, normalized, chunked, embedded, and persisted into storage such as S3 and PostgreSQL/pgvector

## 1. **Overview**

The ingestion pipeline enables Fiscal AI to transform unstructured legal/fiscal documents into structured, queryable vector-based data. This pipeline relies on:
- Lambda for triggering and lightweight logic
- S3 as the central document storage layer
- SQS for buffereing and decoupling ingestion workloads
- EC2 Worker for heavy processing (normalization, chunking embeddings)
- PostgreSQL + pgvector for long-term vector storage

Thus modular architecture ensures scalability, maintainability, and clear separation of responsabilities.

## 2. **Igestion Use Cases**

### 2.1 **New Document Upload (Main Use Case)
Triggered when a PDF, DOCX, or other supported format is uploaded to the S3 /raw/ directory.

Workflow
  1. User or automated system uploads a raw document to:
   - s3://fiscal-ai/raw/<document>
  2. Lambda (1) is triggered by the S3 event:
   - Validates the file
   - Extracts basic metadata 
   - Sends a processing request to the SQS ingestion queue
  3. EC2 Worker pulls the SQS message and performs;
    - Format normalization
    - Canonical JSON generation
    - Chunking
    - Embedding generation using Amazon Titan 
    - Upload of processed outputs to S3:
    - /normalized/<law_id>.json
    - /embeddings/<law_id>_vectors.jsonl
  4. Lambda(2) is triggered by the upload of embeddings:
   - Parses vector records
   - Stores them in PostgreSQL/pgvector
   - Inserts metadata into related tables

### 2.2 Law Revision or Version Update 
When a law is updated (new fiscal year), the updated version must replace or append the previous canonical representation.

- Workflow
- 1. Updated version is uploaded to /raw/
- 2. Standard ingestion pipeline executes
- 3. EC2 Worker tags the law version(version, valid_from, etc)
- 4. Lambda (2) inserts:
- New metadata
- New chunk embeddings
- 5. Old embeddings ramain available unless explicity marked obsolote
- 6. Canonical format in /normalized/ is versioned per law.

### 2.3 Metadara-Only Updates
Some updates involve changes in metadata without modifying the uderlying document.

- Examples:
- Jurisdiction corrections
- Naming standarization
- Tag updates
- Source corrections

- Workflow:
- 1. Metadata JSON is updated or uploaded to:
- /metadata-updates/<law_id>.json
- 2. Lambda (1) sends a metadata update message to SQS
- 3. EC2 Worker
- Reads normalized files
- Updates metadata 
- 4. Lambda (2) updates the laws table in RDS
- No chunking or embedding regeneration is requiered

### 2.4 Full Re-Embedding

Used when:
- New embeddings model is adopted
- Law format changes
- Chunking rules change
- QA issues requiere reprocessing
  
Workflow:
- Admin triggers a re-embedding job (simple SQS message)
- EC2 Worker loads the canonical JSON from /normalized/
- Re-runs chunking or only embeddings
- Uploads new vectors to /embeddings/
- Lambda (2) updates the vector table in PostgreSQL
Old vectors are archived or superseded depending on policy.

### 2.5 Backfill of Historical Documents

- When adding older fiscal codes, previous revisions, or historical DOF documents.
Workflow:
- 1. Bulk files uploaded to /raw/backfill/
- 2. Batch S3 → SQS → EC2 pipeline processes documents sequentially
- 3. Normalization, chunking, embeddings are executed as usual
- 4. Lambda (2) inserts data
- 5. Tagging strategy ensures historical versions are traceable:
```json
{
  "law_id": "ISR_2015",
  "historical": true,
  "source": "official_dof_archive"
}
```

## 3. Summary Table of Use Cases
| Use Case                  | Trigger Source          | Heavy Work on EC2 | RDS Store Needed | Notes                |
| ------------------------- | ----------------------- | ----------------- | ---------------- | -------------------- |
| **New Document Upload**   | S3 `/raw/` event        | Yes               | Yes              | Main ingestion flow  |
| **Law Revision / Update** | New file version        | Yes               | Yes              | Version-aware        |
| **Metadata-Only Update**  | Metadata file upload    | No / Minimal      | Yes              | No embeddings needed |
| **Full Re-Embedding**     | Admin-triggered SQS job | Yes (embeddings)  | Yes              | Model upgrades       |
| **Historical Backfill**   | Bulk upload to S3       | Yes               | Yes              | Uses same pipeline   |
| **Integrity Check**       | Scheduled job           | Optional          | Optional         | Maintenance          |

## 4. Conclusion

These ingestion use cases define how the system processes fiscal/legal documents throughout their lifecycle. The modular approach—splitting work across Lambda, EC2, S3, and RDS—ensures:
- Scalability
- Clear separation of responsibilities
- Easier debugging and monitoring
- Future compatibility with ECS, Step Functions, and GPU-based workloads