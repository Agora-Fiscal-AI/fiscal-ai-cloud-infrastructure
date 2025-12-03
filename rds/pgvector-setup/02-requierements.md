# Requieremnts

## AWS level requieremnts
- AWS account with free Tier enabled (until it expires )
- IAM user with permission for:
- RDS
- VPC
- IAM 

## Technical requierements
- PostgreSQL 15+
Required for compatibility with the latest features and extensions.
- pgvector extension (compatible with RDS)
Supported by Amazon RDS for PostgreSQL (engine version must be compatible).
- psql or PGAdmin for connection test.
Tools for running connectivity and SQL validation tests.
- Python environment for LangChain validation.
Required for LangChain connection tests and sample embedding queries.

## Networking requierements
- Private subnets dedicated to hosting the RDS instance.
- Public or private access depending on the development model (local development, EC2-based, or VPC-only).
- Security group rules allowing inbound access only from trusted IPs, VPN, or specific EC2 instances.
(Avoid exposing RDS to the internet.)
