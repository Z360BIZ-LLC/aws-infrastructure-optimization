---
description: Audit all AWS resources in the account
allowed-tools:
  - Bash
  - Read
  - Write
  - Task
---

# AWS Resource Audit

Generate a complete inventory of all AWS resources in the account.

## Workflow

1. **Compute Resources**
   - List all EC2 instances with details (type, state, tags)
   - List ECS clusters and services
   - List Lambda functions with runtime and memory
   - Check for any EKS clusters

2. **Database Resources**
   - List RDS instances and clusters
   - List DynamoDB tables
   - List ElastiCache clusters
   - Check for any Redshift clusters

3. **Storage Resources**
   - List S3 buckets with creation dates
   - List EBS volumes (attached and unattached)
   - List EFS file systems
   - Check for Glacier vaults

4. **Networking**
   - List VPCs and subnets
   - List load balancers (ALB, NLB, CLB)
   - List CloudFront distributions
   - List Route 53 hosted zones
   - List API Gateway APIs

5. **Security & IAM**
   - List IAM users and roles
   - List security groups
   - Check for AWS Config rules
   - List KMS keys

6. **Messaging & Integration**
   - List SQS queues
   - List SNS topics
   - List EventBridge rules

## Commands

```bash
# EC2
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,Tags[?Key==`Name`].Value|[0],LaunchTime]' --output table

# ECS
aws ecs list-clusters
aws ecs describe-clusters --clusters CLUSTER_ARN

# Lambda
aws lambda list-functions --query 'Functions[].[FunctionName,Runtime,MemorySize,LastModified]' --output table

# RDS
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceClass,Engine,DBInstanceStatus,AllocatedStorage]' --output table

# S3
aws s3 ls

# EBS
aws ec2 describe-volumes --query 'Volumes[].[VolumeId,Size,State,Attachments[0].InstanceId]' --output table

# Security Groups
aws ec2 describe-security-groups --query 'SecurityGroups[].[GroupId,GroupName,VpcId]' --output table
```

## Output

Save comprehensive audit to `reports/resource-audit-YYYY-MM-DD.md` with:
- Resource counts by service
- Detailed resource tables
- Tagging compliance summary
- Recommendations for cleanup
