---
description: Interactive optimization workflow for AWS infrastructure
allowed-tools:
  - Bash
  - Read
  - Write
  - Task
  - AskUserQuestion
---

# AWS Infrastructure Optimization Workflow

Interactive workflow to identify, plan, and implement infrastructure optimizations.

## Workflow

### Phase 1: Discovery
1. Run resource audit to get current state
2. Run cost report to identify spending patterns
3. Run security check for compliance baseline

### Phase 2: Analysis
1. Identify quick wins (unused resources, stopped instances)
2. Find right-sizing opportunities
3. Evaluate Reserved Instance / Savings Plan potential
4. Assess architecture improvements

### Phase 3: Recommendations
Present findings to user with:
- Estimated savings for each recommendation
- Implementation complexity (easy, medium, hard)
- Risk level of changes
- Dependencies between changes

### Phase 4: Planning
For approved optimizations:
1. Create detailed implementation plan
2. Document rollback procedures
3. Define success metrics
4. Schedule implementation (staging first, then production)

### Phase 5: Implementation
With user approval:
1. Execute changes in staging environment
2. Validate changes work as expected
3. Document what was changed
4. Schedule production changes

### Phase 6: Validation
After implementation:
1. Monitor for any issues
2. Verify cost savings are realized
3. Update documentation
4. Create maintenance runbook

## Common Optimizations

### Quick Wins
- Terminate stopped EC2 instances (after backup)
- Delete unattached EBS volumes
- Remove unused Elastic IPs
- Clean up old snapshots
- Delete empty S3 buckets

### Right-Sizing
- Downsize underutilized EC2 instances
- Adjust RDS instance classes
- Optimize Lambda memory allocation
- Review ElastiCache node types

### Architecture
- Consolidate databases
- Implement auto-scaling
- Use Spot instances for non-critical workloads
- Optimize data transfer patterns

### Cost Management
- Enable Cost Explorer (if not enabled)
- Set up billing alerts
- Implement tagging strategy
- Consider Reserved Instances

## Output

Save optimization plan to `docs/optimization-plans/optimization-YYYY-MM-DD.md` with:
- Current state summary
- Proposed changes with justification
- Implementation timeline
- Expected outcomes
- Monitoring plan
