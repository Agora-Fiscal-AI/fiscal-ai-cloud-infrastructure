# Lambda1AgoraFiscalValidatorRole 
## Target
The unique purpose of this role is provide the necessary policies for AgoraFiscalLambdaValidator. For its accurable performance.
### Step A.1 Command Line for Cloud Shell to Create the Role.
- CLI command line
```bash
aws iam create-role \
    --role-name LambdaAgoraFiscalValidatorRole \
    --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
#Custom policy 
aws iam attach-role-policy --role-name LambdaAgoraFiscalValidatorRole --policy-arn arn:aws:iam::aws:policy/AgoraFiscalLambda1FullAccess
```
