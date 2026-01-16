# Z360 AWS Infrastructure Architecture

**Prepared for:** AWS Solutions Architect Meeting
**Date:** January 16, 2026
**AWS Account ID:** 108782097218
**Primary Region:** us-east-1

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What is Z360?](#what-is-z360)
3. [High-Level Architecture](#high-level-architecture)
4. [Infrastructure Components](#infrastructure-components)
5. [Network Architecture](#network-architecture)
6. [Data Flow & Communication](#data-flow--communication)
7. [Auto-Scaling Configuration](#auto-scaling-configuration)
8. [Security Mechanisms](#security-mechanisms)
9. [Monitoring & Alerting](#monitoring--alerting)
10. [Cost Analysis](#cost-analysis)
11. [Recent Optimizations](#recent-optimizations)

---

## Executive Summary

Z360 is a multi-tenant SaaS platform running on AWS using a fully serverless/containerized architecture. The infrastructure supports two main product lines across staging and beta environments:

| Metric | Value |
|--------|-------|
| **ECS Clusters** | 4 (ACTIVE) |
| **ECS Services** | 12 (all FARGATE) |
| **Running Tasks** | 13 |
| **Aurora Databases** | 4 (Serverless v2) |
| **Cache Clusters** | 2 (Valkey Serverless) |
| **Load Balancers** | 4 (Application) |
| **CloudFront CDNs** | 3 (Enabled) |
| **Monthly Cost** | ~$500-700 (excl. Bedrock AI) |

**Infrastructure Health:** All systems operational (verified January 16, 2026)

---

## What is Z360?

### Company Overview

Z360 is a SaaS platform that provides:

1. **Z360 Core Application** - A web-based business application built with Laravel (PHP) featuring:
   - Real-time WebSocket communication (Laravel Reverb)
   - Background job processing (Laravel Queues)
   - File storage and CDN delivery

2. **Z360 Agent Layer** - AI-powered agent services for:
   - Voice processing and transcription (LiveKit integration)
   - AI agent orchestration (Aegra engine)
   - API gateway for agent services

### Product Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Z360 PLATFORM                                  │
├─────────────────────────────────┬───────────────────────────────────────┤
│       Z360 CORE                 │         Z360 AGENT LAYER              │
│                                 │                                       │
│  ┌─────────────────────────┐   │   ┌─────────────────────────────┐     │
│  │  Web Application        │   │   │  Gateway (API)              │     │
│  │  - User Interface       │   │   │  - Request routing          │     │
│  │  - API Endpoints        │   │   │  - Authentication           │     │
│  │  - Business Logic       │   │   │  - Rate limiting            │     │
│  └─────────────────────────┘   │   └─────────────────────────────┘     │
│                                 │                                       │
│  ┌─────────────────────────┐   │   ┌─────────────────────────────┐     │
│  │  Queue Worker           │   │   │  Voice Service              │     │
│  │  - Background jobs      │   │   │  - LiveKit integration      │     │
│  │  - Email notifications  │   │   │  - Audio processing         │     │
│  │  - Data processing      │   │   │  - Transcription            │     │
│  └─────────────────────────┘   │   └─────────────────────────────┘     │
│                                 │                                       │
│  ┌─────────────────────────┐   │   ┌─────────────────────────────┐     │
│  │  Reverb (WebSocket)     │   │   │  Aegra (AI Engine)          │     │
│  │  - Real-time events     │   │   │  - Agent orchestration      │     │
│  │  - Live notifications   │   │   │  - Bedrock integration      │     │
│  │  - Presence channels    │   │   │  - Workflow execution       │     │
│  └─────────────────────────┘   │   └─────────────────────────────┘     │
│                                 │                                       │
└─────────────────────────────────┴───────────────────────────────────────┘
```

### Environments

| Environment | Purpose | Domain | Traffic |
|-------------|---------|--------|---------|
| **Staging** | Development, internal testing | staging.z360.z360.biz | Low |
| **Beta** | Pre-production, customer demos, early adopters | beta.z360.z360.biz | Moderate, variable |

---

## High-Level Architecture

```
                                 ┌─────────────────────────────────────┐
                                 │            INTERNET                  │
                                 └─────────────────┬───────────────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
                    ▼                              ▼                              ▼
          ┌─────────────────┐            ┌─────────────────┐            ┌─────────────────┐
          │   CloudFront    │            │    Route 53     │            │   WAF (5 ACLs)  │
          │   CDN (x3)      │            │  DNS Routing    │            │  Web Security   │
          │                 │            │                 │            │                 │
          │ storage.app     │            │ z360.biz        │            │ Amplify-managed │
          │ cdn.staging     │            │ *.z360.z360.biz │            │ DDoS protection │
          │ cdn.beta        │            │ *.agent.z360.biz│            │ Bot filtering   │
          └────────┬────────┘            └────────┬────────┘            └─────────────────┘
                   │                              │
                   │    ┌─────────────────────────┴─────────────────────────┐
                   │    │                                                   │
                   ▼    ▼                                                   ▼
          ┌─────────────────┐                                      ┌─────────────────┐
          │   S3 Buckets    │                                      │  ACM Certs (11) │
          │   (10 total)    │                                      │  SSL/TLS        │
          │                 │                                      │                 │
          │ app-z360        │                                      │ *.z360.biz      │
          │ livkit-record.  │                                      │ *.z360.z360.biz │
          │ staging-storage │                                      │ *.agent.z360.biz│
          │ beta-storage    │                                      └─────────────────┘
          └─────────────────┘
                                                   │
                ┌──────────────────────────────────┴──────────────────────────────────┐
                │                    APPLICATION LOAD BALANCERS (4)                    │
                │                                                                      │
                │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
                │  │ z360-staging    │  │ z360-beta       │  │ agent-layer-staging │  │
                │  │ ALB             │  │ ALB             │  │ ALB                 │  │
                │  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘  │
                │           │                    │                      │             │
                │  ┌────────┴────────┐  ┌────────┴────────┐  ┌─────────┴──────────┐  │
                │  │ agent-layer-beta│                                              │  │
                │  │ ALB             │                                              │  │
                │  └────────┬────────┘                                              │  │
                └───────────┼───────────────────────────────────────────────────────┘
                            │
       ┌────────────────────┼────────────────────┬────────────────────┐
       │                    │                    │                    │
       ▼                    ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Z360      │      │   Z360      │      │ Agent Layer │      │ Agent Layer │
│  STAGING    │      │   BETA      │      │  STAGING    │      │   BETA      │
│  Cluster    │      │  Cluster    │      │   Cluster   │      │   Cluster   │
├─────────────┤      ├─────────────┤      ├─────────────┤      ├─────────────┤
│ web     (1) │      │ web     (1) │      │ gateway (1) │      │ gateway (1) │
│ queue   (1) │      │ queue   (1) │      │ aegra   (1) │      │ aegra   (1) │
│ reverb  (1) │      │ reverb  (1) │      │ voice   (1) │      │ voice   (1) │
└──────┬──────┘      └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
       │                    │                    │                    │
       │    ┌───────────────┴────────────────────┴────────────────────┘
       │    │
       ▼    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                      │
│                                                                              │
│   ┌───────────────────────────────┐    ┌───────────────────────────────┐    │
│   │     AURORA SERVERLESS v2      │    │      VALKEY SERVERLESS        │    │
│   │     PostgreSQL 16.8           │    │      (Redis-compatible)       │    │
│   │                               │    │                               │    │
│   │  ┌─────────┐  ┌─────────┐    │    │  ┌─────────┐  ┌─────────┐    │    │
│   │  │ staging │  │  beta   │    │    │  │ staging │  │  beta   │    │    │
│   │  │   DB    │  │   DB    │    │    │  │  cache  │  │  cache  │    │    │
│   │  └─────────┘  └─────────┘    │    │  └─────────┘  └─────────┘    │    │
│   │                               │    │                               │    │
│   │  ┌─────────┐  ┌─────────┐    │    │  Auto-scaling ECPU billing    │    │
│   │  │ agent   │  │ agent   │    │    │  Pay-per-request model        │    │
│   │  │ staging │  │  beta   │    │    │                               │    │
│   │  └─────────┘  └─────────┘    │    │                               │    │
│   └───────────────────────────────┘    └───────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure Components

### 1. Compute - ECS Fargate

All application services run on **AWS ECS Fargate** (serverless containers). No EC2 instances to manage.

#### Cluster Configuration

| Cluster | VPC | Services | Tasks | Status |
|---------|-----|----------|-------|--------|
| z360-staging-Cluster | copilot-z360-staging | 3 | 4 | ACTIVE |
| z360-beta-Cluster | copilot-z360-beta | 3 | 3 | ACTIVE |
| z360-agent-layer-staging-Cluster | copilot-z360-agent-layer-staging | 3 | 3 | ACTIVE |
| z360-agent-layer-beta-Cluster | copilot-z360-agent-layer-beta | 3 | 3 | ACTIVE |

#### Service Details

**Z360 Core Services:**

| Service | Purpose | CPU | Memory | Tasks |
|---------|---------|-----|--------|-------|
| **web** | Main Laravel application server | 512-1024 | 1024-2048 MB | 1 |
| **queue** | Laravel Queue worker for background jobs | 256 | 512 MB | 1 |
| **reverb** | Laravel Reverb WebSocket server | 256 | 512 MB | 1 |

**Agent Layer Services:**

| Service | Purpose | CPU | Memory | Tasks |
|---------|---------|-----|--------|-------|
| **gateway** | API gateway, request routing, auth | 512 | 1024 MB | 1 |
| **aegra** | AI agent orchestration engine | 256-512 | 1024 MB | 1 |
| **voice** | Voice processing, LiveKit integration | 1024-2048 | 4096-5120 MB | 1 |

### 2. Database - Aurora Serverless v2

All databases use **Aurora PostgreSQL Serverless v2** for automatic scaling and pay-per-use pricing.

| Database | Engine | Version | Endpoint | Purpose |
|----------|--------|---------|----------|---------|
| z360-staging-db | Aurora PostgreSQL | 16.8 | z360-staging-...cluster-clu88o4y4e7g.us-east-1.rds.amazonaws.com | Staging app data |
| z360-beta-db | Aurora PostgreSQL | 16.8 | z360-beta-...cluster-clu88o4y4e7g.us-east-1.rds.amazonaws.com | Beta app data |
| agent-staging-db | Aurora PostgreSQL | 16.8 | z360-agent-layer-staging-...cluster-clu88o4y4e7g.us-east-1.rds.amazonaws.com | Agent staging data |
| agent-beta-db | Aurora PostgreSQL | 16.8 | z360-agent-layer-beta-...cluster-clu88o4y4e7g.us-east-1.rds.amazonaws.com | Agent beta data |

**Configuration:**
- Instance class: db.serverless
- Auto-scaling: 0.5 - 8 ACU (Aurora Capacity Units)
- Storage: Auto-scaling
- Backup: Automated with 7-day retention

### 3. Cache - Valkey Serverless

Using **Valkey Serverless** (Redis-compatible) for session storage and caching.

| Cache | Engine | Endpoint | Purpose |
|-------|--------|----------|---------|
| z360-staging-valkey | Valkey | z360-staging-valkey-hbph82.serverless.use1.cache.amazonaws.com:6379 | Staging cache/sessions |
| z360-beta-valkey | Valkey | z360-beta-valkey-hbph82.serverless.use1.cache.amazonaws.com:6379 | Beta cache/sessions |

**Configuration:**
- ECPU-based billing (pay per request)
- TLS encryption enabled
- Auto-scaling capacity

### 4. Load Balancing - Application Load Balancers

| Load Balancer | DNS Name | VPC | Purpose |
|---------------|----------|-----|---------|
| z360-s-Publi-hdXAuvDlRXF6 | z360-s-Publi-hdXAuvDlRXF6-571871136.us-east-1.elb.amazonaws.com | z360-staging | Staging traffic |
| z360-b-Publi-VVFRqJywF53r | z360-b-Publi-VVFRqJywF53r-484439536.us-east-1.elb.amazonaws.com | z360-beta | Beta traffic |
| z360-a-Publi-71IFfgE12f4T | z360-a-Publi-71IFfgE12f4T-1444548936.us-east-1.elb.amazonaws.com | agent-staging | Agent staging |
| z360-a-Publi-zQsATBo5XNth | z360-a-Publi-zQsATBo5XNth-958538622.us-east-1.elb.amazonaws.com | agent-beta | Agent beta |

**Features:**
- HTTPS termination with ACM certificates
- HTTP to HTTPS redirect
- Health check endpoints
- Cross-zone load balancing

### 5. Storage - S3 Buckets

| Bucket | Purpose | CDN |
|--------|---------|-----|
| app-z360 | Main application assets | storage.app.z360.biz |
| z360-staging-...-storagebucket | Staging user uploads | cdn.staging.z360.biz |
| z360-beta-...-storagebucket | Beta user uploads | cdn.beta.z360.biz |
| livkit-recordings | Voice call recordings | - |
| zscribe-main | Transcription files | - |
| z360-infrastructure-backups | Infrastructure backups | - |
| stackset-z360-* (2) | CI/CD pipeline artifacts | - |
| task-z360-s3bucket | Task/workflow storage | - |

### 6. CDN - CloudFront Distributions

| Distribution | Domain | Origin | Custom Domain |
|--------------|--------|--------|---------------|
| EHIL1J4E39NR | dwr3e0krmz5y.cloudfront.net | app-z360 S3 | storage.app.z360.biz |
| EAM1UA1FIP41U | d3g2j2avgt60r7.cloudfront.net | staging-storagebucket | cdn.staging.z360.biz |
| E1MVRXC5JJEU5X | d2iu8tqsgvsqff.cloudfront.net | beta-storagebucket | cdn.beta.z360.biz |

### 7. DNS - Route 53

**Public Hosted Zones (9):**

| Zone | Purpose |
|------|---------|
| z360.biz | Primary domain |
| z360.z360.biz | Application subdomain |
| staging.z360.z360.biz | Staging environment |
| beta.z360.z360.biz | Beta environment |
| z360-agent-layer.z360.biz | Agent layer root |
| staging.z360-agent-layer.z360.biz | Agent staging |
| beta.z360-agent-layer.z360.biz | Agent beta |
| archehq.com | Company domain |
| zscribe360.com | Zscribe service |

**Private Hosted Zones (4):** Internal service discovery for each environment

---

## Network Architecture

### VPC Configuration

| VPC | CIDR | Name | Purpose |
|-----|------|------|---------|
| vpc-065b29b166c7dcf63 | 10.0.0.0/16 | copilot-z360-staging | Z360 Core Staging |
| vpc-07f099e800d2490cc | 10.0.0.0/16 | copilot-z360-beta | Z360 Core Beta |
| vpc-01379062cfa09a891 | 10.0.0.0/16 | copilot-z360-agent-layer-staging | Agent Layer Staging |
| vpc-0adfa9e056db7d9b8 | 10.0.0.0/16 | copilot-z360-agent-layer-beta | Agent Layer Beta |

### Subnet Architecture (per VPC)

```
VPC (10.0.0.0/16)
│
├── Public Subnets (ALB, NAT)
│   ├── 10.0.0.0/24 (us-east-1a) - pub0
│   └── 10.0.1.0/24 (us-east-1b) - pub1
│
└── Private Subnets (ECS Tasks, Databases)
    ├── 10.0.2.0/24 (us-east-1a) - priv0
    └── 10.0.3.0/24 (us-east-1b) - priv1
```

### Network Flow

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                    PUBLIC SUBNETS                        │
│                                                          │
│   ┌─────────────────────┐   ┌─────────────────────┐     │
│   │  Application Load   │   │   Internet Gateway   │     │
│   │     Balancer        │   │                      │     │
│   │  (Elastic IPs)      │   │                      │     │
│   └──────────┬──────────┘   └──────────────────────┘     │
│              │                                           │
└──────────────┼───────────────────────────────────────────┘
               │
┌──────────────┼───────────────────────────────────────────┐
│              ▼            PRIVATE SUBNETS                │
│                                                          │
│   ┌─────────────────────┐   ┌─────────────────────┐     │
│   │   ECS Fargate       │   │   ECS Fargate       │     │
│   │   Tasks             │   │   Tasks             │     │
│   │   (us-east-1a)      │   │   (us-east-1b)      │     │
│   └──────────┬──────────┘   └──────────┬──────────┘     │
│              │                         │                 │
│              └────────────┬────────────┘                 │
│                           │                              │
│   ┌─────────────────────┐ │ ┌─────────────────────┐     │
│   │   Aurora Serverless │ │ │   Valkey Serverless │     │
│   │   (Multi-AZ)        │◄┘►│   (Redis-compatible)│     │
│   └─────────────────────┘   └─────────────────────┘     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Note:** No NAT Gateways are deployed. ECS tasks in public subnets with security groups control access.

---

## Data Flow & Communication

### Request Flow - Web Application

```
┌────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ User   │────▶│ Route53 │────▶│   ALB   │────▶│   ECS   │────▶│ Aurora  │
│Browser │     │  (DNS)  │     │ (HTTPS) │     │  (web)  │     │   DB    │
└────────┘     └─────────┘     └─────────┘     └─────────┘     └────┬────┘
                                                    │               │
                                                    ▼               │
                                              ┌─────────┐          │
                                              │ Valkey  │◄─────────┘
                                              │ (cache) │
                                              └─────────┘
```

### Request Flow - WebSocket Connection

```
┌────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ User   │────▶│ Route53 │────▶│   ALB   │────▶│   ECS   │
│Browser │     │  (DNS)  │     │  (WSS)  │     │(reverb) │
└────────┘     └─────────┘     └─────────┘     └────┬────┘
    ▲                                               │
    │                                               ▼
    │                                         ┌─────────┐
    └─────────────────────────────────────────│ Valkey  │
              Real-time Events                │(pub/sub)│
                                              └─────────┘
```

### Request Flow - AI Agent Processing

```
┌────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Client │────▶│ Route53 │────▶│   ALB   │────▶│   ECS   │────▶│   ECS   │
│  App   │     │  (DNS)  │     │ (HTTPS) │     │(gateway)│     │ (aegra) │
└────────┘     └─────────┘     └─────────┘     └─────────┘     └────┬────┘
                                                                    │
                    ┌───────────────────────────────────────────────┘
                    │
                    ▼
              ┌─────────┐     ┌─────────┐     ┌─────────┐
              │ Amazon  │     │  Aurora │     │   ECS   │
              │ Bedrock │     │   DB    │     │ (voice) │
              │(Claude) │     │         │     │(LiveKit)│
              └─────────┘     └─────────┘     └─────────┘
```

### Background Job Processing

```
┌─────────┐          ┌─────────┐          ┌─────────┐
│   ECS   │  ─────▶  │   SQS   │  ─────▶  │   ECS   │
│  (web)  │  enqueue │  Queue  │  dequeue │ (queue) │
└─────────┘          └─────────┘          └────┬────┘
                                               │
                    ┌──────────────────────────┤
                    │                          │
                    ▼                          ▼
              ┌─────────┐              ┌─────────────┐
              │ Aurora  │              │   SES/SNS   │
              │   DB    │              │(Emails/SMS) │
              └─────────┘              └─────────────┘
```

---

## Auto-Scaling Configuration

### Scalable Services (6 Beta Services)

Auto-scaling is configured for all **beta environment** services to handle production traffic.

| Service | Min Tasks | Max Tasks | CPU Target | Memory Target |
|---------|-----------|-----------|------------|---------------|
| z360-beta-web | 1 | 50 | 70% | 70% |
| z360-beta-queue | 1 | 20 | 70% | 70% |
| z360-beta-reverb | 1 | 20 | 70% | 70% |
| z360-agent-layer-beta-gateway | 1 | 50 | 60% | 60% |
| z360-agent-layer-beta-aegra | 1 | 30 | 70% | 70% |
| z360-agent-layer-beta-voice | 1 | 50 | 70% | 60% |

### Scaling Policies

**12 Target Tracking Policies** (2 per service):

| Policy Type | Metric | Target | Scale-Out Cooldown | Scale-In Cooldown |
|-------------|--------|--------|-------------------|-------------------|
| CPU Scaling | ECSServiceAverageCPUUtilization | 60-70% | 0 seconds | 600 seconds |
| Memory Scaling | ECSServiceAverageMemoryUtilization | 60-70% | 0 seconds | 600 seconds |

**Scaling Behavior:**
- **Scale Out:** Instant (0s cooldown) - Responds immediately to traffic spikes
- **Scale In:** Conservative (600s cooldown) - Prevents thrashing, maintains stability
- **Max Capacity:** 20-50 tasks per service - Enterprise-ready for high traffic

### Capacity Calculation

```
Maximum Concurrent Capacity:
├── z360-beta-web:       50 tasks × 1024 CPU = 51,200 CPU units
├── z360-beta-queue:     20 tasks × 256 CPU  = 5,120 CPU units
├── z360-beta-reverb:    20 tasks × 256 CPU  = 5,120 CPU units
├── agent-gateway:       50 tasks × 512 CPU  = 25,600 CPU units
├── agent-aegra:         30 tasks × 256 CPU  = 7,680 CPU units
└── agent-voice:         50 tasks × 2048 CPU = 102,400 CPU units
                                              ─────────────────
Total Maximum Capacity:                        197,120 CPU units
                                              (~197 vCPUs equivalent)
```

---

## Security Mechanisms

### Network Security

#### Security Groups (24 total for Z360 resources)

| Category | Security Group | Purpose |
|----------|----------------|---------|
| **Load Balancer** | PublicHTTPLoadBalancerSecurityGroup | HTTP (80) from internet |
| **Load Balancer** | PublicHTTPSLoadBalancerSecurityGroup | HTTPS (443) from internet |
| **ECS Tasks** | EnvironmentSecurityGroup | Ingress from ALB only |
| **Database** | dbDBClusterSecurityGroup | PostgreSQL (5432) from ECS |
| **Database** | dbWorkloadSecurityGroup | DB access from workloads |
| **Cache** | cacheServerlessSecurityGroup | Valkey (6379) endpoint |
| **Cache** | cacheWorkloadSecurityGroup | Cache access from workloads |

#### Security Group Rules Pattern

```
Internet ──▶ ALB Security Group (80, 443)
                    │
                    ▼
         ECS Environment Security Group
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
    DB Security  Cache Security  Outbound
    Group (5432) Group (6379)    (All)
```

### SSL/TLS Certificates

**11 Active ACM Certificates:**

| Certificate | Domain | Purpose |
|-------------|--------|---------|
| app.z360.biz | Main app | Application endpoint |
| static.z360.biz | Static assets | CDN assets |
| demo.z360.biz | Demo environment | Demos |
| storage.app.z360.biz | Storage API | S3 presigned URLs |
| staging.z360.z360.biz | Staging | Development |
| beta.z360.z360.biz | Beta | Pre-production |
| cdn.staging.z360.biz | Staging CDN | CloudFront |
| cdn.beta.z360.biz | Beta CDN | CloudFront |
| *.staging.agent.z360.biz | Agent wildcard | Agent services |
| staging.z360-agent-layer.z360.biz | Agent staging | Agent API |
| beta.z360-agent-layer.z360.biz | Agent beta | Agent API |

### WAF Protection

**5 WAF Web ACLs** (Amplify-managed, CloudFront scope):
- DDoS protection
- Bot filtering
- SQL injection prevention
- XSS protection
- Rate limiting

### Secrets Management

**AWS Secrets Manager** stores all sensitive credentials:

| Secret | Purpose |
|--------|---------|
| dbAuroraSecret-* | Aurora database credentials (auto-rotated) |
| agentdbAuroraSecret-* | Agent layer DB credentials (auto-rotated) |
| rds-db-credentials/cluster-* | PostgreSQL auto-managed credentials |
| prod/db | Production database connection |

### VPC Isolation

Each environment runs in a **dedicated VPC** with:
- No cross-VPC communication (complete isolation)
- Private subnets for compute and data
- Public subnets only for load balancers
- No NAT Gateways (cost optimization)

---

## Monitoring & Alerting

### CloudWatch Alarms (8 Active)

| Alarm | Metric | Threshold | State |
|-------|--------|-----------|-------|
| z360-beta-web-high-cpu | CPUUtilization | > 85% | OK |
| z360-beta-web-high-memory | MemoryUtilization | > 85% | OK |
| z360-beta-web-task-count-spike | RunningTaskCount | > 10 | INSUFFICIENT_DATA |
| z360-beta-gateway-high-cpu | CPUUtilization | > 80% | OK |
| z360-beta-gateway-high-memory | MemoryUtilization | > 75% | OK |
| z360-beta-gateway-task-count-spike | RunningTaskCount | > 10 | INSUFFICIENT_DATA |
| z360-beta-voice-high-memory | MemoryUtilization | > 80% | OK |
| z360-beta-voice-task-count-spike | RunningTaskCount | > 10 | INSUFFICIENT_DATA |

### SNS Notifications

**Topic:** z360-infrastructure-alerts

| Subscriber | Protocol | Status |
|------------|----------|--------|
| ansar@z360.biz | Email | Confirmed |
| hammasuddin@z360.biz | Email | Pending Confirmation |

**Alarm Actions:**
- Email notification on ALARM state
- Email notification on OK recovery

### AWS Budget Alert

| Budget | Limit | Alert Threshold |
|--------|-------|-----------------|
| z360-monthly-spending-alert | $2,500 USD | 80%, 100%, 120% |

---

## Cost Analysis

### Current Daily Cost (January 16, 2026)

| Service | Daily Cost | Monthly Projection |
|---------|------------|-------------------|
| Amazon ECS (Fargate) | $6.47 | ~$194 |
| Amazon ElastiCache (Valkey) | $4.31 | ~$129 |
| Amazon RDS (Aurora) | $1.85 | ~$56 |
| Elastic Load Balancing | $0.79 | ~$24 |
| Amazon VPC | $0.76 | ~$23 |
| AWS WAF | $0.33 | ~$10 |
| EC2 (misc) | $0.21 | ~$6 |
| Other (ECR, KMS, etc.) | $0.31 | ~$9 |
| **Total (excl. Bedrock)** | **~$15/day** | **~$450-500/month** |

### Monthly Cost Breakdown (Projected)

```
┌────────────────────────────────────────────────────────────────┐
│               MONTHLY COST BREAKDOWN (~$500)                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ECS Fargate      ████████████████████████████████████  39%    │
│  (~$194)                                                        │
│                                                                 │
│  Valkey           █████████████████████████            26%     │
│  (~$129)                                                        │
│                                                                 │
│  Aurora           ███████████                          11%     │
│  (~$56)                                                         │
│                                                                 │
│  Load Balancing   █████                                5%      │
│  (~$24)                                                         │
│                                                                 │
│  VPC (IPv4)       █████                                5%      │
│  (~$23)                                                         │
│                                                                 │
│  Other            ██████████                           14%     │
│  (~$70)                                                         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### February 2026 Prediction

| Scenario | Estimate |
|----------|----------|
| Optimistic | $450-500/month |
| Conservative | $600-700/month |
| AWS Forecast (incl. Bedrock) | $1,626/month |

**Note:** AWS credits are currently covering costs. When credits expire, actual bills will apply.

---

## Recent Optimizations

### Completed Optimizations (January 2026)

| Optimization | Savings | Status |
|--------------|---------|--------|
| Deleted orphaned ElastiCache cluster (3x r7g.large) | $391/month | ✅ |
| Deleted orphaned MySQL RDS (db.t4g.micro) | $14/month | ✅ |
| Terminated 4 unused EC2 instances | $47/month | ✅ |
| Right-sized 12 ECS task definitions | ~$435/month | ✅ |
| Configured auto-scaling (6 services) | Safety net | ✅ |
| Set up CloudWatch alarms (8) | Monitoring | ✅ |
| Created SNS alert topic | Notifications | ✅ |
| Configured $2,500 budget alert | Cost control | ✅ |

### Cost Impact

| Period | Monthly Cost |
|--------|-------------|
| December 2025 (before) | ~$1,765 |
| January 2026 (after) | ~$450-500 |
| **Savings** | **~$1,015-1,315/month** |
| **Annual Savings** | **~$12,180-15,780** |

---

## Infrastructure Summary

### What We're Running

| Category | Resources | Status |
|----------|-----------|--------|
| **Compute** | 4 ECS clusters, 12 services, 13 tasks | All ACTIVE |
| **Database** | 4 Aurora Serverless v2 clusters | All available |
| **Cache** | 2 Valkey Serverless clusters | All available |
| **Load Balancing** | 4 Application Load Balancers | All active |
| **CDN** | 3 CloudFront distributions | All enabled |
| **Storage** | 10 S3 buckets | All active |
| **DNS** | 9 public + 4 private hosted zones | All active |
| **Certificates** | 11 ACM certificates | All ISSUED |
| **Security** | 5 WAF ACLs, 24+ security groups | All active |

### What Makes This Architecture Cost-Effective

1. **100% Serverless/Managed:**
   - No EC2 instances to manage
   - No NAT Gateways (saves ~$65/month per VPC)
   - Pay-per-use for compute, database, and cache

2. **Right-Sized Resources:**
   - ECS tasks sized to actual workload requirements
   - Aurora Serverless scales automatically (0.5-8 ACU)
   - Valkey Serverless scales with traffic

3. **Auto-Scaling Ready:**
   - 6 production services with auto-scaling
   - Can handle traffic spikes automatically
   - Scale-in protection prevents over-scaling

4. **Environment Isolation:**
   - Staging environment minimal footprint
   - Beta environment production-ready
   - No resource sharing between environments

---

**Document Version:** 1.0
**Last Verified:** January 16, 2026
**Prepared By:** Infrastructure Team

*All data verified via AWS CLI on January 16, 2026*
