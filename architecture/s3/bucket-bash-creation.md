# Step Bucket Creation Amazon CLI
## Bucket initial settigs
```bash
aws s3api create-bucket \
    --bucket agora-fiscal-ai-docs \
    --region us-east-1 \
    --create-bucket-configuration LocationConstraint=us-east-1 \ 
    --object-ownership BucketOwnerPreferred
```
## Block Public Access (Crucial)
```bash
aws s3api put-public-access-block \
    --bucket agora-fiscal-ai-docs \
    --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```
## Allow Bucket Versioning (Cricial for legal audit)
```bash
aws s3api put-bucket-versioning \
    --bucket agora-fiscal-ai-docs \
    --versioning-configuration Status=Enabled
```
## Encription with SSE-KMS (Future encryption, CRUCIAL BEFORE DEPLOYING)
- KMS key creation bash command:
```bash
aws kms create-key --description "Ágora Fiscal AI - Encriptación S3" --tags TagKey=Project,TagValue=agora-fiscal-ai
```
- Apply to bucket:
```bash
aws s3api put-bucket-encryption \
    --bucket agora-fiscal-ai-docs \
    --server-side-encryption-configuration '{
      "Rules": [
        {
          "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "aws:kms",
            "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc12345-def6-7890-ghi1-jkl234567890"
          }
        }
      ]
    }'
```

## Lifecycle Policy - Cost-Awareness and legal Retention (CRUCIAL ADJUST BEFORE DEPLOYING)
- Preliminar lifecycle suggested configuration:
```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket agora-fiscal-ai-docs \
    --lifecycle-configuration '{
      "Rules": [
        {
          "ID": "Move-raw-to-IA-after-30-days",
          "Filter": {"Prefix": "raw/"},
          "Status": "Enabled",
          "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"}
          ],
          "NoncurrentVersionTransitions": [
            {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"}
          ]
        },
        {
          "ID": "Delete-old-logs-after-365-days",
          "Filter": {"Prefix": "logs/"},
          "Status": "Enabled",
          "Expiration": {"Days": 365}
        },
        {
          "ID": "Permanent-retention-normalized-embeddings",
          "Filter": {"Prefix": "normalized/"},
          "Status": "Enabled"
        },
        {
          "ID": "Permanent-retention-embeddings",
          "Filter": {"Prefix": "embeddings/"},
          "Status": "Enabled"
        }
      ]
    }'
```
## Create Initial Folder Structure
```bash
# Main folders
aws s3api put-object --bucket agora-fiscal-ai-docs --key raw/
aws s3api put-object --bucket agora-fiscal-ai-docs --key normalized/
aws s3api put-object --bucket agora-fiscal-ai-docs --key chunks/
aws s3api put-object --bucket agora-fiscal-ai-docs --key embeddings/
aws s3api put-object --bucket agora-fiscal-ai-docs --key logs/
aws s3api put-object --bucket agora-fiscal-ai-docs --key retry/
aws s3api put-object --bucket agora-fiscal-ai-docs --key failed/

# Sub-directory example
aws s3api put-object --bucket agora-fiscal-ai-docs --key raw/federal/
aws s3api put-object --bucket agora-fiscal-ai-docs --key raw/estado/
aws s3api put-object --bucket agora-fiscal-ai-docs --key raw/municipal/
```