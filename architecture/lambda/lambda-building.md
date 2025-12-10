# Lambda (1) Building Commands for Amazon CLI

## Step A.1 Creation of Lambda(1) Code Source located at:
```bash
mkdir lambda1 && cd lambda1

cat > lambda_function.py << 'EOF'
import json
import boto3
import urllib.parse
import os

s3 = boto3.client('s3')
sqs = boto3.client('sqs')

BUCKET = 'agora-fiscal-ai-docs'
QUEUE_URL = os.environ['QUEUE_URL']  # lo pasaremos como variable de entorno

def lambda_handler(event, context):
    for record in event['Records']:
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        size = record['s3']['object'].get('size', 0)

        print(f"Nuevo archivo: {key} (tamaño: {size} bytes)")

        # Validaciones tempranas
        if not key.startswith('raw/'):
            print("Ignorado: fuera de raw/")
            continue

        filename = key.split('/')[-1].lower()
        if not filename.endswith(('.pdf', '.docx', '.xml', '.json')):
            raise ValueError(f"Formato no soportado: {filename}")

        if size == 0:
            raise ValueError("Archivo vacío")
        if size > 50 * 1024 * 1024:
            raise ValueError("Archivo > 50 MB")

        # Generar law_id inteligente
        base_name = filename.rsplit('.', 1)[0].upper().replace(' ', '_').replace('-', '_')
        year = base_name[-4:] if base_name[-4:].isdigit() else 'DESCONOCIDO'
        law_id = f"{base_name}_{year}" if year != 'DESCONOCIDO' else base_name

        # Tagging del objeto
        tags = {
            'law_id': law_id,
            'stage': 'raw',
            'jurisdiction': key.split('/')[1],  # federal/estado/municipal
            'source': 'manual',
            'ingested_at': record['eventTime']
        }
        s3.put_object_tagging(
            Bucket=BUCKET,
            Key=key,
            Tagging={'TagSet': [{'Key': k, 'Value': str(v)} for k, v in tags.items()]}
        )

        # Mensaje enriquecido a SQS
        message = {
            "document_id": law_id,
            "s3_raw_key": key,
            "jurisdiction": tags['jurisdiction'],
            "version": year,
            "original_filename": filename,
            "event_time": record['eventTime'],
            "size_bytes": size
        }

        sqs.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps(message),
            MessageGroupId=law_id,
            MessageDeduplicationId=f"{law_id}-{filename}"
        )

        print(f"OK → {law_id} enviado a SQS")

    return {'statusCode': 200}
EOF


```
## Step A.2 - Creation IAM Role for Permissions
- In this step we declare an exclusive **Role** for Lambda(1), the sufficient policies of (S3,SQS,Lambda) for the perfect performance of ingestion processs.

```bash
aws iam create-role \
    --role-name LambdaAgoraFiscalValidatorRole \
    --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess

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