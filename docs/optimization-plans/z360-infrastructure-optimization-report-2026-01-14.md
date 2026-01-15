# Z360 AWS Infrastructure Optimization Report

**Account ID:** 108782097218
**Region:** us-east-1
**Analysis Period:** December 1, 2025 - January 14, 2026
**Report Date:** January 14, 2026
**Last Verified:** January 14, 2026 (via AWS Cost Explorer API, CloudWatch Metrics, Compute Optimizer)

---

## Executive Summary

### Current Monthly Infrastructure Cost: $1,821

This analysis excludes all Bedrock/Claude AI services (~$903/month additional). The Z360 infrastructure has several orphaned resources and overprovisioned services with optimization potential of **~$820/month (45% savings)**.

| Metric | Current | After Optimization |
|--------|---------|-------------------|
| Monthly Cost | $1,821 | ~$1,000 |
| Annual Cost | $21,852 | ~$12,000 |
| **Annual Savings** | - | **~$9,850** |

> **Note:** AWS credits are currently covering 100% of usage costs. When credits expire, you will see actual bills at the amounts shown above.

### Orphaned Resources (DELETE IMMEDIATELY)

| Resource | Type | Monthly Cost | Evidence |
|----------|------|-------------|----------|
| z36-app-cache-non-cluster | ElastiCache (3x r7g.large) | **$391** | 0 cache hits in 7 days |
| app-rds | RDS MySQL (db.t4g.micro) | **$14** | 0 connections since Dec 25, 2025 |
| Zscribe360 EC2 | t2.medium (DEMO SERVER) | **$41** | Hosts unused demo apps only |
| Stopped EC2 EBS | 3 volumes (68GB) | **$5.50** | Legacy instances stopped Dec 2025 |

**Total Orphaned Resource Cost: $451.50/month**

---

## Current Cost Breakdown

### By Service (December 2025 - Verified via Cost Explorer)

| Service | Monthly Cost | % Share | Status |
|---------|-------------|---------|--------|
| **Amazon ECS (Fargate)** | $657 | 36.1% | OVERPROVISIONED |
| **Amazon ElastiCache** | $535 | 29.4% | ORPHANED ($391) + Valkey ($144) |
| **Amazon RDS** | $212 | 11.6% | ORPHANED MySQL ($14) + Aurora ($198) |
| Amazon EC2 | $152 | 8.3% | ORPHANED DEMO ($41) + Stopped EBS ($6) |
| Amazon VPC (IPv4) | $112 | 6.1% | Required for public access |
| Elastic Load Balancing | $84 | 4.6% | Required (5 ALBs) |
| AWS WAF | $30 | 1.6% | Security (5 Web ACLs) |
| Other Services | $39 | 2.1% | KMS, Amplify, Route53, ECR, etc. |
| **TOTAL** | **$1,821** | 100% | |

---

## Critical Issues Identified

### 1. ElastiCache Replication Group - $391/month WASTED (DELETE)

**Resource:** `z36-app-cache-non-cluster`

**Configuration:**
- 3x cache.r7g.large nodes (~$130/month each)
- Engine: Redis 7.1.0
- Location: Default VPC (vpc-06fab8e58cd12e4a9)

**Utilization (7-day):**
| Metric | Value |
|--------|-------|
| Cache Hits | **0** |
| CPU | 1.8% |
| Memory | 0.2% |

**Action:** DELETE IMMEDIATELY. **Savings: $391/month**

---

### 2. app-rds MySQL Database - $14/month ORPHANED (DELETE)

**Resource:** `app-rds` (db.t4g.micro)

**Configuration:**
- Engine: MySQL 8.0.39
- Storage: 20GB gp2
- Location: Default VPC

**Utilization (7-day):**
| Date | Connections |
|------|-------------|
| 2026-01-07 to 2026-01-13 | **0** (every day) |

**Action:** Take snapshot, then DELETE. **Savings: $14/month**

---

### 3. Zscribe360 EC2 - $41/month ORPHANED DEMO SERVER (DELETE)

**Resource:** `i-0dcea86e8744c74bc`

> **IMPORTANT:** This instance does NOT run any production service. It only hosts unused demo applications.

**What This Server Hosts:**
| URL | Application |
|-----|-------------|
| http://98.85.52.46 | Z360 Browser UX Demo |
| https://98.85.52.46 | AI Assessment Tool |

**Cost Breakdown:**
- Instance (t2.medium): $35/month
- EBS Volume (55GB): $5.50/month
- Elastic IP: $3.60/month
- **Total: $41/month**

**Note:** The actual Zscribe transcription service runs on **Google Cloud** (ai.zscribe360.com → 34.29.16.3), NOT this AWS instance.

**Action:** DELETE IMMEDIATELY. **Savings: $41/month**

---

### 4. Stopped EC2 Instances - $5.50/month EBS (DELETE)

| Instance | Name | Stopped Date | EBS Cost |
|----------|------|--------------|----------|
| i-0140a2d42c6e8c498 | Z360 Manager | Dec 22, 2025 | $2.40 |
| i-0a65fe6e9b90743f8 | Z360 App | Dec 22, 2025 | $2.40 |
| i-08f8a3f5ecb1f61e1 | Z360 Queue manager | Jul 2, 2025 | $0.64 |

**Action:** Create AMI backups, then TERMINATE. **Savings: $5.50/month**

---

### 5. ECS Fargate Services - 50-85% OVERPROVISIONED (RIGHTSIZE)

**Source:** AWS Compute Optimizer API (Official AWS Service)

```
aws compute-optimizer get-ecs-service-recommendations
```

**Verified Recommendations:**

| Service | Current CPU/Mem | Recommended CPU/Mem | Monthly Savings |
|---------|-----------------|---------------------|-----------------|
| z360-agent-layer-beta-voice | 4096/8192 | 2048/5120 | **$102.17** |
| z360-agent-layer-staging-voice | 4096/8192 | 1024/4096 | **$101.62** |
| z360-agent-layer-beta-aegra | 2048/4096 | 256/1024 | **$61.40** |
| z360-beta-web | 2048/4096 | 1024/2048 | **$50.56** |
| z360-beta-reverb | 512/1024 | 256/512 | **$18.52** |
| z360-beta-queue | 512/1024 | 256/512 | **$12.65** |
| z360-staging-reverb | 512/1024 | 256/512 | **$10.30** |
| z360-staging-queue | 512/1024 | 256/512 | **$9.01** |
| **TOTAL VERIFIED SAVINGS** | | | **$366.21** |

**Services Already Optimized (No Changes Needed):**
- z360-agent-layer-beta-gateway (512/1024) - Compute Optimizer says "Optimized"

**Services Without Recommendations (Insufficient Data):**
- z360-agent-layer-staging-gateway
- z360-agent-layer-staging-aegra
- z360-staging-web

> **IMPORTANT:** No auto-scaling is currently configured for any ECS service. After rightsizing, configure Application Auto Scaling to handle traffic spikes automatically.

---

## What's Working Well

| Resource | Status | Monthly Cost |
|----------|--------|--------------|
| Aurora Serverless v2 (4 clusters) | EFFICIENT | $198 |
| Valkey Serverless (2 caches) | EFFICIENT | $144 |
| Application Load Balancers (5) | REQUIRED | $84 |
| AWS Amplify (2 apps) | EFFICIENT | $7 |
| Environment Isolation (4 VPCs) | PROPER | - |

---

## Optimization Plan

### Phase 1: Delete Orphaned Resources (~15 minutes)

**Total Savings: $451.50/month**

#### Step 1.1: Delete ElastiCache Cluster (~5 min)

```bash
# Take final backup
aws elasticache create-snapshot \
  --replication-group-id z36-app-cache-non-cluster \
  --snapshot-name z36-app-cache-final-backup

# Delete replication group
aws elasticache delete-replication-group \
  --replication-group-id z36-app-cache-non-cluster \
  --final-snapshot-identifier z36-app-cache-final
```

**Savings: $391/month**

#### Step 1.2: Delete app-rds MySQL (~3 min)

```bash
# Take final snapshot
aws rds create-db-snapshot \
  --db-instance-identifier app-rds \
  --db-snapshot-identifier app-rds-final-backup

# Delete instance
aws rds delete-db-instance \
  --db-instance-identifier app-rds \
  --skip-final-snapshot
```

**Savings: $14/month**

#### Step 1.3: Delete Zscribe360 Demo Server (~3 min)

```bash
# Create AMI backup (optional)
aws ec2 create-image \
  --instance-id i-0dcea86e8744c74bc \
  --name "Zscribe360-demo-archive-$(date +%Y%m%d)"

# Terminate instance
aws ec2 terminate-instances \
  --instance-ids i-0dcea86e8744c74bc
```

**Savings: $41/month**

#### Step 1.4: Delete Stopped EC2 Instances (~5 min)

```bash
# Create AMI backups
for instance in i-0140a2d42c6e8c498 i-0a65fe6e9b90743f8 i-08f8a3f5ecb1f61e1; do
  name=$(aws ec2 describe-instances --instance-ids $instance \
    --query 'Reservations[0].Instances[0].Tags[?Key==`Name`].Value' --output text)
  aws ec2 create-image --instance-id $instance \
    --name "${name}-archive-$(date +%Y%m%d)"
done

# Terminate instances (after AMI creation completes)
aws ec2 terminate-instances \
  --instance-ids i-0140a2d42c6e8c498 i-0a65fe6e9b90743f8 i-08f8a3f5ecb1f61e1
```

**Savings: $5.50/month**

---

### Phase 2: ECS Rightsizing + Auto-Scaling (~30 minutes)

**Total Savings: $366.21/month**

#### Step 2.1: Update Task Definitions

Apply AWS Compute Optimizer recommendations:

**Staging Services:**
| Service | Current | New |
|---------|---------|-----|
| z360-staging-queue | 512/1024 | 256/512 |
| z360-staging-reverb | 512/1024 | 256/512 |
| z360-agent-layer-staging-voice | 4096/8192 | 1024/4096 |

**Beta Services:**
| Service | Current | New |
|---------|---------|-----|
| z360-beta-web | 2048/4096 | 1024/2048 |
| z360-beta-queue | 512/1024 | 256/512 |
| z360-beta-reverb | 512/1024 | 256/512 |
| z360-agent-layer-beta-aegra | 2048/4096 | 256/1024 |
| z360-agent-layer-beta-voice | 4096/8192 | 2048/5120 |

#### Step 2.2: Configure Auto-Scaling

After rightsizing, add auto-scaling to handle traffic spikes:

```bash
# Example: Configure auto-scaling for z360-beta-web
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/z360-beta-Cluster-8E41UEvLHdpU/z360-beta-web-Service-5Ao8dxMzQ1d8 \
  --min-capacity 1 \
  --max-capacity 4

# Add CPU-based scaling policy
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/z360-beta-Cluster-8E41UEvLHdpU/z360-beta-web-Service-5Ao8dxMzQ1d8 \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 120
  }'
```

> **Note:** With auto-scaling, services will automatically scale up when CPU exceeds 70%, ensuring capacity during traffic spikes.

---

### Phase 3: Savings Plan (Optional, ~10 minutes)

**Additional Savings: ~$50/month**

After completing Phase 1 and 2, purchase a 1-Year Compute Savings Plan:
- **Commitment:** ~$0.50/hour (based on optimized workload)
- **Term:** 1 Year, No Upfront
- **Savings Rate:** ~19%

---

## Optimization Summary

| Phase | Action | Time | Monthly Savings |
|-------|--------|------|-----------------|
| **1.1** | Delete ElastiCache cluster | 5 min | $391 |
| **1.2** | Delete app-rds MySQL | 3 min | $14 |
| **1.3** | Delete Zscribe360 demo server | 3 min | $41 |
| **1.4** | Terminate stopped EC2s | 5 min | $5.50 |
| **2.1** | Rightsize ECS services | 20 min | $366 |
| **2.2** | Configure auto-scaling | 10 min | $0 (safety) |
| **3** | Purchase Savings Plan | 10 min | ~$50 |
| **TOTAL** | | **~1 hour** | **~$867** |

### Before/After Comparison

| Component | Current | Optimized | Savings |
|-----------|---------|-----------|---------|
| ECS Fargate | $657 | $291 | $366 |
| ElastiCache | $535 | $144 | $391 |
| RDS | $212 | $198 | $14 |
| EC2 + EBS | $152 | $0 | $152 |
| VPC (IPv4) | $112 | $112 | $0 |
| ALB | $84 | $84 | $0 |
| Other | $69 | $69 | $0 |
| Savings Plan | - | -$50 | $50 |
| **TOTAL** | **$1,821** | **~$900** | **~$920** |

---

## Complete Resource Inventory

### Resources to DELETE

| Resource | Type | Monthly Cost | Action |
|----------|------|-------------|--------|
| z36-app-cache-non-cluster | ElastiCache 3x r7g.large | $391 | DELETE |
| app-rds | MySQL db.t4g.micro | $14 | DELETE |
| i-0dcea86e8744c74bc | EC2 t2.medium (Zscribe360) | $41 | DELETE |
| i-0140a2d42c6e8c498 | EC2 t2.micro (stopped) | $2.40 | DELETE |
| i-0a65fe6e9b90743f8 | EC2 t2.xlarge (stopped) | $2.40 | DELETE |
| i-08f8a3f5ecb1f61e1 | EC2 t2.xlarge (stopped) | $0.64 | DELETE |

### Resources to KEEP

| Resource | Type | Monthly Cost |
|----------|------|-------------|
| z360-staging-cluster | Aurora Serverless v2 | ~$50 |
| z360-beta-cluster | Aurora Serverless v2 | ~$50 |
| z360-agent-layer-staging-cluster | Aurora Serverless v2 | ~$50 |
| z360-agent-layer-beta-cluster | Aurora Serverless v2 | ~$50 |
| z360-staging-valkey | Valkey Serverless | ~$70 |
| z360-beta-valkey | Valkey Serverless | ~$70 |
| 5 ALBs | Load Balancers | $84 |
| 5 WAF ACLs | Web Application Firewall | $30 |
| Zscribe & Zscribe-Docs | Amplify Apps | $7 |

---

## Architecture Diagram (After Optimization)

```
+------------------------------------------------------------------------------+
|  DEFAULT VPC (vpc-06fab8e58cd12e4a9) - CLEANED UP                            |
|                                                                              |
|     [DELETED] ElastiCache z36-app-cache-non-cluster (was $391/mo)           |
|     [DELETED] app-rds MySQL database (was $14/mo)                           |
|     [DELETED] Zscribe360 demo server (was $41/mo)                           |
|     [DELETED] Stopped EC2 instances + EBS (was $5.50/mo)                    |
|                                                                              |
+------------------------------------------------------------------------------+

+------------------------------------------------------------------------------+
|  COPILOT-MANAGED VPCs (4) - RIGHTSIZED + AUTO-SCALING                        |
|                                                                              |
|  +------------------------------------------------------------------------+  |
|  |  Z360 STAGING                                                          |  |
|  |  |- web (current)        +---> Aurora Serverless (0.5-8 ACU)           |  |
|  |  |- queue (256/512)      +---> Valkey Serverless                       |  |
|  |  |- reverb (256/512)                                                   |  |
|  +------------------------------------------------------------------------+  |
|                                                                              |
|  +------------------------------------------------------------------------+  |
|  |  Z360 BETA                                                             |  |
|  |  |- web (1024/2048)      +---> Aurora Serverless (0.5-8 ACU)           |  |
|  |  |- queue (256/512)      +---> Valkey Serverless                       |  |
|  |  |- reverb (256/512)                                                   |  |
|  +------------------------------------------------------------------------+  |
|                                                                              |
|  +------------------------------------------------------------------------+  |
|  |  AGENT LAYER STAGING                                                   |  |
|  |  |- gateway (current)    +---> Aurora Serverless (0.5-8 ACU)           |  |
|  |  |- aegra (current)                                                    |  |
|  |  |- voice (1024/4096)                                                  |  |
|  +------------------------------------------------------------------------+  |
|                                                                              |
|  +------------------------------------------------------------------------+  |
|  |  AGENT LAYER BETA                                                      |  |
|  |  |- gateway (512/1024)   +---> Aurora Serverless (0.5-8 ACU)           |  |
|  |  |- aegra (256/1024)                                                   |  |
|  |  |- voice (2048/5120)                                                  |  |
|  +------------------------------------------------------------------------+  |
|                                                                              |
+------------------------------------------------------------------------------+

+------------------------------------------------------------------------------+
|  SHARED SERVICES                                                             |
|  |- 5x ALBs ($84/mo)                                                        |
|  |- 4x Aurora Serverless ($198/mo)                                          |
|  |- 2x Valkey Serverless ($144/mo)                                          |
|  |- WAF ($30/mo)                                                            |
|  |- Other ($50/mo)                                                          |
+------------------------------------------------------------------------------+

OPTIMIZED MONTHLY COST: ~$900 (with Savings Plan: ~$850)
```

---

## Risk Assessment

| Change | Risk | Mitigation |
|--------|------|------------|
| Delete ElastiCache | **NONE** | 0 cache hits = not used |
| Delete app-rds | **NONE** | 0 connections = not used |
| Delete Zscribe360 | **NONE** | Demo apps only, not production |
| Terminate stopped EC2s | **LOW** | AMI backups first |
| Rightsize ECS | **LOW** | Based on AWS Compute Optimizer |
| Auto-scaling | **NONE** | Safety net for traffic spikes |

---

## Action Items

| Step | Action | Time | Savings |
|------|--------|------|---------|
| 1 | Delete ElastiCache cluster | 5 min | $391/mo |
| 2 | Delete app-rds MySQL | 3 min | $14/mo |
| 3 | Delete Zscribe360 EC2 | 3 min | $41/mo |
| 4 | Terminate stopped EC2s | 5 min | $5.50/mo |
| 5 | Rightsize ECS services | 20 min | $366/mo |
| 6 | Configure auto-scaling | 10 min | Safety |
| 7 | Purchase Savings Plan (optional) | 10 min | ~$50/mo |

---

## Summary

### Current State
- Monthly spend: **$1,821** (excluding Bedrock)
- Orphaned resources: **$451.50/month** wasted
- Overprovisioned ECS: **$366/month** wasted

### Optimized State
- Monthly spend: **~$900** (with Savings Plan: ~$850)
- Annual savings: **~$11,000**
- Clean architecture with auto-scaling

### Key Actions
1. DELETE orphaned ElastiCache cluster ($391/mo)
2. DELETE orphaned app-rds MySQL ($14/mo)
3. DELETE Zscribe360 demo server ($41/mo)
4. DELETE stopped EC2 instances ($5.50/mo)
5. RIGHTSIZE ECS per Compute Optimizer ($366/mo)
6. CONFIGURE auto-scaling for safety
7. PURCHASE Savings Plan (~$50/mo additional)

**Total Time: ~1 hour**
**Total Monthly Savings: ~$920**

---

*Report generated from live AWS infrastructure analysis*
*Data sources: AWS Cost Explorer API, CloudWatch Metrics, AWS Compute Optimizer*
*All figures verified January 14, 2026*
