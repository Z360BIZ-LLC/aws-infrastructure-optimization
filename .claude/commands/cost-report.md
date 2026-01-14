---
description: Generate AWS cost analysis report
allowed-tools:
  - Bash
  - Read
  - Write
  - Task
---

# AWS Cost Analysis Report

Generate a comprehensive cost analysis report for the AWS infrastructure.

## Workflow

1. **Gather Cost Data**
   - Query Cost Explorer for monthly spending (if available)
   - Get EC2 instance costs and utilization
   - Check RDS instance costs
   - Analyze S3 storage costs
   - Review Lambda invocation costs
   - Check ElastiCache costs

2. **Identify Optimization Opportunities**
   - Stopped EC2 instances (still incurring EBS costs)
   - Underutilized instances (CPU < 20% average)
   - Oversized database instances
   - Unused S3 storage
   - Lambda functions with high memory allocation but low usage

3. **Generate Recommendations**
   - Right-sizing opportunities
   - Reserved Instance or Savings Plan candidates
   - Resources to terminate
   - Storage class optimizations

4. **Create Report**
   - Save report to `reports/cost-report-YYYY-MM-DD.md`
   - Include summary, findings, and actionable recommendations
   - Estimate potential monthly savings

## Commands to Use

```bash
# Cost by service (requires Cost Explorer)
aws ce get-cost-and-usage --time-period Start=$(date -d "30 days ago" +%Y-%m-%d),End=$(date +%Y-%m-%d) --granularity DAILY --metrics BlendedCost --group-by Type=DIMENSION,Key=SERVICE

# EC2 utilization
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=INSTANCE_ID --start-time $(date -d "7 days ago" -Iseconds) --end-time $(date -Iseconds) --period 86400 --statistics Average

# S3 bucket sizes
aws s3 ls --summarize --human-readable --recursive s3://BUCKET_NAME

# RDS instance metrics
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=DB_ID --start-time $(date -d "7 days ago" -Iseconds) --end-time $(date -Iseconds) --period 86400 --statistics Average
```

## Output Format

Create a markdown report with:
- Executive Summary
- Current Spending Overview
- Resource-by-Resource Analysis
- Optimization Recommendations (with estimated savings)
- Action Items (prioritized)
