# Overview - PostgreSQL + pgvector

The Fiscal AI Platform leverages PostgreSQL with the pgvector extension as its vector database for storing embeddings generated from legislative documents.
This choice provides a robust, scalable, and AWS-native solution for vector similarity search while keeping operational complexity low.

## Why pgvector?

- Fully managed using Amazon RDS
Benefit from automated backups, monitoring, updates, and high availability.
- Strong consistency and durability guarantees
Guarantees reliable vector storage and deterministic query results—critical for legal and regulatory workloads.
- Easy integration with LangChain
Works natively with LangChain’s vector store interfaces and retrieval modules.
- Supports hybrid seacrh (vector + keyword)
Allows combining semantic search (ANN) with traditional SQL filters and full-text queries.
- Scales easily to Aurora in the future
Deployment can scale vertically via RDS instance classes, or evolve into Amazon Aurora PostgreSQL for higher throughput and distributed storage.

This document set provides a complete step-by-step guide to deploying PostgreSQL with pgvector on AWS.