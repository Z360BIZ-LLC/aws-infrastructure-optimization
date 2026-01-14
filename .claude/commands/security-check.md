---
description: Run security assessment on AWS infrastructure
allowed-tools:
  - Bash
  - Read
  - Write
  - Task
---

# AWS Security Assessment

Perform a comprehensive security check of the AWS infrastructure.

## Workflow

1. **IAM Security**
   - Check for root account usage
   - List users without MFA
   - Find overly permissive IAM policies
   - Check for unused access keys
   - Review IAM password policy

2. **Network Security**
   - Find security groups with 0.0.0.0/0 ingress
   - Check for public subnets with sensitive resources
   - Review NACLs for overly permissive rules
   - Check VPC flow logs status

3. **Storage Security**
   - Check S3 bucket public access settings
   - Verify S3 bucket encryption
   - Check EBS volume encryption
   - Review S3 bucket policies

4. **Database Security**
   - Check RDS public accessibility
   - Verify RDS encryption at rest
   - Review RDS security groups
   - Check for database backups

5. **Compute Security**
   - Check EC2 instances in public subnets
   - Review instance metadata service (IMDSv2)
   - Check for unencrypted AMIs
   - Review Lambda function permissions

6. **Logging & Monitoring**
   - Verify CloudTrail is enabled
   - Check CloudWatch log groups
   - Review GuardDuty status
   - Check for Config rules

## Commands

```bash
# IAM - Root account access keys
aws iam get-account-summary

# IAM - Users without MFA
aws iam list-users --query 'Users[].UserName' --output text | tr '\t' '\n' | while read user; do
  mfa=$(aws iam list-mfa-devices --user-name "$user" --query 'MFADevices' --output text)
  [ -z "$mfa" ] && echo "No MFA: $user"
done

# Security groups with open access
aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName,Description]' --output table

# Public S3 buckets
for bucket in $(aws s3 ls --query 'Buckets[].Name' --output text 2>/dev/null | tr '\t' ' '); do
  aws s3api get-public-access-block --bucket "$bucket" 2>/dev/null || echo "Check manually: $bucket"
done

# RDS public accessibility
aws rds describe-db-instances --query 'DBInstances[?PubliclyAccessible==`true`].[DBInstanceIdentifier]' --output table

# CloudTrail status
aws cloudtrail describe-trails --query 'trailList[].[Name,IsMultiRegionTrail,IsLogging]' --output table

# Unencrypted EBS volumes
aws ec2 describe-volumes --query 'Volumes[?Encrypted==`false`].[VolumeId,Size,State]' --output table
```

## Risk Levels

- **CRITICAL**: Immediate action required (public data, root keys)
- **HIGH**: Address within 24 hours (open security groups, no MFA)
- **MEDIUM**: Address within 1 week (missing encryption, logging gaps)
- **LOW**: Best practice improvements

## Output

Save security report to `reports/security-check-YYYY-MM-DD.md` with:
- Executive summary with risk score
- Critical findings (immediate action)
- Detailed findings by category
- Remediation steps for each finding
- Compliance checklist
