# RDS PostgreSQL Deployment Guide

## Step 1 - Create the RDS Instance
1. Open AWS Console -> RDS -> Databases -> Create database
2. Select:
    - Engine: PostgreSQL
    - Version 15.x (vector support)
    - Template: Free tier
3. DB Instance Identifier:
    - 'fiscal-ai-postgres'
4. Credentials:
    - Master username: 'fiscal_admin'
    - password: Store securely using AWS Secrets Manager or an encrypted local vault (never store in plaintext or version control).

## Step 2 - Instance Class
- db.t3.micro (free tier layer)
This instance type is sufficient for development and early-stage workloads.

## Step 3 - Storage
- Allocated Storage: 20 GB

- Storage Type: General Purpose SSD (GP2)

- Autoscaling: Disabled for initial deployment
- (Can be enabled later for scaling or production workloads.)

## Step 4 - Connectivity 
- VPC: (default or dedicated)
- Public access: NO
(RDS must not be publicly exposed.)
- Security Group Configuration:
- Allow inbound traffic on port 5432 only from:
- A trusted static IP, or a specific EC2 instance within the same VPC, or a VPN or bastion host
 

