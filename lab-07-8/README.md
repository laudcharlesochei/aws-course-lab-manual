# Phase 6: Creating Target Groups and an Application Load Balancer

## Lab Overview
In this phase, you will create an Application Load Balancer with target groups to route traffic between your customer and employee microservices.  
The load balancer will provide a single entry point for your application and implement path-based routing to direct requests to the appropriate microservices.

### Prerequisites
- Completed Phase 5 with ECR repositories and ECS cluster  
- AWS CLI configured with appropriate permissions  
- Understanding of VPC and subnet configuration  
- Basic knowledge of load balancing concepts  

---

## Step 1: Environment Preparation

### 1.1 Set Environment Variables
```bash
export AWS_REGION=us-east-1
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=LabVPC" --query 'Vpcs[0].VpcId' --output text)
export SUBNET1=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PublicSubnet1" --query 'Subnets[0].SubnetId' --output text)
export SUBNET2=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=PublicSubnet2" --query 'Subnets[0].SubnetId' --output text)

echo "VPC ID: $VPC_ID"
echo "Subnet 1: $SUBNET1"
echo "Subnet 2: $SUBNET2"
```

### 1.2 Verify Network Configuration
```bash
aws ec2 describe-vpcs --vpc-ids $VPC_ID --region $AWS_REGION
aws ec2 describe-subnets --subnet-ids $SUBNET1 $SUBNET2 --region $AWS_REGION
```

---

## Step 2: Create Target Groups

### 2.1 Create Customer Target Groups
```bash
aws elbv2 create-target-group   --name customer-tg-one   --protocol HTTP   --port 8080   --target-type ip   --vpc-id $VPC_ID   --health-check-path /   --region $AWS_REGION

aws elbv2 create-target-group   --name customer-tg-two   --protocol HTTP   --port 8080   --target-type ip   --vpc-id $VPC_ID   --health-check-path /   --region $AWS_REGION
```

### 2.2 Create Employee Target Groups
```bash
aws elbv2 create-target-group   --name employee-tg-one   --protocol HTTP   --port 8080   --target-type ip   --vpc-id $VPC_ID   --health-check-path /admin/suppliers   --region $AWS_REGION

aws elbv2 create-target-group   --name employee-tg-two   --protocol HTTP   --port 8080   --target-type ip   --vpc-id $VPC_ID   --health-check-path /admin/suppliers   --region $AWS_REGION
```

### 2.3 Store Target Group ARNs
```bash
export CUSTOMER_TG_ONE_ARN=$(aws elbv2 describe-target-groups --names customer-tg-one --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
export CUSTOMER_TG_TWO_ARN=$(aws elbv2 describe-target-groups --names customer-tg-two --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
export EMPLOYEE_TG_ONE_ARN=$(aws elbv2 describe-target-groups --names employee-tg-one --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
export EMPLOYEE_TG_TWO_ARN=$(aws elbv2 describe-target-groups --names employee-tg-two --query 'TargetGroups[0].TargetGroupArn' --output text --region $AWS_REGION)
```

---

## Step 3: Create Security Group for Load Balancer

### 3.1 Create Security Group
```bash
aws ec2 create-security-group   --group-name microservices-sg   --description "Security group for microservices load balancer"   --vpc-id $VPC_ID   --region $AWS_REGION

export SG_ID=$(aws ec2 describe-security-groups --group-names microservices-sg --query 'SecurityGroups[0].GroupId' --output text --region $AWS_REGION)
```

### 3.2 Configure Inbound Rules
```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $AWS_REGION
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 8080 --cidr 0.0.0.0/0 --region $AWS_REGION
```

---

## Step 4: Create Application Load Balancer

### 4.1 Create Load Balancer
```bash
aws elbv2 create-load-balancer   --name microservicesLB   --subnets $SUBNET1 $SUBNET2   --security-groups $SG_ID   --scheme internet-facing   --type application   --region $AWS_REGION

export ALB_ARN=$(aws elbv2 describe-load-balancers --names microservicesLB --query 'LoadBalancers[0].LoadBalancerArn' --output text --region $AWS_REGION)
export ALB_DNS=$(aws elbv2 describe-load-balancers --names microservicesLB --query 'LoadBalancers[0].DNSName' --output text --region $AWS_REGION)
```

---

## Step 5: Configure Listeners and Routing Rules

### 5.1 Create HTTP:80 Listener
```bash
aws elbv2 create-listener   --load-balancer-arn $ALB_ARN   --protocol HTTP   --port 80   --default-actions Type=forward,TargetGroupArn=$CUSTOMER_TG_TWO_ARN   --region $AWS_REGION

export LISTENER_80_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[?Port==`80`].ListenerArn' --output text --region $AWS_REGION)
```

### 5.2 Add Path-Based Rule to HTTP:80
```bash
aws elbv2 create-rule   --listener-arn $LISTENER_80_ARN   --priority 10   --conditions Field=path-pattern,Values='/admin/*'   --actions Type=forward,TargetGroupArn=$EMPLOYEE_TG_TWO_ARN   --region $AWS_REGION
```

### 5.3 Create HTTP:8080 Listener
```bash
aws elbv2 create-listener   --load-balancer-arn $ALB_ARN   --protocol HTTP   --port 8080   --default-actions Type=forward,TargetGroupArn=$CUSTOMER_TG_ONE_ARN   --region $AWS_REGION

export LISTENER_8080_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[?Port==`8080`].ListenerArn' --output text --region $AWS_REGION)
```

### 5.4 Add Path-Based Rule to HTTP:8080
```bash
aws elbv2 create-rule   --listener-arn $LISTENER_8080_ARN   --priority 10   --conditions Field=path-pattern,Values='/admin/*'   --actions Type=forward,TargetGroupArn=$EMPLOYEE_TG_ONE_ARN   --region $AWS_REGION
```

---

## Step 6: Verification and Testing

### 6.1 Verify Load Balancer Configuration
```bash
aws elbv2 describe-rules --listener-arn $LISTENER_80_ARN --region $AWS_REGION
aws elbv2 describe-rules --listener-arn $LISTENER_8080_ARN --region $AWS_REGION
```

### 6.2 Create Documentation
**File:** deployment/load-balancer-config.md
```markdown
# Load Balancer Configuration

## Application Load Balancer
- **Name:** microservicesLB  
- **DNS Name:** $ALB_DNS  
- **Type:** Application Load Balancer  
- **Scheme:** internet-facing  

## Target Groups
### Customer
- customer-tg-one â†’ Port 8080  
- customer-tg-two â†’ Port 8080  

### Employee
- employee-tg-one â†’ Port 8080 (/admin/suppliers)  
- employee-tg-two â†’ Port 8080 (/admin/suppliers)
```

---

## Step 7: Create Validation Script

### 7.1 Test Script
**File:** deployment/test-load-balancer.sh
```bash
#!/bin/bash
set -e

echo "Testing Load Balancer..."
nslookup $ALB_DNS
curl -I http://$ALB_DNS/
curl -I http://$ALB_DNS:8080/
```

---

## Troubleshooting

| Issue | Resolution |
|--------|-------------|
| Target groups not created | Verify VPC ID and re-run command |
| Load balancer fails | Check subnet and SG configuration |
| Health checks failing | Update health check path and interval |

---

## Conclusion
âœ… Created four target groups for blue/green deployment  
âœ… Configured an Application Load Balancer with two listeners  
âœ… Added path-based routing rules  
âœ… Implemented security group and validated configuration  

### Next Steps
- Create ECS services for both microservices  
- Register services with target groups  
- Configure CodeDeploy for blue/green deployments  
- Test end-to-end functionality  
