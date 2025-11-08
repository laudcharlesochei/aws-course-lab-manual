# Phase 5: Creating ECR Repositories, ECS Cluster, Task Definitions, and AppSpec Files

## Lab Overview
In this phase, you will prepare your microservices for production deployment by creating Amazon ECR repositories for Docker images, setting up an ECS cluster, defining task configurations, and creating deployment specifications.  
This lab uses **GitHub Codespaces** or your local environment, with **GitHub** for version control.

### Prerequisites
- Completed Phase 4 with working Docker containers  
- AWS CLI configured with proper permissions  
- GitHub repository containing microservices code  
- Docker Desktop (for local testing)

---

## Step 1: Configure AWS Environment

### 1.1 Verify AWS CLI Configuration
```bash
aws configure list
aws sts get-caller-identity
```

### 1.2 Set Environment Variables
```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_BASE_URI=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

echo "Account ID: $ACCOUNT_ID"
echo "ECR Base URI: $ECR_BASE_URI"
```

---

## Step 2: Create ECR Repositories and Upload Docker Images

### 2.1 Authenticate Docker with ECR
```bash
aws ecr get-login-password --region $AWS_REGION |   docker login --username AWS --password-stdin $ECR_BASE_URI
```

### 2.2 Create ECR Repositories
```bash
aws ecr create-repository --repository-name customer --region $AWS_REGION
aws ecr create-repository --repository-name employee --region $AWS_REGION
aws ecr describe-repositories --region $AWS_REGION
```

### 2.3 Set ECR Repository Permissions
```bash
cat > customer-repo-policy.json << EOF
{
  "Version": "2008-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "ecr:*"
  }]
}
EOF

aws ecr set-repository-policy   --repository-name customer   --policy-text file://customer-repo-policy.json   --region $AWS_REGION

aws ecr set-repository-policy   --repository-name employee   --policy-text file://customer-repo-policy.json   --region $AWS_REGION

rm customer-repo-policy.json
```

### 2.4 Tag and Push Docker Images
```bash
cd /path/to/microservices-project
docker tag customer:latest $ECR_BASE_URI/customer:latest
docker tag employee:latest $ECR_BASE_URI/employee:latest
docker push $ECR_BASE_URI/customer:latest
docker push $ECR_BASE_URI/employee:latest
```

---

## Step 3: Create ECS Cluster

### 3.1 Create via AWS Console
- Open **Amazon ECS** in AWS Management Console  
- Click **Create Cluster** â†’ **Networking only (Fargate)**  
- Configure:
  - Cluster name: `microservices-serverlesscluster`
  - Use existing VPC (LabVPC)
  - Select `PublicSubnet1` and `PublicSubnet2`
  - Enable CloudWatch Container Insights (optional)
- Click **Create** and wait for status = **ACTIVE**

### 3.2 Create via AWS CLI
```bash
aws ecs create-cluster   --cluster-name microservices-serverlesscluster   --capacity-providers FARGATE FARGATE_SPOT   --default-capacity-provider-strategy   capacityProvider=FARGATE,weight=1   capacityProvider=FARGATE_SPOT,weight=1   --region $AWS_REGION
```

---

## Step 4: Set Up Deployment Repository

### 4.1 Create Local Deployment Directory
```bash
mkdir -p deployment && cd deployment
git init
git branch -M dev
echo "# Deployment Configuration Files" > README.md
git add README.md
git commit -m "Initial deployment repository setup"
```

### 4.2 Create GitHub Repository
- Create a new private repository: `coffee-suppliers-deployment`
- Description: *Deployment configuration for coffee suppliers microservices*
```bash
git remote add origin https://github.com/YOUR_USERNAME/coffee-suppliers-deployment.git
git push -u origin dev
```

---

## Step 5: Create Task Definitions

### 5.1 Get Required Information
```bash
export RDS_ENDPOINT=$(aws rds describe-db-instances   --query 'DBInstances[0].Endpoint.Address'   --output text   --region $AWS_REGION)

echo "RDS Endpoint: $RDS_ENDPOINT"
```

### 5.2 Customer Task Definition
**File:** deployment/taskdef-customer.json
```json
{
  "containerDefinitions": [{
    "name": "customer",
    "image": "customer",
    "environment": [
      {"name": "APP_DB_HOST", "value": "<RDS-ENDPOINT>"},
      {"name": "APP_DB_USER", "value": "admin"},
      {"name": "APP_DB_PASSWORD", "value": "lab-password"},
      {"name": "APP_DB_NAME", "value": "COFFEE"}
    ],
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "essential": true
  }],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT-ID>:role/ecsTaskExecutionRole",
  "family": "customer-microservice"
}
```

### 5.3 Employee Task Definition
**File:** deployment/taskdef-employee.json
```json
{
  "containerDefinitions": [{
    "name": "employee",
    "image": "employee",
    "environment": [
      {"name": "APP_DB_HOST", "value": "<RDS-ENDPOINT>"},
      {"name": "APP_DB_USER", "value": "admin"},
      {"name": "APP_DB_PASSWORD", "value": "lab-password"},
      {"name": "APP_DB_NAME", "value": "COFFEE"}
    ],
    "portMappings": [{"containerPort": 8081, "protocol": "tcp"}],
    "essential": true
  }],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<ACCOUNT-ID>:role/ecsTaskExecutionRole",
  "family": "employee-microservice"
}
```

---

## Step 6: Create AppSpec Files

### Customer AppSpec File
**File:** deployment/appspec-customer.yaml
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "customer"
          ContainerPort: 8080
```

### Employee AppSpec File
**File:** deployment/appspec-employee.yaml
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "employee"
          ContainerPort: 8081
```

---

## Step 7: CI/CD Preparation

### 7.1 Update Task Definitions for Pipeline
```bash
cp taskdef-customer.json taskdef-customer-pipeline.json
sed -i 's|"image": "customer"|"image": "<IMAGE1_NAME>"|' taskdef-customer-pipeline.json

cp taskdef-employee.json taskdef-employee-pipeline.json
sed -i 's|"image": "employee"|"image": "<IMAGE1_NAME>"|' taskdef-employee-pipeline.json
```

### 7.2 Create Deployment Script
**File:** deployment/deploy.sh
```bash
#!/bin/bash
set -e
echo "Starting deployment process..."

sed -i "s/<IMAGE1_NAME>/$IMAGE1_NAME/g" taskdef-*.json
sed -i "s/<RDS-ENDPOINT>/$RDS_ENDPOINT/g" taskdef-*.json
sed -i "s/<ACCOUNT-ID>/$ACCOUNT_ID/g" taskdef-*.json

echo "Deployment configurations updated successfully."
```

### 7.3 Commit and Push Deployment Files
```bash
git add .
git status
git commit -m "feat: Add ECS task definitions and AppSpec files"
git push origin dev
```

---

## Step 8: Verification and Validation

### 8.1 Verify ECR and ECS
```bash
aws ecr describe-repositories --region $AWS_REGION
aws ecr describe-images --repository-name customer --region $AWS_REGION
aws ecs describe-clusters --clusters microservices-serverlesscluster --region $AWS_REGION
aws ecs list-task-definitions --status ACTIVE --region $AWS_REGION
```

---

## Conclusion
âœ… Created and configured ECR repositories  
âœ… Pushed Docker images to ECR  
âœ… Built ECS Fargate cluster  
âœ… Defined and registered ECS task definitions  
âœ… Created AppSpec files for CodeDeploy  
âœ… Prepared CI/CD configurations in GitHub

### Next Steps
- Create ECS services for both microservices  
- Configure Application Load Balancer  
- Deploy services to ECS cluster  
- Validate end-to-end system functionality
