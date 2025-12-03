# Enabling pgvector on RDS PostgreSQL

Once the database is created, connec using psql or PGAdmin

## Run:

- CREATE EXTENSION IF NOT EXIXSTS vector;

## Verify:

- SELECT extanme FROM pg_extension;

- Expected output: 
- vector