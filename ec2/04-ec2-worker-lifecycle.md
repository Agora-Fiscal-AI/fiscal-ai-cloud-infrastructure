# EC2 Worker Lifecycle

## 1. Overview

This document specifies runtime lifecycle and operational behavior of the EC2 ingestion worker usedby the Fiscal AI platform.
It formalizes hpw the worker picks tasks from SQS, processes documents (normalize -> chunk -> embed), uploads artifacts to s3, and triggers persistence via Lambda. The goal is reliable, obeservable, and restart-safe wroker process that is ready for future migration to EC2 / Step Functions.

´´´yaml
BOOT
└─> INIT (load config, connect to AWS services)
└─> IDLE (long-poll SQS)
└─> PROCESSING (message received)
├─> DOWNLOADING (fetch raw from S3)
├─> NORMALIZING (XML/ocr → canonical JSON)
├─> CHUNKING (split into semantic chunks)
├─> EMBEDDING (Titan via Bedrock / batching)
├─> UPLOADING (S3 normalized & embeddings file)
└─> FINALIZE (remove SQS msg / push status)
└─> WAIT (backoff / throttling)
└─> SHUTDOWN (SIGTERM or manual)
´´´
## 2. Lifecycle Pseudocode:
```yaml

**Notes:**
- Any fatal error during PROCESSING moves the job to a `FAILED` path and optionally writes a record to S3 `/failed/` or DLQ for manual inspection.
- The worker is designed idempotently: re-processing the same SQS message should not create duplicates (use dedupe by `document_id` + `chunk_id` / conflict-safe DB inserts).

---

## 2. Core Runtime Loop (pseudocode)

```python
while not shutdown_requested:
    msg = sqs.receive_message(queue_url, WaitTimeSeconds=20, MaxNumberOfMessages=1)
    if not msg:
        continue  # long-poll returns nothing, keep waiting

    try:
        job = parse_message(msg)
        if already_processed(job.document_id, job.version):
            delete_message(msg)  # idempotence
            continue

        # mark start (optional checkpoint)
        process_document(job)
        delete_message(msg)

    except TransientError as e:
        # leave message in queue, optionally change visibility
        handle_transient_failure(msg, e)
    except FatalError as e:
        archive_failed_job(job, e)
        delete_message(msg)  
    except Exception as e:
        archive_failed_job(job, e)
        delete_message(msg)


```
## 3. SQS Polling & Message Handling
### 3.1 Polling strategy
- Use long polling (ReceiveMessage with WaitTimeSeconds=10..20) to reduce empty responses and cost.
- Pullo one message at a time(initially) to simplify processing and visibility management.

### 3.2 Visibility Timeout
- Set VisibilityTimeout to expected processing time + safety margin (e.g., 30-60 min for heacy documents)
- For longer jobs, periodically call ChangeMessageVisibility to extend the timeout while processing.
### 3.3 Dead-Letter Queue (DLQ)
- Attach a DLQ to the SQS queue with a bounded maxReceierCount (e.g., 3-5)
- Messaged that repeatedly fail move to DQL for manual inspection.
### 3.4 Message content 
SQS message should be a compact JSON:
```json
{
  "document_id": "ISR_2024",
  "s3_raw_key": "raw/federal/2024/ley-iva.pdf",
  "uploader_id": "user-42",
  "ingest_timestamp": "2025-01-12T15:31:45Z",
  "priority": "normal",
  "version": "2024-01"
}
```
## 4. Detailed Processing Workflow (step by step)
### 4.1 Validate $ Duplicate
1. Parse SQS message.
2. Check **laws** table or a ightweight processing-index for **document_id** + version
- if **proceced** and not **force_reprocess**, delete message and return.
- if **in-progress**, optionally extend visibility and skip to newt message.
### 4.2 Download Raw File
- Download object from S3 to local workspace (EBS instance storgae)
- Validate file format, size, and tags.
### 4.3 Normalize -> Canonical JSON
- Convert to PDF/DOCX -> strcutured XML -> canonical JSON 
- Run schema validation (JSON Schema/ tests)
- Produce **normalized/<document_id>/law.json and manifest.json**

### 4.4 Chunking
- Use languaje-aware chunker:Spit by sections, articles or semantic length (between 200-1200 tokens).
- For each chunk prouce:
- **chunk_id** (stable deterministic ID)
- **chunk_text**
- **chunk_meta** (Position,article,page)
- Save per-chunk JSON in **s3://.../chunks/<document_id>/<chunk_id>.json**
### 4.5 Embeddings (Batching)
- Group chunks into batches sized per Bedrock/Titan practices.
- Call Bedrock embeddings API via LangChain/boto3.
- If a batch call fails transiently: retry with exponential backoff and jitter
- Produce **embeddings** array aligned with chunk order.

### 4.6 Format Vectors for pgvector
- Create **embeddings** output file (JSONL) compatible with Lambda ingestion:
- One record per line: **{"chunk_id":"...","vector": [...], "law_id": "...", "metadata": {...}}**
- Add per-chunk metadata in the same or separate JSONL file.
### 4.7 Upload Outputs to S3
- Upload canonical JSON to **/normalized/<document_id>/law.json**
- Upload **chunks** and embeddings artifacts to:
- **/chunks/<document_id>/...**
- **/embeddings/<document_id>_vectors.jsonl**
### 4.8 Finalize & Checkpoint
- Insert or update an entry in laws table: **status = "normalized" / status = "embedded" and chunk_count**.
- Delete SQS message (if everything succeeded).
- Emit metrics and structured logs.
### 5. Error Handling & Recovery
## 5.1 Failure Classes
- Transient Errors: network issues, Bedrock throttling, S3 timeouts.
- Action: retry with backoff; extend SQS visibility while retrying.
- Permanent / Fatal Errors: corrupt file, invalid schema, unrecoverable parsing errors.
- Action: archive raw file to /failed/, push an error record to monitoring dashboard, move or delete message (or send to DLQ).
- Partial Failures: some chunks embedded, others failed.
- Action: persist partial results, mark document partial_failed, and create a reprocess job (SQS) for remaining chunks or full re-embedding.
## 5.2 Idempotency
- Ensure chunk IDs are deterministic (e.g., hash(document_id + chunk_index)) so re-processing does not create duplicates.
- Use INSERT ... ON CONFLICT DO UPDATE patterns for DB writes.
- Consider using a processing-lock table to mark in_progress with timestamp; stale locks can be reclaimed.