# SportsDataBackup

## Overview

SportsDataBackup automates fetching sports highlights, storing data in Amazon S3 and DynamoDB, processing videos, and running on a schedule using AWS ECS Fargate and EventBridge. The project utilizes templated JSON files with environment variable injection for easy configuration and deployment.

## Features

- Automated retrieval of sports highlights using RapidAPI.
- Storage of highlight images and metadata in Amazon DynamoDB.
- Backup and processing of highlight videos in Amazon S3.
- Deployment of a Dockerized application to AWS ECS Fargate.
- Event-driven execution using AWS EventBridge.
- Logging and monitoring with Amazon CloudWatch

## Prerequisites

Before running the project, ensure you have the following installed and set up:

**1. Create a RapidAPI Account:** A RapidAPI account is required to fetch highlight images and videos. The Sports Highlights API will be used, with data from NCAA (USA College Basketball),
   which is available in the basic free plan.

**2. Verify Required Dependencies**

Ensure the following dependencies are installed:
- Docker (Pre-installed in most environments)
`docker --version`

- AWS CLI (Pre-installed in AWS CloudShell)
`aws --version`

- Python3
`python3 --version`

- gettext (for envsubst command-line utility)

    - Install on Linux/macOS:
    - `sudo apt install gettext`  # Ubuntu/Debian
    - `brew install gettext`  # macOS (Homebrew)

    - Install on Windows:
    - `choco install gettext`

**3. Retrieve AWS Account ID**
`aws sts get-caller-identity --query "Account" --output text`

Save your AWS Account ID for later use.

**4. Retrieve AWS Access Keys**

Check if you have access keys in the IAM Dashboard:
- Navigate to IAM > Users > Security Credentials
- Generate new keys if required
- Save them securely

## Architectural Diagram
  ![image](https://github.com/user-attachments/assets/e4a98547-6677-443d-b647-38a40829c417)


## Setup Instructions

**Step 1: Clone The Repository**

`git clone https://github.com/princemaxi/SportsDataBackup`
`cd SportsDataBackup/src`

**Step 2: Configure Environment Variables**

Create and configure the .env file by replacing the following values:
```
AWS_ACCOUNT_ID=your-account-id
AWS_ACCESS_KEY=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-access-key
AWS_REGION=us-east-1
S3_BUCKET_NAME=your-s3-bucket
RAPIDAPI_KEY=your-rapidapi-key
MEDIA_CONVERT_ENDPOINT=$(aws mediaconvert describe-endpoints --query "Endpoints[0].Url" --output text)
SUBNET_ID=subnet-xxx
SECURITY_GROUP_ID=sg-xxx
```

**Step 3: Load Environment Variables**
```
set -a
source .env
set +a
```
Verify the variables:
```
echo $AWS_LOGS_GROUP
echo $TASK_FAMILY
echo $AWS_ACCOUNT_ID
```

**Step 4: Generate Final JSON Files from Templates**
```
envsubst < taskdef.template.json > taskdef.json
envsubst < s3_dynamodb_policy.template.json > s3_dynamodb_policy.json
envsubst < ecsTarget.template.json > ecsTarget.json
envsubst < ecseventsrole-policy.template.json > ecseventsrole-policy.json
```
**Step 5: Build and Push Docker Image**

1. Create an ECR Repository

`aws ecr create-repository --repository-name sports-backup`

2. Log In to AWS ECR

`aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com`

3. Build the Docker Image

`docker build -t sports-backup .`

4. Tag and Push the Image
```
docker tag sports-backup:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
```

**Step 6: Create AWS Resources**

- Register the ECS Task Definition

`aws ecs register-task-definition --cli-input-json file://taskdef.json --region ${AWS_REGION}`

- Create the CloudWatch Logs Group

`aws logs create-log-group --log-group-name "${AWS_LOGS_GROUP}" --region ${AWS_REGION}`

- Attach S3/DynamoDB Policy to ECS Task Execution Role
```
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name S3DynamoDBAccessPolicy \
  --policy-document file://s3_dynamodb_policy.json
```

- Create the ECS Events Role

`aws iam create-role --role-name ecsEventsRole --assume-role-policy-document file://ecsEventsRole-trust.json`

- Attach the ECS Events Role Policy

`aws iam put-role-policy --role-name ecsEventsRole --policy-name ecsEventsPolicy --policy-document file://ecseventsrole-policy.json`

**Step 7: Schedule the Task using AWS EventBridge**

1. Create the EventBridge Rule

`aws events put-rule --name SportsBackupScheduleRule --schedule-expression "rate(1 day)" --region ${AWS_REGION}`

2. Add the Target

`aws events put-targets --rule SportsBackupScheduleRule --targets file://ecsTarget.json --region ${AWS_REGION}`

**Step 8: Manually Test the ECS Task**
```
aws ecs run-task \
  --cluster sports-backup-cluster \
  --launch-type Fargate \
  --task-definition ${TASK_FAMILY} \
  --network-configuration "awsvpcConfiguration={subnets=[\"${SUBNET_ID}\"],securityGroups=[\"${SECURITY_GROUP_ID}\"],assignPublicIp=\"ENABLED\"}" \
  --region ${AWS_REGION}
```

## What We Learned

- Using templated JSON files for deployment
- Automating data backup with Amazon DynamoDB
- Logging and monitoring using CloudWatch Logs
- Deploying an event-driven ECS task with AWS Fargate and EventBridge

## Future Enhancements

- Automated backup of DynamoDB tables to S3
- Batch processing of entire JSON files (supporting more than 10 videos per execution)

## Contributing

Contributions are welcome! Please submit a pull request or open an issue for discussion.

_License: This project is licensed under the MIT License._
