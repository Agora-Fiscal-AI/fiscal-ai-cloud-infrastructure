# ec2-worker-creation 

## Step A.1 Create Instance Profile
- AgoraFiscalEC2WorkerRole creation located at: ../IAM/roles/
```bash
aws iam create-instance-profile --instance-profile-name AgoraFiscalEC2WorkerProfile
aws iam add-role-to-instance-profile --instance-profile-name AgoraFiscalEC2WorkerProfile --role-name AgoraFiscalEC2WorkerRole
```
## Step A.2 Create a High-Restricted Securuty Group 
- Documentation located at: ../../networking/ec2-securty-groups/
- **Creation of Security Group:**

```bash
    aws ec2 create-security-group \
    --group-name AgoraWorkerSG \
    --description "Restricted outbound just HTTP & SSH" \
    
    # Outbound rules just for (Bedrock, S3)
    aws ec2 authorize-security-group-egress \
    --group-id sg-0abcd1234efgh5678 \
    --protocol tcp --port 443 --cidr 0.0.0.0/0

    # Temporal SSH connection restricted by unique IP 
    aws ec2 authorize-security-group-ingress \
    --group-id sg-0abcd1234efgh5678 \
    --protocol tcp --port 22 --cidr IP/32
```

## Step A.4 User Data Archive Before Deploying
- We create the user data archive with temporary python script
```bash
# Content of user-data file.sh with all the worker content 

#!/bin/bash
yum update -y
yum install -y python3 python3-pip git  jq awscli
# Work Directory creation
mkdir -p /home/ec2-user/agora-worker
cd /home/ec2-user/agora-worker
chown ec2-user:ec2-user /home/ec2-user/agora-worker
# Some Dependencies
pip3 install boto3 langchain-aws pydantic 

```
- Once created the user data content, we create the file .sh to upload it into the the launching command line
```bash
cat > user-data.sh << 'EOF'
# content
EOF
```


## Step A.5 Launch The EC2 Worker (Amazon Linux 2023 + t3.large)
- command line CLI creation:
  
```bash
# Important to change or update the AMI id for Amazon Linux
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \     
  --instance-type t3.large \
  --iam-instance-profile Name=AgoraFiscalEC2WorkerProfile \
  --security-group-ids sg-0abcd1234efgh5678 \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=agora-fiscal-ai},{Key=Name,Value=worker-01}]' \
  --user-data file://user-data.sh \
  --count 1
```
