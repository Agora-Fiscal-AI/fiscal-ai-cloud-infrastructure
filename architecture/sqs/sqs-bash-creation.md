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
## Step A.3 Create access policy for S3 of send messages to SQS
- First we create a sqs-policy.json from AWS cloud shell using -cat
```bash
cat > sqs-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:s3:::agora-fiscal-ai-docs"
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

### Step A.4 Configure S3 Event Notification in order to send the message to SQS (not Lambda directly)
```bash
cat > s3-event.json <<EOF
{
  "QueueConfigurations": [
    {
      "QueueArn": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {   
          "FilterRules": [   #Declaration of filters to send the notification to SQS in case of accomplishment. Inside raw/filename.pdf o docx only
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
      "QueueArn": "arn:aws:sqs:us-east-1:921157071104:agora-fiscal-ingestion-queue",
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
# Set the new event to the s3 bucket configuration
aws s3api put-bucket-notification-configuration \
    --bucket agora-fiscal-ai-docs \
    --notification-configuration file://s3-event.json

```
### Step A.5 Tag all the queues for better cost and governance
```bash
aws sqs tag-queue \
--queue-url  https://sqs.us-east-1.amazonaws.com/921157071104/agora-fiscal-ingestion-queue \
--tags Project=agora-fiscal-ai,Environment=prodcution,Component=ingestion-queue

aws sqs tag-queue \
--queue-url https://sqs.us-east-1.amazonaws.com/921157071104/agora-fiscal-dlq \
--tags Project=agora-fiscal-ia,Environment=production,Component=dql
```
