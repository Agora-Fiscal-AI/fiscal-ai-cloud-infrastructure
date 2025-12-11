# Lambda (1) Building Commands for Amazon CLI

## Step A.1 Creation of Lambda(1) Code Source located at:
- Repository URL: https://github.com/Agora-Fiscal-AI/fiscal-ai-lambda-functions.git
```bash
mkdir lambda1 && cd lambda1

```
## Step A.2 - Creation IAM Role for Permissions
- In this step we declare an exclusive **Role** for Lambda(1), the sufficient policies of (S3,SQS,Lambda) for the perfect performance of ingestion processs.
- Custom directory path: ../IAM/policies/
```bash
aws iam create-role \
    --role-name LambdaAgoraFiscalValidatorRole \
    --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
#Custom policy 
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AgoraFiscalLambda1FullAccess

```

## Step A.3 Deploy lambda function

```bash
zip function.zip lambda_function.py
aws lambda create-function \
    --function-name agora-fiscal-lambda-validator \
    --runtime python3.11 \
    #ARN of the previous Role attached on the Lambda properties
    --role arn:aws:iam::921157071104:role/Lambda1AgoraFiscalValidatorRole \  
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --timeout 30 \
    --memory-size 512 \
    #Attached the ingestion queue's URL
    --environment Variables={QUEUE_URL=arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue} \   
    --region us-east-1
```

## Step A.4 Apply Permissions to S3 to be able of invoke Lambda(1)
```bash
aws lambda add-permission \
    --function-name agora-fiscal-lambda-validator \
    --statement-id s3-invoke-validator \
    --action lambda:InvokeFunction \
    --principal s3.amazonaws.com \
    --source-arn arn:aws:s3:::agora-fiscal-ai-docs
```

## Step A.4 Connection from S3 to Lambda(1)
```bash
# We create the json-settings for the notification configuration in S3
cat > notification.json <<EOF
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:921157071104:function:agora-fiscal-lambda-validator",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "raw/"
            },
            {
              "Name": "suffix",
              "Value": ".pdf"
            }
          ]
        }
      }
    },
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:921157071104:function:agora-fiscal-lambda-validator",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "raw/"
            },
            {
              "Name": "suffix",
              "Value": ".docx"
            }
          ]
        }
      }
    }
  ]
}
EOF
# Attach the new event to the S3 bucket
aws s3api put-bucket-notification-configuration \
    --bucket agora-fiscal-ai-docs \
    --notification-configuration file://notification.json
```

## Step A.5 SQS Access Policy
- Define the access policy to ensure SQS receives messages only from lambda(1)
  
```json
{
  "Version": "2012-10-17",
  "Id": "AllowOnlyValidatorLambda",
  "Statement": [
    {
      "Sid": "AllowSendMessageFromValidatorLambda",
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:lambda:us-east-1:921157071104:function:agora-fiscal-lambda-validator"
        }
      }
    }
  ]
}
```