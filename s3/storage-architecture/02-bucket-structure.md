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
´´´psql
fiscal-ai-docs/
│
├── raw/
│   ├── federal/
│   │     └── <YEAR>/<original_files.pdf, docx>
│   ├── state/
│   └── municipal/
│
├── normalized/
│   └── <LAW_ID>/
│         ├── law.json                  # canonical JSON
│         ├── manifest.json             # metadata & stats
│         ├── sections/                 # optional future expansion
│         └── version-info.json         # historical info
│
├── chunks/
│   └── <LAW_ID>/
│         ├── chunks.jsonl              # all chunks in JSONL
│         ├── chunk-manifest.json       # metadata describing chunk count, errors
│         └── checkpoints/              # for resume operations
│               └── batch-<N>.json
│
├── embeddings/
│   └── <LAW_ID>/
│         ├── vectors.jsonl             # full embedding dump for EC2 → RDS
│         ├── embeddings-manifest.json  # metadata for lambda(2)
│         └── <LAW_ID>.ready            # event-trigger flag
│
├── logs/
│   └── <LAW_ID>/
│         ├── correlation.json          # flow tracking
│         ├── worker-log-<timestamp>.txt
│         └── errors/                   # any pipeline warnings
│
└── retry/
    └── <LAW_ID>/
          ├── recovery-note.json
          └── input-message.json        # original SQS message snapshot


´´´
  
## Folder Definitions
### 2. /raw/ — Entry Point for All Documents

This is where users upload files. Events from this directory trigger Lambda (1).
```psql
raw/
└── <jurisdiction>/<YEAR>/<filename>
```
Example:

- raw/federal/2025/ley-iva-2025.pdf
Metadata applied as S3 object tags:

```json
{
  "law_id": "IVA_2025",
  "stage": "raw",
  "source": "dof",
  "jurisdiction": "federal"
}
```


## 3. /normalized/<LAW_ID>/ — Canonical JSON Output

EC2 Worker writes normalized structured documents here.
```psql
normalized/
└── <LAW_ID>/
      ├── law.json
      ├── manifest.json
      ├── version-info.json (future implementation)
      └── sections/ (future implementation)

```
Example manifest.json
```json
{
  "law_id": "IVA_2025",
  "normalized_at": "2025-02-12T19:20Z",
  "sha256": "ae039f...",
  "chunks_expected": 542
}
```

## 4. /chunks/<LAW_ID>/ — All Chunking Artifacts

The EC2 Worker creates a complete set of semantic units.
```psql
chunks/
└── <LAW_ID>/
      ├── chunks.jsonl
      ├── chunk-manifest.json
      └── checkpoints/
           ├── batch-01.json
           ├── batch-02.json
           └── ...
```
Example chunks.jsonl
```jsonl
{"chunk_id":"IVA_2025_001","text":"...","page":2}
{"chunk_id":"IVA_2025_002","text":"...","page":3}
```
Example chunk-manifest.json
```json
{
  "law_id": "IVA_2025",
  "total_chunks": 542,
  "avg_length_tokens": 850
}
```

## 5. /embeddings/<LAW_ID>/ — Embedding Vector Dumps

Final EC2 job output, consumed by Lambda (2).
```psql
embeddings/
└── <LAW_ID>/
      ├── vectors.jsonl
      ├── embeddings-manifest.json
      └── <LAW_ID>.ready
```
Example vectors.jsonl
```json
{"chunk_id":"IVA_2025_001","vector":[0.11,0.02,...],"page":2}
{"chunk_id":"IVA_2025_002","vector":[0.09,0.14,...],"page":3}

```
Ready flag file:

- embeddings/IVA_2025/IVA_2025.ready


**This signals Lambda (2) to begin database insertion.**

## 6. /logs/<LAW_ID>/ — Worker Execution Logs

All worker logs, trace IDs, and diagnostic output.
```psql
logs/
└── <LAW_ID>/
      ├── correlation.json
      ├── worker-log-20250212.txt
      └── errors/
```
Example correlation.json
```json
{
  "correlation_id": "corr-IVA_2025-17381231",
  "document_id": "IVA_2025",
  "stages": ["raw","normalized","chunked","embedded"]
}
```

## 7. /retry/<LAW_ID>/ — Resumable Error Recovery

Used when EC2 is interrupted or encounters pipeline errors.
```psql
retry/
└── <LAW_ID>/
      ├── input-message.json
      └── recovery-note.json
```
Example recovery-note.json
```json
{
  "resume": true,
  "start_from": "embedding_batch_12",
  "reason": "spot interruption"
}

```
