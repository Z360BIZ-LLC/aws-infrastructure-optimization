# AWS Infrastructure Optimization Project

A Claude Code-powered repository for managing, optimizing, and maintaining AWS infrastructure.

## Overview

This project provides AI-assisted infrastructure management for the **z360** application ecosystem, including:
- Cost analysis and optimization
- Resource auditing and inventory
- Security assessment
- Ongoing maintenance workflows

## AWS Account Info

- **Account ID**: 108782097218
- **Primary Region**: us-east-1
- **Environments**: staging, beta

## Quick Start

### Prerequisites

1. **AWS CLI** - Must be authenticated with appropriate permissions
   ```bash
   aws sts get-caller-identity  # Verify authentication
   ```

2. **uv** - Python package manager for MCP servers (already installed)
   ```bash
   uv --version  # Should show 0.8.x or higher
   ```

3. **Claude Code** - AI assistant CLI

### Using This Project

Open this directory in Claude Code:
```bash
cd "/Users/ansar/Documents/AWS INFRA"
claude
```

### Available Commands

| Command | Description |
|---------|-------------|
| `/project:cost-report` | Generate AWS cost analysis report |
| `/project:resource-audit` | Audit all AWS resources |
| `/project:security-check` | Run security assessment |
| `/project:optimize` | Run optimization workflow |

## Project Structure

```
AWS INFRA/
├── README.md                 # This file
├── CLAUDE.md                 # Claude Code context and instructions
├── .mcp.json                 # MCP server configuration
├── .claude/
│   ├── settings.json         # Claude Code settings with safety hooks
│   ├── commands/             # Slash commands
│   │   ├── cost-report.md
│   │   ├── resource-audit.md
│   │   ├── security-check.md
│   │   └── optimize.md
│   └── agents/               # Subagent definitions
│       ├── cost-analyzer.md
│       ├── security-auditor.md
│       └── resource-manager.md
├── docs/
│   ├── architecture/         # Infrastructure documentation
│   ├── runbooks/             # Operational procedures
│   └── optimization-plans/   # Saved optimization recommendations
├── reports/                  # Generated reports
└── scripts/aws/              # Helper scripts
```

## MCP Servers

This project uses AWS MCP servers for enhanced Claude capabilities:

| Server | Purpose |
|--------|---------|
| `awslabs.billing-cost-management-mcp-server` | Cost Explorer, Budgets, Compute Optimizer |
| `awslabs.cloudwatch-mcp-server` | Log analysis, metrics, alarms |
| `awslabs.ccapi-mcp-server` | Cloud Control API for resource management |
| `awslabs.aws-documentation-mcp-server` | AWS documentation access |

### Setting Up MCP Servers

Run this command to add the MCP servers:
```bash
claude mcp add awslabs.billing-cost-management-mcp-server -s user -e AWS_REGION=us-east-1
claude mcp add awslabs.cloudwatch-mcp-server -s user -e AWS_REGION=us-east-1
claude mcp add awslabs.ccapi-mcp-server -s user -e AWS_REGION=us-east-1
claude mcp add awslabs.aws-documentation-mcp-server -s user
```

Verify with:
```bash
claude mcp list
```

## Infrastructure Summary

### Current Resources (us-east-1)

| Service | Count | Details |
|---------|-------|---------|
| EC2 Instances | 4 | 1 running, 3 stopped |
| RDS Databases | 5 | 1 MySQL, 4 Aurora PostgreSQL Serverless |
| ECS Clusters | 4 | z360 staging/beta, agent-layer staging/beta |
| Lambda Functions | 33+ | Mostly z360 service functions |
| S3 Buckets | 9 | Application storage |
| ElastiCache | 3 | Valkey nodes (cache.r7g.large) |
| CloudFront | 4 | CDN distributions |
| Load Balancers | 5 | Application load balancers |
| SQS Queues | 1 | Z360AlphaSQS |

## Workflows

### 1. Cost Optimization
```
/project:cost-report
```
Analyzes spending patterns and identifies optimization opportunities.

### 2. Resource Audit
```
/project:resource-audit
```
Generates complete inventory of all AWS resources.

### 3. Security Assessment
```
/project:security-check
```
Checks for security misconfigurations and compliance issues.

### 4. Optimization Planning
```
/project:optimize
```
Interactive workflow to plan and implement optimizations.

## Safety Rules

- **Never delete resources** without explicit confirmation
- **Always plan before apply** for any infrastructure changes
- **Log all operations** for audit trail
- **Test in staging** before production changes

## Notes

- Cost Explorer access may need to be enabled in AWS Console
- Some operations require specific IAM permissions
- Reports are saved to `reports/` directory
