# Phase 7: Creating Two Amazon ECS Services

## Lab Overview
In this phase, you will create two Amazon ECS services - one for the customer microservice and one for the employee microservice. These services will deploy your containerized applications to the ECS cluster and integrate with the Application Load Balancer target groups for proper traffic routing.

### Prerequisites
- Completed Phase 6 with target groups and Application Load Balancer  
- ECS cluster "microservices-serverlesscluster" created and active  
- Task definitions registered for both microservices  
- AWS CLI configured with appropriate permissions  

---

## Step 1: Environment Preparation

### 1.1 Set Environment Variables
```bash
# Set common variables
export AWS_REGION=us-east-1
export CLUSTER_NAME=microservices-serverlesscluster

# Get network configuration
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=LabVPC" --query 'Vpcs[0].VpcId' --output text --region $AWS_REGION)
export SUBNET1=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PublicSubnet1" --query 'Subnets[0].SubnetId' --output text --region $AWS_REGION)
export SUBNET2=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PublicSubnet2" --query 'Subnets[0].SubnetId' --output text --region $AWS_REGION)
export SECURITY_GROUP=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=microservices-sg" --query 'SecurityGroups[0].GroupId' --output text --region $AWS_REGION)

# Get target group ARNs
export CUSTOMER_TG_TWO_ARN=$(aws elbv2 describe-target-groups --names customer-tg-two --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
export EMPLOYEE_TG_TWO_ARN=$(aws elbv2 describe-target-groups --names employee-tg-two --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)

echo "VPC: $VPC_ID"
echo "Subnet 1: $SUBNET1"
echo "Subnet 2: $SUBNET2"
echo "Security Group: $SECURITY_GROUP"
echo "Customer TG Two: $CUSTOMER_TG_TWO_ARN"
echo "Employee TG Two: $EMPLOYEE_TG_TWO_ARN"
```

### 1.2 Verify Task Definitions
```bash
# Get latest task definition revisions
export CUSTOMER_TASK_REVISION=$(aws ecs describe-task-definition --task-definition customer-microservice --query 'taskDefinition.revision' --output text --region $AWS_REGION)
export EMPLOYEE_TASK_REVISION=$(aws ecs describe-task-definition --task-definition employee-microservice --query 'taskDefinition.revision' --output text --region $AWS_REGION)

echo "Customer Task Revision: $CUSTOMER_TASK_REVISION"
echo "Employee Task Revision: $EMPLOYEE_TASK_REVISION"

# Verify ECS cluster is active
aws ecs describe-clusters --clusters $CLUSTER_NAME --region $AWS_REGION --query 'clusters[0].status'
```

---

## Step 2: Create Customer Microservice ECS Service

### 2.1 Create Service Configuration File
**File:** `deployment/create-customer-microservice-tg-two.json`
```json
{
    "taskDefinition": "customer-microservice:REVISION-NUMBER",
    "cluster": "microservices-serverlesscluster",
    "loadBalancers": [
        {
            "targetGroupArn": "MICROSERVICE-TG-TWO-ARN",
            "containerName": "customer",
            "containerPort": 8080
        }
    ],
    "desiredCount": 1,
    "launchType": "FARGATE",
    "schedulingStrategy": "REPLICA",
    "deploymentController": {
        "type": "CODE_DEPLOY"
    },
    "networkConfiguration": {
        "awsvpcConfiguration": {
            "subnets": [
                "PUBLIC-SUBNET-1-ID",
                "PUBLIC-SUBNET-2-ID"
            ],
            "securityGroups": [
                "SECURITY-GROUP-ID"
            ],
            "assignPublicIp": "ENABLED"
        }
    }
}
```

### 2.2 Update Configuration with Actual Values
```bash
cd /path/to/your/deployment-repository

cat > create-customer-microservice-tg-two.json << EOF
{
    "taskDefinition": "customer-microservice:$CUSTOMER_TASK_REVISION",
    "cluster": "microservices-serverlesscluster",
    "loadBalancers": [
        {
            "targetGroupArn": "$CUSTOMER_TG_TWO_ARN",
            "containerName": "customer",
            "containerPort": 8080
        }
    ],
    "desiredCount": 1,
    "launchType": "FARGATE",
    "schedulingStrategy": "REPLICA",
    "deploymentController": {
        "type": "CODE_DEPLOY"
    },
    "networkConfiguration": {
        "awsvpcConfiguration": {
            "subnets": [
                "$SUBNET1",
                "$SUBNET2"
            ],
            "securityGroups": [
                "$SECURITY_GROUP"
            ],
            "assignPublicIp": "ENABLED"
        }
    }
}
EOF

cat create-customer-microservice-tg-two.json
```

### 2.3 Create Customer ECS Service
```bash
aws ecs create-service     --service-name customer-microservice     --cli-input-json file://create-customer-microservice-tg-two.json     --region $AWS_REGION

aws ecs describe-services     --cluster $CLUSTER_NAME     --services customer-microservice     --region $AWS_REGION     --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount}'
```

---

## Step 3: Create Employee Microservice ECS Service

### 3.1 Create Service Configuration File
**File:** `deployment/create-employee-microservice-tg-two.json`
```bash
cat > create-employee-microservice-tg-two.json << EOF
{
    "taskDefinition": "employee-microservice:$EMPLOYEE_TASK_REVISION",
    "cluster": "microservices-serverlesscluster",
    "loadBalancers": [
        {
            "targetGroupArn": "$EMPLOYEE_TG_TWO_ARN",
            "containerName": "employee",
            "containerPort": 8080
        }
    ],
    "desiredCount": 1,
    "launchType": "FARGATE",
    "schedulingStrategy": "REPLICA",
    "deploymentController": {
        "type": "CODE_DEPLOY"
    },
    "networkConfiguration": {
        "awsvpcConfiguration": {
            "subnets": [
                "$SUBNET1",
                "$SUBNET2"
            ],
            "securityGroups": [
                "$SECURITY_GROUP"
            ],
            "assignPublicIp": "ENABLED"
        }
    }
}
EOF
cat create-employee-microservice-tg-two.json
```
