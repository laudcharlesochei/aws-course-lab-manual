

# Phase 7: Creating Two Amazon ECS Services

## Lab Overview
In this phase, you will create two Amazon ECS services â€” one for the **customer** microservice and one for the **employee** microservice. These services will deploy your containerized applications to the ECS cluster and integrate with the Application Load Balancer target groups for proper traffic routing.

## Prerequisites
- Completed Phase 6 with target groups and Application Load Balancer
- ECS cluster `microservices-serverlesscluster` created and active
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
# Navigate to deployment directory
cd /path/to/your/deployment-repository

# Create updated configuration file with actual values
cat > create-customer-microservice-tg-two.json << 'EOF'
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

# Verify the file was created correctly
cat create-customer-microservice-tg-two.json
```

### 2.3 Create Customer ECS Service
```bash
# Create the customer microservice
aws ecs create-service   --service-name customer-microservice   --cli-input-json file://create-customer-microservice-tg-two.json   --region $AWS_REGION

# Verify service creation
aws ecs describe-services   --cluster $CLUSTER_NAME   --services customer-microservice   --region $AWS_REGION   --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount}'
```

---

## Step 3: Create Employee Microservice ECS Service

### 3.1 Create Service Configuration File
**File:** `deployment/create-employee-microservice-tg-two.json`
```bash
# Create employee service configuration
cat > create-employee-microservice-tg-two.json << 'EOF'
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

# Verify the file was created correctly
cat create-employee-microservice-tg-two.json
```

### 3.2 Create Employee ECS Service
```bash
# Create the employee microservice
aws ecs create-service   --service-name employee-microservice   --cli-input-json file://create-employee-microservice-tg-two.json   --region $AWS_REGION

# Verify service creation
aws ecs describe-services   --cluster $CLUSTER_NAME   --services employee-microservice   --region $AWS_REGION   --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount}'
```

---

## Step 4: Service Verification and Monitoring

### 4.1 Check Service Status
```bash
# Check both services status
echo "=== Customer Microservice Status ==="
aws ecs describe-services   --cluster $CLUSTER_NAME   --services customer-microservice   --region $AWS_REGION   --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}'

echo "=== Employee Microservice Status ==="
aws ecs describe-services   --cluster $CLUSTER_NAME   --services employee-microservice   --region $AWS_REGION   --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}'
```

### 4.2 Check Task Status
```bash
# List running tasks for both services
echo "=== Running Tasks ==="
aws ecs list-tasks   --cluster $CLUSTER_NAME   --region $AWS_REGION   --query 'taskArns[]'   --output table

# Describe tasks if any are running
TASK_ARNS=$(aws ecs list-tasks --cluster $CLUSTER_NAME --region $AWS_REGION --query 'taskArns' --output text)
if [ ! -z "$TASK_ARNS" ]; then
  aws ecs describe-tasks     --cluster $CLUSTER_NAME     --tasks $TASK_ARNS     --region $AWS_REGION     --query 'tasks[*].{TaskArn:taskArn,LastStatus:lastStatus,DesiredStatus:desiredStatus,Group:group}'
fi
```

### 4.3 Check Target Group Health
```bash
# Check target group health status
echo "=== Target Group Health Status ==="
for tg_name in customer-tg-two employee-tg-two; do
  echo "--- $tg_name ---"
  TG_ARN=$(aws elbv2 describe-target-groups --names $tg_name --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
  aws elbv2 describe-target-health     --target-group-arn $TG_ARN     --region $AWS_REGION     --query 'TargetHealthDescriptions[*].{Target:Target.Id,Health:TargetHealth.State,Reason:TargetHealth.Reason}'     --output table
done
```

---

## Step 5: Create Service Management Scripts

### 5.1 Create Service Status Check Script
**File:** `deployment/check-services.sh`
```bash
#!/bin/bash
set -e

echo "=== ECS Services Status Check ==="

# Set variables
AWS_REGION="us-east-1"
CLUSTER_NAME="microservices-serverlesscluster"
SERVICES=("customer-microservice" "employee-microservice")

for service in "${SERVICES[@]}"; do
  echo "--- $service ---"
  aws ecs describe-services     --cluster $CLUSTER_NAME     --services $service     --region $AWS_REGION     --query 'services[0].{ServiceName:serviceName,Status:status,DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}'     --output table
done

echo "=== Task Status ==="
TASK_ARNS=$(aws ecs list-tasks --cluster $CLUSTER_NAME --region $AWS_REGION --query 'taskArns' --output text)
if [ ! -z "$TASK_ARNS" ]; then
  aws ecs describe-tasks     --cluster $CLUSTER_NAME     --tasks $TASK_ARNS     --region $AWS_REGION     --query 'tasks[*].{TaskArn:taskArn,LastStatus:lastStatus,DesiredStatus:desiredStatus,LaunchType:launchType}'     --output table
else
  echo "No running tasks found"
fi
```

### 5.2 Create Service Update Script
**File:** `deployment/update-service.sh`
```bash
#!/bin/bash
set -e

SERVICE_NAME=$1
TASK_DEFINITION=$2

if [ -z "$SERVICE_NAME" ] || [ -z "$TASK_DEFINITION" ]; then
  echo "Usage: ./update-service.sh <service-name> <task-definition>"
  echo "Example: ./update-service.sh customer-microservice customer-microservice:2"
  exit 1
fi

echo "Updating service: $SERVICE_NAME with task definition: $TASK_DEFINITION"

aws ecs update-service   --cluster microservices-serverlesscluster   --service $SERVICE_NAME   --task-definition $TASK_DEFINITION   --region us-east-1

echo "Service update initiated for: $SERVICE_NAME"
```

### 5.3 Make Scripts Executable
```bash
chmod +x deployment/check-services.sh
chmod +x deployment/update-service.sh
```

---

## Step 6: Troubleshooting and Service Management

### 6.1 Common Service Issues and Solutions

**Issue: Service fails to create**
```bash
# Check if service already exists
aws ecs describe-services --cluster $CLUSTER_NAME --services customer-microservice --region $AWS_REGION

# If service exists, delete it first
aws ecs delete-service   --cluster $CLUSTER_NAME   --service customer-microservice   --force   --region $AWS_REGION

# Wait for service to be deleted
sleep 30
```

**Issue: Tasks stuck in PENDING state**
```bash
# Check events for error messages
aws ecs describe-services   --cluster $CLUSTER_NAME   --services customer-microservice   --region $AWS_REGION   --query 'services[0].events[0:5]'

# Check task execution role
aws iam get-role --role-name ecsTaskExecutionRole
```

**Issue: Tasks failing health checks**
```bash
# Check CloudWatch logs for container errors
aws logs describe-log-groups --log-group-name-prefix awslogs-capstone --region $AWS_REGION

# Check specific log streams
LOG_GROUP=$(aws logs describe-log-groups --log-group-name-prefix awslogs-capstone --query 'logGroups[0].logGroupName' --output text --region $AWS_REGION)
aws logs describe-log-streams --log-group-name $LOG_GROUP --region $AWS_REGION --query 'logStreams[0:3].logStreamName'
```

### 6.2 Service Scaling Configuration
```bash
# Configure service auto-scaling (optional for future use)
cat > customer-service-scaling.json << 'EOF'
{
  "serviceNamespace": "ecs",
  "resourceId": "service/$CLUSTER_NAME/customer-microservice",
  "scalableDimension": "ecs:service:DesiredCount",
  "minCapacity": 1,
  "maxCapacity": 4,
  "roleARN": "arn:aws:iam::$ACCOUNT_ID:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService"
}
EOF
```

---

## Step 7: Documentation and Repository Updates

### 7.1 Create Service Configuration Documentation
**File:** `deployment/ecs-services-config.md`
```markdown
# ECS Services Configuration

## Customer Microservice Service
- **Service Name**: customer-microservice
- **Task Definition**: customer-microservice:[revision]
- **Target Group**: customer-tg-two
- **Desired Count**: 1
- **Launch Type**: FARGATE
- **Deployment Controller**: CODE_DEPLOY

## Employee Microservice Service
- **Service Name**: employee-microservice
- **Task Definition**: employee-microservice:[revision]
- **Target Group**: employee-tg-two
- **Desired Count**: 1
- **Launch Type**: FARGATE
- **Deployment Controller**: CODE_DEPLOY

## Network Configuration
- **VPC**: LabVPC
- **Subnets**: PublicSubnet1, PublicSubnet2
- **Security Group**: microservices-sg
- **Public IP**: Enabled

## Management Commands

### Check Service Status
```bash
./check-services.sh
```
### Update Service
```bash
./update-service.sh customer-microservice customer-microservice:2
```
### Scale Service
```bash
aws ecs update-service   --cluster microservices-serverlesscluster   --service customer-microservice   --desired-count 2   --region us-east-1
```
```

### 7.2 Commit Configuration to Repository
```bash
# Add all service configuration files to git
git add create-customer-microservice-tg-two.json
git add create-employee-microservice-tg-two.json
git add check-services.sh
git add update-service.sh
git add ecs-services-config.md

# Commit the service configurations
git commit -m "feat: Add ECS services configuration

- Customer microservice service definition
- Employee microservice service definition  
- Service management and monitoring scripts
- Comprehensive documentation
- Integration with target groups for load balancing"

# Push to GitHub
git push origin dev
```

---

## Step 8: Final Verification

### 8.1 Comprehensive Service Health Check
```bash
# Run comprehensive health check
./deployment/check-services.sh

# Check load balancer DNS
export ALB_DNS=$(aws elbv2 describe-load-balancers --names microservicesLB --query 'LoadBalancers[0].DNSName' --output text --region $AWS_REGION)
echo "Load Balancer DNS: $ALB_DNS"

# Test basic connectivity (services may not be healthy yet)
echo "Testing load balancer endpoints (may fail until tasks are healthy)..."
curl -s http://$ALB_DNS/ > /dev/null && echo "Port 80: Accessible" || echo "Port 80: Not accessible yet"
curl -s http://$ALB_DNS:8080/ > /dev/null && echo "Port 8080: Accessible" || echo "Port 8080: Not accessible yet"
```

### 8.2 Monitor Service Deployment
```bash
# Monitor service deployment progress
echo "Monitoring service deployment..."
for i in {1..10}; do
  echo "Check $i:"
  ./deployment/check-services.sh
  echo "---"
  sleep 30
done
```

---

## Troubleshooting

### Common ECS Service Issues

**Issue: Service creation fails with `ResourceNotFoundException`**
```bash
# Verify cluster exists
aws ecs describe-clusters --clusters $CLUSTER_NAME --region $AWS_REGION

# Verify task definition exists
aws ecs describe-task-definition --task-definition customer-microservice --region $AWS_REGION
```

**Issue: Tasks fail to start**
```bash
# Check task stopped reasons
aws ecs describe-tasks   --cluster $CLUSTER_NAME   --tasks <task-arn>   --region $AWS_REGION   --query 'tasks[0].stoppedReason'

# Check container exit codes
aws ecs describe-tasks   --cluster $CLUSTER_NAME   --tasks <task-arn>   --region $AWS_REGION   --query 'tasks[0].containers[*].{Name:name,ExitCode:exitCode,Reason:reason}'
```

**Issue: Service stuck in DRAINING state**
```bash
# Force delete service if stuck
aws ecs delete-service   --cluster $CLUSTER_NAME   --service customer-microservice   --force   --region $AWS_REGION
```

---

## Conclusion

### Architecture Status
- **Services:** Two ECS services running in `microservices-serverlesscluster`  
- **Deployment:** CodeDeploy controller ready for blue/green deployments  
- **Networking:** Services deployed across multiple availability zones  
- **Load Balancing:** Integrated with target groups for traffic distribution  
- **Monitoring:** Comprehensive health checks and status monitoring  

### Service Configuration Summary
- **Customer Service:** Linked to `customer-tg-two` target group  
- **Employee Service:** Linked to `employee-tg-two` target group  
- **Scalability:** Ready for horizontal scaling configuration  
- **Deployment:** Prepared for CodeDeploy blue/green deployment strategy  

Your ECS services are now created and configured, providing the foundation for automated blue/green deployments through CodeDeploy and CodePipeline.
