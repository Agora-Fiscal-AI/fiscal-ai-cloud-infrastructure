# Fiscal AI Cloud Infrastructure V1

## Overview
This repository contains the cloud infrastructure documentation for the Fiscal AI Platform.
Its primary objective is to define, standardize, and document a minimal, scalable, and cost-efficient cloud architecture that supports the platform’s early development while enabling seamless long-term evolution.

# Initial Scope (V1):
This first version focuses on establishing the foundational components required for a secure and production-ready environment:
- A minimal architecure deployed on AWS
- Amazon RDS (PostgreSQL) configured with the pgvector extension for embedding storage
- Networking fundamentals: VPC design, subnets, routing, and security groups
- High-level architectural diagrams, decision records, and core design principles
- Future-proof structure that supports progressive enhancements such as CI/CD pipelines, Terraform IaC, EKS, Fargate, ECR, and ECS for DevOps.

## Repository Structure
- /
- ├── rds/            → Documentation for PostgreSQL, pgvector setup, backups, and best practices
- ├── network/        → VPC design, public/private subnets, routing tables, NAT, and security groups
- ├── architecture/   → System architecture diagrams, RAG flow, ADRs, and service interactions
- └── references/     → Cost estimations, AWS quotas/limits, roadmap, and additional resources

## Current Scope (MVP Infrastructure)
The Minimal Viable Product consists of:
- A single Amazon RDS PostgreSQL instance (Free tier eligible)
- pgvector extension for vector storage 
- S3 for PDF/DOCX document, and standarized XML/JSON strucutured knowledge storage before embedding service.
- EC2 or local notebooks using LangChain (as main framework) to interact with Amazon Bedrock and RDS services.
- Networking configured through a secure VPC with restricted access.

### This current scope does not consider using:
- Containers
- Orchestation
- Load Balancers
- CI/CD pipelines-yet

## Future Improvements
This repo is designed to expand with:
- Docker + ECR
- ECS Fargate or EKS
- IaC (Terraform)
- KMS Encryption
- API Gateway + Lambda integrations
- Distribuited RAG indexing services


