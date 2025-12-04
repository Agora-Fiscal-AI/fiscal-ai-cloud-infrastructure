# 02-bucket-structure.md — Bucket Structure & Data Organization
## Purpose
This document defines the directory structure, naming conventions, and organizational strategy used inside the Amazon S3 bucket for Ágora Fiscal AI.
A consistent structure ensures:
- predictable access paths
- cleaner automation
- easier data lineage tracing
- reduced ambiguity across raw/processed/normalized data
- seamless integration with the RDS pgvector pipeline

## Bucket Naming Convenion
For the current infrastructure, one bucket is sufficient:
- agora-fiscal-ai-docs
  
Later it can scale to the following structure:
- agora-fiscal-ai-docs
- agora-fiscal-ai-dev
- agora-fiscal-ai-stage
  
## Top-Level Folder Structure
- agora-fiscal-ai-docs/
-  ├── raw/
-  ├── processed/
-  ├── normalized/
-  └── tmp/
  
## Folder Definitions
1. raw/ - Original Source Documents
   Contains unmodified documents fetched from official sources such as (La Sombra de Arteaga or DOF Diario Oficial de la Federacion), state reposiories or local authorities
   ### Contents may include:
   - PDF files
   - DOCX files
   - XLSX (Future posible implementation)
  
    ### Example structure:
- raw/
- ├── federal/
- │    ├── 2025/
- │    │    ├── ley-impuestos.pdf
- │    │    
- ├── state/
- │    ├── Queretaro/
- │    │    ├── 2024/
- |    |    |    ├── Huimilpan (Municipe)/
- │    │    │    |   ├── ley-ingresos-qro.pdf

1. processed/ — Cleaned & Parsed Documents
Documents that have been transformed from raw format into partially structured data.
## Examples:
- XML parsed into preliminary JSON
- Extracted text segments
- Early metadata extraction
## Example:
- processed/
-  ├── 00123/                     # law_id from DB
-  │    ├── segments.xml
-  │    └── metadata.json
  
1. normalized/ — Canonical JSON Representation (Final Form)
This is the **most important folder** for the vector database pipeline
- Every law is stored in a canonical, standarized JSON fromat:
- normalized/<law_id>/<canonical-law>.json
  
This JSON is what:
- The chunking pipeline reads
- Metadata is extracted from
- pgvector embeddings originate from

### Example:
- normalized/
- ├── 00123/
- │    └── ley-del-iva.json
- ├── 00456/
- │    └── codigo-fiscal-federacion.json

### Each JSON file typically contains:
- law title
- article/chapter structure
- paragraphs
- table of contents
- references
- metadata
- version information
- date of validity
- jurisdiction level

4. tmp/ — Temporary Working Files

## Used for:
- intermediate results
- ephemeral processing steps
- staging areas for lambda/processing jobs
Files here may be deleted automatically via lifecycle rules.
### Example:
- tmp/
- ├── 00123/
- │    └── chunk-temp-2025-01-10.json
- |    └─  manifests.json
### Example of manifests.json
``` json
{
  "law_id": 123,
  "canonical_file": "law.json",
  "created_at": "2025-01-10T21:00:00Z",
  "normalization_pipeline": "norm_v1",
  "hash_sha256": "AB23984984F0A..."
}

```
5. Extra Notes
- Every time when a document is uploaded to S3, it must be uploaded with a Tag. 
- Example:
- Phase 1: A new raw doc uploaded:
```json
 {
  "law_id": "123",
  "stage": "raw",
  "source": "dof",
  "jurisdiction": "federal"
}
```
- Phase 2: Pipeline converts RAW -> Processed:
```json
{
  "law_id": "123",
  "stage": "processed",
  "pipeline": "normalize_v1"
}
```
- Phase 3: Generation to canonic JSON:
```json
{
  "law_id": "123",
  "stage": "normalized",
  "version": "2025.01",
  "jurisdiction": "federal"
}
```
