# Z360 AWS Infrastructure Optimization Report

**Account ID:** 108782097218
**Region:** us-east-1
**Analysis Period:** December 1, 2025 - January 14, 2026
**Report Date:** January 14, 2026

---

## Executive Summary

### Current Monthly Infrastructure Cost: $1,712

This analysis excludes all Bedrock/Claude AI services as requested. The Z360 infrastructure is significantly over-provisioned in several key areas, with optimization potential of **$800-1,000/month (47-58% savings)**.

| Metric | Current | After Optimization |
|--------|---------|-------------------|
| Monthly Cost | $1,712 | $750-900 |
| Annual Cost | $20,544 | $9,000-10,800 |
| **Annual Savings** | - | **$9,700-11,500** |

---

## Current Cost Breakdown

### By Service (45-Day Period, Prorated to Monthly)

| Service | Monthly Cost | % Share | Status |
|---------|-------------|---------|--------|
| **Amazon ECS (Fargate)** | $634 | 37.0% | 🔴 OVERPROVISIONED |
| **Amazon ElastiCache** | $509 | 29.7% | 🔴 ORPHANED CLUSTER |
| Amazon RDS (Aurora) | $201 | 11.7% | ✅ EFFICIENT |
| Amazon VPC (IPv4) | $105 | 6.1% | ⚠️ CAN OPTIMIZE |
| Amazon EC2 | $101 | 5.9% | 🔴 UNDERUTILIZED |
| Elastic Load Balancing | $80 | 4.7% | ✅ REQUIRED |
| AWS WAF | $30 | 1.7% | ✅ SECURITY |
| Route 53 | $9 | 0.5% | ✅ DNS |
| Other (KMS, ECR, Secrets, etc.) | $43 | 2.7% | ✅ MINIMAL |
| **TOTAL** | **$1,712** | 100% | |

### Detailed Usage Breakdown

| Usage Type | Monthly Cost | Description |
|------------|-------------|-------------|
| Fargate vCPU Hours | $520 | 12 ECS services across 4 clusters |
| ElastiCache cache.r7g.large | $372 | 3-node replication group (UNUSED) |
| Aurora Serverless v2 | $180 | 4 database clusters |
| Valkey Serverless Data | $119 | Staging + Beta caches |
| Fargate Memory | $114 | Memory allocation for tasks |
| Public IPv4 Addresses | $103 | 30+ public IPs across services |
| ALB Usage | $80 | 5 load balancers |
| EC2 t2.xlarge | $64 | 2 stopped instances (EBS only) |
| EC2 t2.medium | $33 | Zscribe360 transcription service |
| WAF WebACL | $22 | Web application firewall |
| Valkey ECPUs | $18 | Serverless compute units |
| RDS db.t4g.micro | $11 | MySQL instance (app-rds) |
| EBS Volumes | $9 | gp3 storage for EC2 |
| Route 53 Hosted Zones | $9 | 13 DNS zones |

---

## Critical Issues Identified

### 1. ElastiCache Replication Group - $372/month WASTED 🔴

**Current Configuration:**
- 3x cache.r7g.large nodes ($124/month each)
- Replication group: `z36-app-cache-non-cluster`
- VPC: Default VPC (vpc-06fab8e58cd12e4a9)

**Utilization Metrics (7-day average):**
| Metric | Value | Assessment |
|--------|-------|------------|
| CPU Utilization | 1.8% | MASSIVELY UNDERUTILIZED |
| Memory Usage | **0.2%** | ALMOST EMPTY |
| Current Connections | 5-6 | Internal connections only |
| **Cache Hits (7 days)** | **0** | **NO APPLICATION USING IT** |

**Investigation Finding:**
This ElastiCache cluster was created for the **legacy EC2-based Z360 application** which has since been migrated to ECS Fargate. The ECS environments (staging/beta) use **separate Valkey Serverless caches**. This cluster is completely orphaned.

**Evidence:**
- Named "z36-app-cache" - matches legacy Z360 naming
- Located in Default VPC with stopped legacy EC2 instances
- Zero cache hits proves no application is reading from it
- Staging/Beta environments have their own `z360-staging-valkey` and `z360-beta-valkey` caches

**Recommendation:** Delete this cluster immediately. **Savings: $372/month**

---

### 2. ECS Fargate Services - 50-85% OVERPROVISIONED 🔴

AWS Compute Optimizer identified significant overprovisioning across all ECS services:

| Service | Current CPU | Current Mem | Recommended CPU | Recommended Mem | Savings |
|---------|------------|-------------|-----------------|-----------------|---------|
| agent-staging-aegra | 2048 | 4096 | 256 | 1024 | **85%** |
| agent-staging-voice | 4096 | 8192 | 1024 | 4096 | **70%** |
| agent-beta-aegra | 2048 | 4096 | 256 | 1024 | **85%** |
| agent-beta-voice | 4096 | 8192 | 2048 | 5120 | **48%** |
| z360-beta-web | 2048 | 4096 | 1024 | 2048 | **50%** |
| z360-staging-queue | 512 | 1024 | 256 | 512 | **50%** |
| z360-staging-reverb | 512 | 1024 | 256 | 512 | **50%** |
| z360-beta-queue | 512 | 1024 | 256 | 512 | **50%** |
| z360-beta-reverb | 512 | 1024 | 256 | 512 | **50%** |

**Current ECS Monthly Cost:** $634/month
**Optimized ECS Monthly Cost:** $264/month
**Savings: $370/month (58%)**

---

### 3. EC2 Instances - UNDERUTILIZED + ORPHANED 🔴

**Running Instance:**

| Instance | Name | Type | Monthly Cost | CPU Utilization | Recommendation |
|----------|------|------|-------------|-----------------|----------------|
| i-0dcea86e8744c74bc | Zscribe360 | t2.medium | $33 | **4.8% avg** | Downsize to t3.small |

The Zscribe360 instance is using only 4.8% CPU on average (27.9% max). This is a transcription service that could run on a much smaller instance.

**Recommendation:** Change to t3.small ($17/month). **Savings: $16/month**

**Stopped Instances (Still Incurring EBS Costs):**

| Instance | Name | Type | Purpose | EBS Size | Monthly EBS Cost |
|----------|------|------|---------|----------|------------------|
| i-0140a2d42c6e8c498 | Z360 Manager | t2.micro | Legacy management console | 30GB | $2.40 |
| i-0a65fe6e9b90743f8 | Z360 App | t2.xlarge | Legacy app server | 30GB | $2.40 |
| i-08f8a3f5ecb1f61e1 | Z360 Queue manager | t2.xlarge | Legacy queue worker | 8GB | $0.64 |

**Context:** These represent the legacy EC2-based Z360 deployment that has been migrated to ECS Fargate (z360-staging/beta clusters). They have been stopped since early 2025.

**Recommendation:** Create AMI snapshots for archival, then terminate instances and delete EBS volumes. **Savings: $5.50/month**

---

### 4. Public IPv4 Addresses - $103/month ⚠️

AWS charges $0.005/hour for public IPv4 addresses since February 2024.

**Current Usage:**
- 11 Elastic IPs
- 30+ Network Interfaces with public IPs (ECS tasks, ALBs)
- Includes 2-3 idle addresses ($2.70/month)

**Breakdown:**
| Resource Type | Count | Monthly Cost |
|---------------|-------|-------------|
| ALB Public IPs | 10 | ~$36 |
| ECS Task Public IPs | 12+ | ~$44 |
| EC2 Elastic IPs | 4 | ~$15 |
| Other/Idle | 4+ | ~$8 |

**Optimization Options:**
1. Use NAT Gateway for ECS tasks (private subnets) - saves ~$30-40/month but adds NAT cost
2. Remove idle Elastic IPs - saves ~$3/month
3. Consider IPv6 for future deployments (free)

**Note:** This is lower priority as ALBs require public IPs and the savings are moderate.

---

## What's Working Well ✅

### 1. Aurora Serverless v2 - EFFICIENT
- 4 clusters running at 0.5 ACU baseline (minimum)
- Scales to 8 ACU during peak usage
- Perfect for variable workloads
- **No changes needed**

### 2. Valkey Serverless (Staging/Beta) - EFFICIENT
- Auto-scales with demand
- No manual capacity management
- Cost scales with actual usage
- **No changes needed**

### 3. Application Load Balancers - REQUIRED
- 5 ALBs serve each environment
- Required for the current architecture
- **No changes possible without architecture redesign**

### 4. Environment Isolation - PROPER
- Staging and Beta in separate VPCs
- Good security practice
- **Maintain this pattern**

---

## Optimization Plan (Moderate Approach)

### Target: Reduce from $1,712/month to ~$850/month

### Phase 1: Immediate Wins (Week 1)

#### 1.1 Delete Orphaned ElastiCache Cluster
**Savings: $372/month**

```bash
# Step 1: Verify no connections (already confirmed - 0 cache hits)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CacheHits \
  --dimensions Name=CacheClusterId,Value=z36-app-cache-non-cluster-001 \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 604800 --statistics Sum

# Step 2: Take final backup (optional)
aws elasticache create-snapshot \
  --replication-group-id z36-app-cache-non-cluster \
  --snapshot-name z36-app-cache-final-backup

# Step 3: Delete replication group
aws elasticache delete-replication-group \
  --replication-group-id z36-app-cache-non-cluster \
  --final-snapshot-identifier z36-app-cache-final
```

**Who Does This:** You (requires confirmation for deletion)

#### 1.2 Archive and Terminate Stopped EC2 Instances
**Savings: $5.50/month**

```bash
# Create AMIs for archival
for instance in i-0140a2d42c6e8c498 i-0a65fe6e9b90743f8 i-08f8a3f5ecb1f61e1; do
  name=$(aws ec2 describe-instances --instance-ids $instance \
    --query 'Reservations[0].Instances[0].Tags[?Key==`Name`].Value' --output text)
  aws ec2 create-image --instance-id $instance \
    --name "${name}-archive-$(date +%Y%m%d)" \
    --description "Archived before termination"
done

# After AMI creation completes, terminate instances
aws ec2 terminate-instances \
  --instance-ids i-0140a2d42c6e8c498 i-0a65fe6e9b90743f8 i-08f8a3f5ecb1f61e1
```

**Who Does This:** You (requires confirmation for termination)

---

### Phase 2: ECS Rightsizing (Week 2-3)

#### 2.1 Update Staging ECS Task Definitions
**Savings: ~$100/month**

I can update the task definitions by modifying the CPU/memory values. The changes are applied on next deployment.

**Staging Services (Lower Priority):**
| Service | Current | New |
|---------|---------|-----|
| z360-staging-web | 1024/2048 | 512/1024 |
| z360-staging-queue | 512/1024 | 256/512 |
| z360-staging-reverb | 512/1024 | 256/512 |
| agent-staging-gateway | 512/1024 | 256/512 |
| agent-staging-aegra | 2048/4096 | 512/1024 |
| agent-staging-voice | 4096/8192 | 1024/4096 |

**Who Does This:** I can prepare the task definition updates; you trigger deployments

#### 2.2 Update Beta ECS Task Definitions
**Savings: ~$270/month**

**Beta Services (Sized for 100 Concurrent Businesses):**
| Service | Current | New | Rationale |
|---------|---------|-----|-----------|
| z360-beta-web | 2048/4096 | 1024/2048 | Laravel handles 500+ req/s per vCPU |
| z360-beta-queue | 512/1024 | 512/1024 | Keep for queue headroom |
| z360-beta-reverb | 512/1024 | 256/512 | WebSocket server is lightweight |
| agent-beta-gateway | 512/1024 | 512/1024 | Keep for API routing |
| agent-beta-aegra | 2048/4096 | 512/1024 | Per Compute Optimizer |
| agent-beta-voice | 4096/8192 | 2048/4096 | Voice needs headroom |

**Who Does This:** I can prepare the task definition updates; you trigger deployments

---

### Phase 3: EC2 Rightsizing (Week 4)

#### 3.1 Downsize Zscribe360 Instance
**Savings: $16/month**

```bash
# Step 1: Stop instance
aws ec2 stop-instances --instance-ids i-0dcea86e8744c74bc

# Step 2: Change instance type
aws ec2 modify-instance-attribute \
  --instance-id i-0dcea86e8744c74bc \
  --instance-type "{\"Value\": \"t3.small\"}"

# Step 3: Start instance
aws ec2 start-instances --instance-ids i-0dcea86e8744c74bc
```

**Who Does This:** I can execute (with your permission during maintenance window)

---

### Phase 4: Savings Plan (Week 5)

#### 4.1 Purchase 1-Year Compute Savings Plan
**Savings: $144/month**

Based on AWS recommendations for your workload:
- **Commitment:** $0.74/hour
- **Term:** 1 Year, No Upfront
- **Coverage:** Fargate, EC2, Lambda
- **Savings Rate:** 19%

```bash
# View recommendation details
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS
```

**Who Does This:** You (financial commitment in AWS console)

---

## Optimization Summary

| Phase | Action | Monthly Savings | Cumulative |
|-------|--------|-----------------|------------|
| **Phase 1** | Delete ElastiCache cluster | $372 | $372 |
| **Phase 1** | Terminate stopped EC2s | $5.50 | $377.50 |
| **Phase 2** | Rightsize Staging ECS | $100 | $477.50 |
| **Phase 2** | Rightsize Beta ECS | $270 | $747.50 |
| **Phase 3** | Rightsize Zscribe360 EC2 | $16 | $763.50 |
| **Phase 4** | 1-Year Savings Plan | $144 | **$907.50** |

### Before/After Comparison

| Component | Current Monthly | Optimized Monthly | Savings |
|-----------|----------------|-------------------|---------|
| ECS Fargate | $634 | $264 | $370 |
| ElastiCache | $509 | $137* | $372 |
| EC2 | $101 | $80 | $21 |
| Other | $468 | $468 | $0 |
| Savings Plan | - | -$144 | $144 |
| **TOTAL** | **$1,712** | **$805** | **$907** |

*Valkey Serverless remains at ~$137/month

---

## New Architecture (Post-Optimization)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  DEFAULT VPC (vpc-06fab8e58cd12e4a9)                                         │
│  ┌────────────────────┐                                                      │
│  │ Zscribe360         │  ← Downsized to t3.small                            │
│  │ t3.small ($17/mo)  │  ← CPU: 4.8% avg is fine for t3.small              │
│  │ → app-rds MySQL    │                                                      │
│  └────────────────────┘                                                      │
│                           ❌ ElastiCache cluster DELETED (was $372/mo)       │
│                           ❌ Stopped EC2 instances TERMINATED                │
├──────────────────────────────────────────────────────────────────────────────┤
│  COPILOT-MANAGED VPCs (4)                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Z360 STAGING (vpc-065b29b166c7dcf63)                                   │ │
│  │  ├─ web (512/1024) ─────────┐                                          │ │
│  │  ├─ queue (256/512)         ├──→ Aurora Serverless (0.5-8 ACU)         │ │
│  │  └─ reverb (256/512)        └──→ Valkey Serverless                     │ │
│  │  Monthly: ~$60                                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Z360 BETA (vpc-07f099e800d2490cc)                                      │ │
│  │  ├─ web (1024/2048) ────────┐                                          │ │
│  │  ├─ queue (512/1024)        ├──→ Aurora Serverless (0.5-8 ACU)         │ │
│  │  └─ reverb (256/512)        └──→ Valkey Serverless                     │ │
│  │  Monthly: ~$90                                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  AGENT LAYER STAGING (vpc-01379062cfa09a891)                            │ │
│  │  ├─ gateway (256/512) ──────┐                                          │ │
│  │  ├─ aegra (512/1024)        ├──→ Aurora Serverless (0.5-8 ACU)         │ │
│  │  └─ voice (1024/4096)       │                                          │ │
│  │  Monthly: ~$70                                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  AGENT LAYER BETA (vpc-0adfa9e056db7d9b8)                               │ │
│  │  ├─ gateway (512/1024) ─────┐                                          │ │
│  │  ├─ aegra (512/1024)        ├──→ Aurora Serverless (0.5-8 ACU)         │ │
│  │  └─ voice (2048/4096)       │                                          │ │
│  │  Monthly: ~$110                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────────┤
│  SHARED SERVICES                                                             │
│  ├─ 5x ALBs ($80/mo) - Required for traffic routing                         │
│  ├─ 4x Aurora Serverless ($180/mo) - Database layer                         │
│  ├─ 2x Valkey Serverless ($137/mo) - Caching layer                          │
│  ├─ WAF ($30/mo) - Security                                                 │
│  └─ Other (Route53, KMS, ECR, etc.) ($50/mo)                                │
└──────────────────────────────────────────────────────────────────────────────┘

Total Optimized Monthly Cost: ~$805
+ 1-Year Savings Plan Discount: -$144
= NET MONTHLY COST: ~$661
```

---

## Future Optimization Opportunities

Beyond the moderate optimization plan, here are additional options for further cost reduction:

### 1. Self-Hosted Redis/Valkey (Advanced)

**Current:** Valkey Serverless = ~$137/month
**Alternative:** Self-hosted Redis on t3.small = ~$20/month

| Aspect | Valkey Serverless | Self-Hosted Redis |
|--------|-------------------|-------------------|
| Monthly Cost | $137 | ~$20 |
| Operational Overhead | None | High (patching, monitoring, backups) |
| High Availability | Built-in | Must configure replication |
| Scaling | Automatic | Manual |
| **Recommendation** | Keep for now | Only if ops team capacity exists |

**Savings if implemented:** $117/month
**Trade-off:** Requires DevOps resources for management

### 2. Database Consolidation (Advanced)

**Current:** 4 Aurora Serverless clusters (staging + beta for z360 + agent-layer)
**Alternative:** Consolidate staging environments to shared cluster

| Configuration | Monthly Cost | Risk |
|---------------|-------------|------|
| 4 clusters (current) | $180 | Low - isolated |
| 3 clusters (merge staging) | $135 | Medium - shared resources |
| 2 clusters (staging + beta) | $90 | Higher - less isolation |

**Savings if implemented:** $45-90/month
**Trade-off:** Reduced environment isolation

### 3. Staging Environment Scheduling (Moderate)

**Current:** Staging runs 24/7
**Alternative:** Schedule staging to run only during business hours

| Schedule | Hours/Week | Monthly Cost | Savings |
|----------|------------|--------------|---------|
| 24/7 (current) | 168 | $130 | - |
| Mon-Fri 6am-10pm | 80 | $62 | $68 |
| Mon-Fri 9am-6pm | 45 | $35 | $95 |

**Implementation:** Use AWS Instance Scheduler or Lambda to stop/start ECS services

**Savings if implemented:** $68-95/month
**Trade-off:** Staging unavailable during off-hours

### 4. Move Zscribe360 to Fargate (Architectural Change)

**Current:** EC2 t3.small = $17/month + EBS + maintenance
**Alternative:** Containerize and run on Fargate

| Approach | Monthly Cost | Benefits |
|----------|-------------|----------|
| EC2 t3.small | $17 | Simple, familiar |
| Fargate (0.25vCPU/0.5GB) | $9 | No patching, auto-scaling |

**Savings if implemented:** $8/month
**Trade-off:** Requires containerization effort

### 5. Alternative Hosting Providers (Major Change)

For reference only - significant migration effort required:

| Service | AWS Current | Alternative | Potential Savings |
|---------|------------|-------------|-------------------|
| Fargate | $264/mo | Fly.io, Railway | 30-50% |
| Aurora | $180/mo | PlanetScale, Neon | 20-40% |
| Redis | $137/mo | Upstash, Redis Cloud | 30-50% |

**Note:** These alternatives sacrifice AWS ecosystem integration, support, and compliance certifications. Not recommended unless significant cost pressure exists.

---

## Action Items Summary

### What I Can Execute (With Your Permission)

| Action | Risk | Requires |
|--------|------|----------|
| Prepare ECS task definition updates | None | Your deployment trigger |
| Execute EC2 instance type change | Low | Maintenance window approval |
| Generate CloudFormation/Copilot config changes | None | Your review and apply |

### What You Need To Do

| Action | Priority | Estimated Effort |
|--------|----------|------------------|
| Delete ElastiCache replication group | HIGH | 5 minutes |
| Create AMIs for stopped EC2 instances | MEDIUM | 15 minutes |
| Terminate stopped EC2 instances | MEDIUM | 5 minutes |
| Trigger ECS service redeployments | MEDIUM | 30 minutes |
| Purchase Savings Plan (AWS Console) | LOW | 10 minutes |

---

## Risk Assessment

| Change | Risk Level | Mitigation |
|--------|-----------|------------|
| Delete ElastiCache cluster | **LOW** | 0 cache hits confirms no usage |
| Terminate stopped EC2s | **LOW** | Create AMI backups first |
| Rightsize ECS staging | **LOW** | Non-production, easy to revert |
| Rightsize ECS beta | **MEDIUM** | Deploy during low-traffic, monitor |
| Rightsize Zscribe360 | **LOW** | Quick to resize back if needed |
| Savings Plan | **LOW** | 1-year commitment is reasonable |

---

## Verification Checklist

After optimization, verify:

- [ ] AWS Cost Explorer shows daily spend reduction within 48 hours
- [ ] All ECS services remain healthy (`aws ecs describe-services`)
- [ ] CloudWatch alarms not triggered for CPU/memory
- [ ] Application response times unchanged (check ALB metrics)
- [ ] Staging environment accessible and functional
- [ ] Beta environment handling normal traffic
- [ ] Zscribe360 transcription service operational
- [ ] Database connections successful across all environments

---

## Summary

### Current State
- Monthly spend: $1,712 (excluding Bedrock)
- Major waste: $372/month on orphaned ElastiCache
- Significant overprovisioning: 50-85% on ECS services
- Underutilized EC2: Zscribe360 at 4.8% CPU

### Optimized State
- Monthly spend: ~$661 (with Savings Plan)
- Annual savings: ~$12,600
- Clean architecture: No orphaned resources
- Right-sized services: Based on actual utilization data

### Key Actions
1. **Immediate:** Delete orphaned ElastiCache ($372/mo savings)
2. **Week 2-3:** Rightsize all ECS services ($370/mo savings)
3. **Week 4:** Rightsize Zscribe360 EC2 ($16/mo savings)
4. **Week 5:** Purchase 1-Year Savings Plan ($144/mo savings)

---

*Report generated from live AWS infrastructure analysis*
*Data source: AWS Cost Explorer API, CloudWatch Metrics, Compute Optimizer*
