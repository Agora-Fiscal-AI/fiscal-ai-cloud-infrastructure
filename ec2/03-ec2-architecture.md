# EC2 Architecture
**Fiscal AI Cloud Infrastructure -Compute Layer Design**
This document describes the architecture, deployment model, and internal structure of the EC2 Worker Node responsible for all heavy ingestions tasks within the Fiscal AI porcessing pipeline

## 1. Rola of EC2 in the ingestion pipeline
The EC2 Worker is the core processing engine of the system. It executes tasks that requiere:
- Persistent execution environment
- Custom Python libraries (NLP, parsing PDF/XML processing)
- Higher CPU and memory capacity than Lambda allows.
- Long-running workflows (chunking, batch embeddings)
- Network access to S3, Bedrock, and RDS
  
EC2 receives tasks from SQS, processes them, stores outputs in S, and notifies downstream services via S3 event triggers.

## 2. High-Level Architecture Diagram
```javascript
     S3 /raw/ (document uploaded)
                  │
                  ▼
         Lambda (1) – ingestion check
                  │
                  ▼
               SQS Queue
                  │ (pull)
                  ▼
     ┌────────────────────────────┐
     │        EC2 Worker          │
     │  - normalization           │
     │  - canonical JSON          │
     │  - chunking (CPU)          │
     │  - embeddings (Titan)      │
     └────────────────────────────┘
                  │
         uploads to S3 (/normalized/, /embeddings/)
                  │
                  ▼
        Lambda (2) – RDS insert
                  │
                  ▼
            PostgreSQL + pgvector
```

## 3. EC2 Instance Specifications
### 3.1 Instance Type
For this initial project phase:
- t3.large
- 2 vCPU
- 8 GB RAM
- Cost-effective
- Suitable for chunking + Bedrock API embedding calls
Future scalability can move to:
- c5.xlarge or c6i.large for CPU-intensive chunks.
- g5.xlarge if CPU acceleration becomes neccesary.

## 4. EC2 Networking & IAM
### 4.1 VPC Placement
- Private subnet recommended
- Outbound access via NAT Gateway (for Bedrock + S3 + RDS traffic)
### 4.2 Security Groups
- Allow otbound HTTPS
- No inbound access except SSH (restricted to IP or via Session Manager)
### 4.3 IAM Role for EC2 Instance
Must include:
- s3:GetObject, s3:Putboject
- sqs: ReceiveMessage, sql:DeleteMesasage
- bedrock:InvokeModel
- rds-db:connect (if using IAM auth later)

## 5. Internal Architecture of EC2 Worker

### The EC2 machine will run a worker application consisting of multiple internal modules:
```arduino

ec2-worker/
 ├── config/
 │    └── settings.yaml
 ├── ingestion/
 │    ├── sqs_listener.py
 │    ├── file_downloader.py
 ├── normalization/
 │    ├── xml_to_json.py
 │    ├── metadata_extractor.py
 ├── chunking/
 │    ├── chunker.py
 │    └── nlp_preprocessor.py
 ├── embeddings/
 │    ├── titan_embedder.py
 │    └── vector_formatter.py
 ├── s3/
 │    ├── uploader.py
 │    └── path_resolver.py
 ├── utils/
 │    ├── logger.py
 │    └── exceptions.py
 └── main.py
```

## 6. EC2 Worker Responsabilities
### 6.1 SQS Message Consumption
- Long-polling strategy
- Retry with exponential backoff
- Deletes messages only when processing succeeds
### 6.2 Document Retrieval
- Downloads raw files from S3
- Validates MIMME type, size, metadata
### 6.3 Normalization
- Convert PDF->XML->Canonical JSON
### 6.4 Chunking
- Semantic or structural chunking.
- Detect articles, sections, clauses.
- Generate enriched metadata.
```json
{
  "chunk_id": "123-045",
  "law_id": "123",
  "position": 45,
  "text": "El impuesto al valor agregado...",
  "tokens": 382
}

```
### 6.5 Embeddings Generation
- Calls Amazon Titan Embeddings via Bedrock
- Handles batch requests
- Formats vectors into pgvector-ready rows:
```json
{
  "chunk_id": "123-045",
  "vector": [0.123, -0.55, ...],
  "metadata_path": "s3://.../chunks/123/045.json"
}

```
### 6.6 Storing Outputs
Uploads results to:
```perl
s3://fiscal-ai/normalized/<law_id>.json
s3://fiscal-ai/chunks/<law_id>/<chunk_id>.json
s3://fiscal-ai/embeddings/<law_id>_vectors.jsonl

```

## 7. Operational Features
### 7.1 Logging
- Structured logs using JSON
### 7.2 Error Recovery
- Non-fatal errors send the message back to SQS
- Fatal errors logged + archived in /failed/ directory
### 7.3 Scaling Strategy
- Manual scaling in early stages
- Future migration path:
- EC2 -> Auto Scaling Group
- EC2 -> ECS Fargate
- EC2 -> Step Fuctions + Lambda-only pipelines
  
## 8. Future Enhancements

| Feature                                                  | When     | Notes                      |
| -------------------------------------------------------- | -------- | -------------------------- |
| Migrate embedding workload to ECS Fargate (GPU optional) | Later    | Cost + speed benefits      |
| Replace long-running EC2 scripts with Step Functions     | Later    | Better orchestration       |
| Switch to IAM-based RDS auth                             | Later    | Improves security          |
| Implement batching queue for large ingestion             | Later    | Supports massive backfills |

