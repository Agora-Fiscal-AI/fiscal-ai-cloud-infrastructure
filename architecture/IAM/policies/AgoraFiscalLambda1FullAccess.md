# AgoraFiscalLambda1FullAccess
## Policy Creation
### Step A.1 Json with the required permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-1:921157071104:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectTagging",
                "s3:PutObjectTagging",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::agora-fiscal-ai-docs",
                "arn:aws:s3:::agora-fiscal-ai-docs/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "sqs:SendMessage",
            "Resource": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue"
        }
    ]
}
```
### Step A.2 Create the managed policy 
```bash
aws iam create-policy \
    --policy-name AgoraFiscalLambda1FullAccess \
    --description "Complete permissions for lambda(1) (S3 tagging + SQS + logs)" \
    --policy-document file://agora-fiscal-lambda-policy.json
```