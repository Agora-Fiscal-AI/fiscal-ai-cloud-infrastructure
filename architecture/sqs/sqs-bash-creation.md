# SQS Bash Creation
## Step A.1 - Dead-Letter Queue Creation (DLQ)
```bash
aws sqs create-queue \
    --queue-name agora-fiscal-dlq \ 
    --region us-east-1 \
    --attributes '{
      "MessageRetentionPeriod": "1209600", #Seconds = 14 Days
      "VisibilityTimeout": "60"
    }'
```
- DLQ ARN: arn:aws:sqs:us-east-1:921157071104:agora-fiscal-dlq
## Step A.2 Main Ingestion Queue
```bash
aws sqs create-queue \
    --queue-name agora-fiscal-ingestion-queue \
    --region us-east-1 \
    --attributes '{
      "MessageRetentionPeriod": "1209600",
      "VisibilityTimeout": "3600",
      "ReceiveMessageWaitTimeSeconds": "20",
      "RedrivePolicy": {
        "deadLetterTargetArn": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-dlq",
        "maxReceiveCount": "4"
      }
    
```
- **Parameter Explanation**
- VisibilityTimeout = 1 hour -> enough time for EC2/ECS to process large documents.
- Long Polling 20 seconds -> for less requests per month
- maxReceiveCount = 4 -> after 4 failures go to DQL
- Retention 14 days -> enough time to look for the error
## Step A.3 Create access policy for Lambda of send messages to SQS
- First we create a sqs-policy.json from AWS cloud shell using -cat
```bash
cat > sqs-policy.json <<EOF
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
      "Resource": "$QUEUE_ARN$",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "$LAMBDA1_ARN"
        }
      }
    }
  ]
}
EOF
```
- Policy insertion command line process
```bash
#- Befeore adding the new .json policy, is necessary to compact the .json in one line:
jq -c '.' sqs-policy.json
#- Then its necessary to scape all the quotations
sed 's/"/\\"/g'
#- insert all the polivy into a string "Policy":"<here>"
cat > sqs-attributes.json <<EOF
{
  "Policy": "$POLICY_ESCAPED"
}
EOF
#- set the new policy into queue --attributes
aws sqs set-queue-attributes \
    --queue-url https://sqs.us-east-1.amazonaws.com/921157071104/agora-fiscal-ingestion-queue \
    --attributes file://sqs-policy.json
```
### Step A.4 Tag all the queues for better cost and governance
```bash
aws sqs tag-queue \
--queue-url  https://sqs.us-east-1.amazonaws.com/921157071104/agora-fiscal-ingestion-queue \
--tags Project=agora-fiscal-ai,Environment=prodcution,Component=ingestion-queue

aws sqs tag-queue \
--queue-url https://sqs.us-east-1.amazonaws.com/921157071104/agora-fiscal-dlq \
--tags Project=agora-fiscal-ia,Environment=production,Component=dql
```
