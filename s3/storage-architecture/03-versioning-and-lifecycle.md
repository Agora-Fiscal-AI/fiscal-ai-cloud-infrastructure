# 03-versioning-and-lifecycle

## Purpose

This document defines the versining strategy, retention rules, and lifecycle configuration for the Amazon S3 bucket used in Agora Fiscal AI.
## The goal is to ensure:
- data itegrity
- legal auditability
- cost-efficient storage
- reproducibility of the entire processing pipeline
- support for future rollback and reprocessing
  
1. Versioning Strategy
## 1.1 Bucket Versioning
Versioning must be enabled on the S3 bucket:
- agora-fiscal-ai-docs
### Reasoning:
- Legal documents may be updated, corrected, or rectified by government authorities.
- Processing pipelines (normalization, chunking, embeddings) must be reproducible.
- Accidental overwrites must be reversible.
- Ensures long-term auditability of source material.

## 1.2 Versioned Directories
- Each law´s folder inside normalized/ may contain versions over time.
- The selected strategy is the following: Replacing canonical file
- Can overwrite: normalized/00123/law.json
Since S3 versioning is enabled, older versions remain accesible automatically.

2. Lifecycle Management
Lifecycle rules build cost efficiency and cleanup automation into the storage layer.

## 2.1 Lifecycle Rules Overview
Defined four lifecycle policies:

| Folder        | Retention Rule           | Reason                             |
|---------------|---------------------------|-------------------------------------|
| `raw/`        | Long-term retention       | Legal traceability                  |
| `processed/`  | 180-day retention         | Needed only for pipeline debugging  |
| `normalized/` | Permanent                 | Canonical dataset                   |
| `tmp/`        | Auto-delete after 7 days  | Temporary compute artifacts         |


## 2.2 Recommended Lifecycle Rules
### raw/ folder
- Retention: NEVER DELETE
- Rationale:
- Raw files represent the original source of truth
- Needed for legal audits, verifying OCR accuracy, or re-normalizing documents.
### processed/ folder
- Transition to Glacier: 60 days
- Delete: 180 days
### normalized/ folder
- Retention: NEVER DELETE
- Transition to Glacier Deep Archive: optional (future)
- Rationale:
- Canonical files are the heart of your RAG system.
- Required to rebuild embeddings, chunks, and even the Postgres metadata.
### tmp/ folder
- Delete after: 7 days
- Temporary files should never accumulate.

3. Metadata Tagging Standarts
To ensure traceability and integration with external services, all files uplodes to S3 should include consistent object tags.
| Tag            | Value                              | Purpose                            |
|----------------|------------------------------------|------------------------------------|
| `law_id`       | `<integer>`                        | Links S3 files to RDS `laws` table |
| `stage`        | raw / processed / normalized / tmp | Pipeline stage                     |
| `version`      | v1, v2, etc.                       | (Optional) explicit version        |
| `source`       | "dof" / "state-cdmx" / etc.        | Trace document origin              |
| `jurisdiction` | federal / state / municipal        | Classification                     |

## Example tag set:
``` json
{
  "law_id": "123",
  "stage": "normalized",
  "version": "2025.01",
  "source": "dof",
  "jurisdiction": "federal"
}
```

5. Naming Convenions
To ensure consistency, files inside S3 must follow predictable naming:
## 5.1 Raw Documents:
- raw/federal/2025/ley-iva.pdf
- raw/federal/2025/ley-iva.xml

## 5.2 Processed Files
- processed/00123/segments.json
- processed/00123/ocr-clean.txt

## 5.3 Normalized Canonical Files
- normalized/00123/law.json
- normalized/00123/metadata.json
- normalized/00123/manifest.json

## 5.4 Temporary Files
- tmp/00123/chunk-temp-2025-01-10.json