# 🚀 Dockerized Application Deployment on AWS ECS
A comprehensive deployment guide for containerized applications on AWS ECS Fargate with automated CI/CD pipeline, secure networking, and integrated AWS services.

## 📋 Table of Contents
- [Overview](Overview)
- [Architecture](Architecture)
- [Technologies](Technologies)
- [Deployment Phases](Deployment-Phases)
- [Setup Instructions](Setup-Instructions)
- [Configuration](Configuration)
- [Troubleshooting](Troubleshooting)
- [Monitoring](Monitoring)
- [GitHub Roles](GitHub-Roles)

## 📖 Overview
This project implements a comprehensive AWS ECS Fargate deployment for a Dockerized application. It features automated CI/CD with AWS CodeBuild, secure networking architecture with VPC endpoints, integrated database with DynamoDB, and scalable infrastructure with Application Load Balancer.

## 🏗️ Architecture
graph TB
    A[S3 Code Storage] --> B[CodeBuild]
    B --> C[ECR Registry]
    C --> D[ECS Fargate]
    D --> E[Application Load Balancer]
    E --> F[End Users]
    
    G[VPC] --> D
    H[DynamoDB] --> D
    I[VPC Endpoints] --> D
    J[CloudWatch] --> D
    
    subgraph "Private Network"
        D
        H
        I
    end
    
    subgraph "Public Network"
        E
        A
        B
    end

## 🛠️ Technologies Used
Service	Purpose	Version

AWS ECS Fargate	Container orchestration	-

AWS CodeBuild	CI/CD pipeline	-
Amazon ECR	Container registry	-
Amazon S3	Code storage	-
AWS VPC	Networking	-
Application Load Balancer	Traffic distribution	-
Amazon DynamoDB	NoSQL database	-
AWS IAM	Access management	-
Amazon CloudWatch	Monitoring & logging	-

## 📁 Project Structure
```text
dockerized-ecs-deployment/
├── docker/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── entrypoint.sh
├── src/
│   ├── app.py
│   ├── models/
│   └── utils/
├── infrastructure/
│   ├── ecs-task-definition.json
│   ├── buildspec.yml
│   └── iam-policies.json
├── scripts/
│   ├── deploy.sh
│   └── configure.sh
├── docs/
│   ├── architecture.md
│   └── setup-guide.md
├── tests/
│   ├── test_app.py
│   └── test_integration.py
├── .github/
│   └── workflows/
├── README.md
└── LICENSE
```

## 🚀 Deployment Phases
### Phase 1: Core Infrastructure Setup
1. Create ECR Repository - Container image storage
2. Setup VPC - 2 Public & 2 Private Subnets with NAT Gateways
3. Configure Security Groups - For ALB and ECS
4. Create VPC Endpoints - ECR API, ECR Docker, S3 Gateway
5. Setup DynamoDB Tables - answers and assignments

### Phase 2: CI/CD Pipeline
1. Upload Code to S3 - Dockerfile and dependencies
2. Configure CodeBuild - Automated build process
3. Build Docker Image - Automated image creation and push

### Phase 3: ECS Deployment
1. Create IAM Roles - ECS execution and task roles
2. Define ECS Task - Container specifications
3. Create ECS Cluster - Fargate cluster setup
4. Configure Load Balancer - ALB with target groups
5. Deploy ECS Service - Application deployment

### Phase 4: Optional Configuration
1. EC2 Instance Setup - For manual configuration
2. Application Configuration - Environment variables setup
3. Local Testing - Development environment setup

## 🔧 Setup Instructions
### 1. Create ECR Repository
```bash
aws ecr create-repository \
    --repository-name japan-kaizan \
    --region us-east-1 \
    --image-scanning-configuration scanOnPush=true
```

### 2. Create VPC and Subnets
```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)

# Create public subnets
PUB_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PUB_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)

# Create private subnets
PRIV_SUBNET_1=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PRIV_SUBNET_2=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.4.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
```

### 3. Create Security Groups
```bash
# Load Balancer Security Group
ALB_SG=$(aws ec2 create-security-group \
    --group-name ALB-Security-Group \
    --description "Security group for Application Load Balancer" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# ECS Security Group
ECS_SG=$(aws ec2 create-security-group \
    --group-name ECS-Security-Group \
    --description "Security group for ECS tasks" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)
```

### 4. Create VPC Endpoints
``` bash
# ECR API Endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name com.amazonaws.us-east-1.ecr.api \
    --vpc-endpoint-type Interface \
    --subnet-ids $PRIV_SUBNET_1 $PRIV_SUBNET_2 \
    --security-group-ids $ECS_SG \
    --private-dns-enabled
```

### 5. Create DynamoDB Tables
```bash
# Create answers table
aws dynamodb create-table \
    --table-name answers \
    --attribute-definitions \
        AttributeName=student_id,AttributeType=S \
        AttributeName=assignment_question_id,AttributeType=S \
    --key-schema \
        AttributeName=student_id,KeyType=HASH \
        AttributeName=assignment_question_id,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST \
    --global-secondary-indexes \
        "[{\"IndexName\": \"assignment_question_id-index\", \
          \"KeySchema\": [{\"AttributeName\": \"assignment_question_id\", \"KeyType\": \"HASH\"}], \
          \"Projection\": {\"ProjectionType\": \"ALL\"}}]"
```

### 6. Upload Code to S3
``` bash
# Create S3 bucket
aws s3 mb s3://your-bucket-name --region us-east-1

# Upload Dockerfile and dependencies
aws s3 cp Dockerfile.zip s3://your-bucket-name/code/Dockerfile.zip
aws s3 cp requirements.txt s3://your-bucket-name/code/requirements.txt
```

### 7. Create CodeBuild Project
``` bash
aws codebuild create-project \
    --name docker-build-project \
    --source type=S3,location=your-bucket-name/code/Dockerfile.zip \
    --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:5.0,computeType=BUILD_GENERAL1_MEDIUM \
    --service-role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/service-role/codebuild-service-role \
    --artifacts type=NO_ARTIFACTS \
    --logs-config cloudWatchLogs=status=ENABLED
```

### 8. Create ECS Task Definition
```json
{
  "family": "japan-kaizan-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecs-task-role",
  "containerDefinitions": [
    {
      "name": "app-container",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/japan-kaizan:latest",
      "portMappings": [
        {
          "containerPort": 8501,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/japan-kaizan",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 9. Create Application Load Balancer
``` bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name my-application-lb \
    --subnets $PUB_SUBNET_1 $PUB_SUBNET_2 \
    --security-groups $ALB_SG \
    --type application \
    --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
    --name ecs-target-group \
    --protocol HTTP \
    --port 80 \
    --target-type ip \
    --vpc-id $VPC_ID \
    --query 'TargetGroups[0].TargetGroupArn' --output text)

# Create listener
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 10. Create ECS Cluster and Service
```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name production-cluster

# Create ECS service
aws ecs create-service \
    --cluster production-cluster \
    --service-name japan-kaizan-service \
    --task-definition japan-kaizan-task:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$PRIV_SUBNET_1,$PRIV_SUBNET_2],securityGroups=[$ECS_SG],assignPublicIp=DISABLED}" \
    --load-balancers "targetGroupArn=$TG_ARN,containerName=app-container,containerPort=8501"
```

## ⚙️ Configuration
### Environment Variables

Variable	Description	Default Value
AWS_REGION	AWS region for deployment	us-east-1
ECR_REPOSITORY	ECR repository URI	Set during deployment
S3_BUCKET	S3 bucket for code storage	Set during deployment
DYNAMODB_TABLE_PREFIX	DynamoDB table prefix	production-
LOG_LEVEL	Application log level	INFO

### IAM Roles Configuration
### ECS Execution Role Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:*",
        "s3:*"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/*",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

### Buildspec.yml
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY
      - echo "Downloading source code from S3..."
      - aws s3 sync s3://$S3_BUCKET/code/ ./
  
  build:
    commands:
      - echo "Building Docker image..."
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - echo "Tagging image..."
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $ECR_REPOSITORY:$IMAGE_TAG
  
  post_build:
    commands:
      - echo "Pushing image to ECR..."
      - docker push $ECR_REPOSITORY:$IMAGE_TAG
      - echo "Writing image definitions file..."
      - printf '[{"name":"app-container","imageUri":"%s"}]' $ECR_REPOSITORY:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

## 📚 API Documentation
### Base URL
```text
http://your-alb-dns-name.us-east-1.elb.amazonaws.com
```

### Endpoints
POST /api/assignments - Create Assignment
### Request:

```json
{
  "teacher_id": "teacher123",
  "assignment_name": "Math Homework",
  "questions": [
    {
      "question": "Solve 2x + 5 = 15",
      "points": 10
    }
  ]
}
```

### Response:

```json
{
  "assignment_id": "assign_123456",
  "status": "created",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### GET /api/assignments/{assignment_id} - Get Assignment
### Response:

```json
{
  "assignment_id": "assign_123456",
  "teacher_id": "teacher123",
  "assignment_name": "Math Homework",
  "questions": [
    {
      "question_id": "q1",
      "question": "Solve 2x + 5 = 15",
      "points": 10
    }
  ],
  "created_at": "2024-01-15T10:30:00Z"
}
```

### POST /api/answers - Submit Answer
### Request:

```json
{
  "student_id": "student456",
  "assignment_question_id": "assign_123456_q1",
  "answer": "x = 5",
  "submission_time": "2024-01-15T11:00:00Z"
}
```

## 🚨 Troubleshooting
### Common Issues & Solutions
Issue	Symptoms	Solution
ECS Tasks Stuck in PENDING	Tasks not starting, no logs	Check VPC endpoints connectivity and IAM permissions
Load Balancer 503 Errors	Service unavailable errors	Verify target group health checks and security groups
CodeBuild Build Fails	Build timeout or permission errors	Check S3 permissions and ECR repository access
DynamoDB Connection Issues	Timeout or access denied	Verify VPC routing and IAM role permissions
Container Exit Code 137	Out of memory errors	Increase task memory allocation in ECS

### Debugging Commands
```bash
# Check ECS task status
aws ecs describe-tasks \
    --cluster production-cluster \
    --tasks $(aws ecs list-tasks --cluster production-cluster --query 'taskArns' --output text)

# View container logs
aws logs get-log-events \
    --log-group-name /ecs/japan-kaizan \
    --log-stream-name ecs/app-container/$(date +%Y/%m/%d) \
    --limit 50

# Check ALB target health
aws elbv2 describe-target-health \
    --target-group-arn $TG_ARN

# Monitor CodeBuild builds
aws codebuild batch-get-builds \
    --ids $(aws codebuild list-builds --query 'ids' --output text)
```

## 📊 Monitoring
### CloudWatch Metrics
ECS Metrics: CPUUtilization, MemoryUtilization
ALB Metrics: RequestCount, TargetResponseTime, HTTPCode_ELB_5XX_Count
DynamoDB Metrics: ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits
CodeBuild Metrics: BuildDuration, BuildsCount

### Setting Up Alarms
```bash
# High CPU alarm for ECS
aws cloudwatch put-metric-alarm \
    --alarm-name ECS-HighCPU \
    --metric-name CPUUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=ClusterName,Value=production-cluster

# 5XX errors alarm for ALB
aws cloudwatch put-metric-alarm \
    --alarm-name ALB-5XX-Errors \
    --metric-name HTTPCode_ELB_5XX_Count \
    --namespace AWS/ApplicationELB \
    --statistic Sum \
    --period 60 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1
```

## 🔮 Future Enhancements
### Short-term Improvements
Implement Blue/Green Deployments using AWS CodeDeploy

Add Auto Scaling Policies based on custom metrics

Implement CI/CD Pipeline with GitHub Actions integration

Add Health Check Endpoints for better monitoring

### Medium-term Enhancements
Integrate AWS X-Ray for distributed tracing

Implement Canary Deployments for safe feature releases

Add CloudFront CDN for global content delivery

Implement AWS WAF for enhanced security

### Long-term Vision
Multi-region Deployment for disaster recovery

Cost Optimization with Spot instances and Savings Plans

Advanced Monitoring with Datadog or New Relic integration

Infrastructure as Code with Terraform or CDK


