# AgoraFiscalEC2WorkerRole.md
## Target
This role grants access of the necessary policies to the EC2 Worker, the main brain of the entire architecture.
- CLI Command line Step by Step creation.

```bash
aws iam create-role \
  --role-name AgoraFiscalEC2WorkerRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect": "Allow", "Principal": {"Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}]
  }'
```
- Here we stablish the minimal necessary policies
```bash
aws iam attach-role-policy --role-name AgoraFiscalEC2WorkerRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam attach-role-policy --role-name AgoraFiscalEC2WorkerRole --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess
aws iam attach-role-policy --role-name AgoraFiscalEC2WorkerRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```