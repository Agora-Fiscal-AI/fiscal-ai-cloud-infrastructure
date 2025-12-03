# RDS - PostgreSQL + pgvector

## Overview
This module contains all documentation related to the database layer of the Fiscal AI Platform.
Its focus is to establish a minimal, reliable, and production-ready foundation for vector storage, supporting both current functionality and future scalability without requiring architectural redesign.

## Contents
- Deployment of Amazon RDS (PostgreSQL)
Instructions and recommendations for provisioning a secure and scalable PostgreSQL instance on AWS.
- Enabling pgvector extensions
Steps required to activate and validate the pgvector extension for embedding storage and similarity search.
- Schema desgin for emeddings
Data modeling guidelines for storing vectors, metadata, and indexes efficiently.
- Connection procedures from LangChain
Best practices and examples for connecting LangChain to the RDS instance.
- Validation and testing
How to verify connectivity, run sample queries, perform similarity searches, and ensure performance expectations

## Direcotry Structure
-pgvector-setup/
-├── overview.md
-├── requirements.md
-├── rds-postgresql-deployment.md
-├── enable-pgvector-extension.md
-├── schema-design.md
-└── connection-test.md

