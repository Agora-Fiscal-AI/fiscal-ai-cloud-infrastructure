# EC2 Overview for Fiscal AI Cloud Infrastructure

Amazon EC2 (Elastic Compute Cloud) serves as the primary compute layer for heavy and flexible processing tasks within the Fiscal AI ingestion and cost-effective environment for development, experimentation, and CPU-intesive stages

- IMPORTANT NOTE: This is a preliminary stage, so, the using of ECS/Fargate and step functions is still not implemented within the infrastructure, nevertheless the implementation of this services is an unquestionably consideration for better cost-benefit optimization and modularity work-flow orchestation model.

## 1. Purpose of EC2 inthe current Architecture

EC2 hosts the components that require persistent compute power, custom libraries, or long-running execution. These tasks cannot be reliability executed with the time, memory, or evironment constrints of AWS Lambda.

EC2 is responsible for:

### **1.1 Data Ingestion**
- The EC2 worker performs the first heavy stage after the lightweight validation Lambda:
- Consumes messages from SQS (document ingestion jobs)
- Downloads raw source files from S3 /raw/
- Stages temporary files and prepares them for normalization
  
### **1.2 Document Normalization**
EC2 runs the CPU-intensive transformation steps:
- Converting PDF / DOCX → Canonical JSON
- Extracting structured metadata from legal/fiscal documents
- Validating section-level integrity
- Standardizing document schemas and labels
This step ensures that every law, rule, or annex follows the same internal structure before chunking.
  
### **1.3 Chunking Process and Natural Languaje Process**
The worker executes the NLP pre-embedding processes:
- Splitting normalized documents into meaningful semantic chunks
- Applying CPU-heavy linguistic processing (tokenization, cleaning, scoring)
- Preparing batches for embedding generation
- Producing embedding-ready JSON packages
These tasks are high-CPU and/or require specialized Python dependencies, making EC2 the right execution layer.


### **1.4 Embedding Generation**
Although future versions will migrate embeddings to ECS/Fargate + GPU if needed, EC2 currently:
- Loads language models for encoding process through LangChain framework (Titan) 
- Generates embeddings for each semantic chunk
- Produces vector files compatible with pgvector
- The worker then uploads results to:
S3 /normalized/
S3 /embeddings/
Triggering Lambda (2) for RDS insertion.

### **1.5 Scheduled and Utility Jobs**
- Cron-based ingeston cycles.
- Reprocessing otdated laws.
- Regenerating embeddings for updated versions.
- Maintenance scripts (cleanup, integrity checks)
