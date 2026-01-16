# Aurora PostgreSQL Encryption Migration Runbook

**Created:** January 16, 2026
**Priority:** CRITICAL - Required for HIPAA Compliance
**Estimated Duration:** 2-4 hours per cluster (with maintenance window)

---

## Overview

All 4 Aurora PostgreSQL Serverless v2 clusters are currently **NOT ENCRYPTED**. Aurora encryption cannot be enabled on existing clusters - migration to new encrypted clusters is required.

### Clusters Requiring Migration

| Cluster | Environment | Database Size | Priority |
|---------|-------------|--------------|----------|
| z360-staging-* | Staging | ~1-2 GB | 1 (migrate first) |
| z360-agent-layer-staging-* | Staging | ~500 MB | 2 |
| z360-beta-* | Beta | ~5-10 GB | 3 (production-like) |
| z360-agent-layer-beta-* | Beta | ~1 GB | 4 |

---

## Pre-Migration Checklist

- [ ] Schedule maintenance window (off-peak hours)
- [ ] Notify stakeholders of potential downtime
- [ ] Verify backup retention policy
- [ ] Verify KMS key permissions
- [ ] Verify application can handle reconnection
- [ ] Document current connection strings

---

## Migration Process (Per Cluster)

### Phase 1: Preparation

```bash
# Variables - update for each cluster
export OLD_CLUSTER="z360-staging-addonsstack-vg9z1x4cx7y2-dbdbcluster-d4asbozx0ixh"
export NEW_CLUSTER="z360-staging-db-encrypted"
export SNAPSHOT_ID="z360-staging-snapshot-$(date +%Y%m%d)"
export KMS_KEY_ALIAS="alias/aws/rds"  # or custom KMS key
export REGION="us-east-1"
```

### Phase 2: Create Snapshot

```bash
# Create snapshot of current cluster
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier $OLD_CLUSTER \
  --db-cluster-snapshot-identifier $SNAPSHOT_ID \
  --region $REGION

# Wait for snapshot completion
aws rds wait db-cluster-snapshot-available \
  --db-cluster-snapshot-identifier $SNAPSHOT_ID
```

### Phase 3: Copy Snapshot with Encryption

```bash
# Copy snapshot with encryption enabled
aws rds copy-db-cluster-snapshot \
  --source-db-cluster-snapshot-identifier $SNAPSHOT_ID \
  --target-db-cluster-snapshot-identifier "${SNAPSHOT_ID}-encrypted" \
  --kms-key-id $KMS_KEY_ALIAS \
  --region $REGION

# Wait for encrypted snapshot
aws rds wait db-cluster-snapshot-available \
  --db-cluster-snapshot-identifier "${SNAPSHOT_ID}-encrypted"
```

### Phase 4: Restore Encrypted Cluster

```bash
# Get original cluster configuration
aws rds describe-db-clusters \
  --db-cluster-identifier $OLD_CLUSTER \
  --query 'DBClusters[0].[VpcSecurityGroupIds,DBSubnetGroup,ServerlessV2ScalingConfiguration,Port]' \
  --output json

# Restore from encrypted snapshot
aws rds restore-db-cluster-from-snapshot \
  --db-cluster-identifier $NEW_CLUSTER \
  --snapshot-identifier "${SNAPSHOT_ID}-encrypted" \
  --engine aurora-postgresql \
  --engine-version 16.4 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=4 \
  --vpc-security-group-ids <SECURITY_GROUP_IDS> \
  --db-subnet-group-name <SUBNET_GROUP> \
  --region $REGION

# Wait for cluster to be available
aws rds wait db-cluster-available \
  --db-cluster-identifier $NEW_CLUSTER

# Create writer instance
aws rds create-db-instance \
  --db-instance-identifier "${NEW_CLUSTER}-instance" \
  --db-cluster-identifier $NEW_CLUSTER \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --region $REGION
```

### Phase 5: Update Application

1. **Update Secrets Manager secret** with new endpoint:
```bash
# Get new endpoint
aws rds describe-db-clusters \
  --db-cluster-identifier $NEW_CLUSTER \
  --query 'DBClusters[0].Endpoint'

# Update the secret (secret ARN from task definition)
# The DB_SECRET in task definitions references Secrets Manager
```

2. **Force ECS service redeployment** to pick up new connection:
```bash
aws ecs update-service \
  --cluster <CLUSTER_NAME> \
  --service <SERVICE_NAME> \
  --force-new-deployment
```

### Phase 6: Verification

```bash
# Verify encryption on new cluster
aws rds describe-db-clusters \
  --db-cluster-identifier $NEW_CLUSTER \
  --query 'DBClusters[0].StorageEncrypted'

# Verify application connectivity
# Run application health checks
```

### Phase 7: Cleanup (After Validation Period)

```bash
# Only after 24-48 hours of stable operation

# Delete old unencrypted cluster
aws rds delete-db-cluster \
  --db-cluster-identifier $OLD_CLUSTER \
  --skip-final-snapshot

# Delete old snapshots
aws rds delete-db-cluster-snapshot \
  --db-cluster-snapshot-identifier $SNAPSHOT_ID
```

---

## Detailed Steps per Cluster

### 1. z360-staging (Migrate First - Lowest Risk)

```bash
export OLD_CLUSTER="z360-staging-addonsstack-vg9z1x4cx7y2-dbdbcluster-d4asbozx0ixh"
export NEW_CLUSTER="z360-staging-db-encrypted"
```

**ECS Services to Update:**
- z360-staging-web
- z360-staging-queue
- z360-staging-reverb
- z360-staging-scheduler (if exists)

**Secret to Update:**
- `arn:aws:secretsmanager:us-east-1:108782097218:secret:dbAuroraSecret-46WKQDM9FLzK-L9dkom`

### 2. z360-agent-layer-staging

```bash
export OLD_CLUSTER="z360-agent-layer-staging-addonsst-agentdbdbcluster-1ownthodemtg"
export NEW_CLUSTER="z360-agent-layer-staging-db-encrypted"
```

**ECS Services to Update:**
- z360-agent-layer-staging-gateway
- z360-agent-layer-staging-aegra
- z360-agent-layer-staging-voice

### 3. z360-beta (Production-like - Extra Care)

```bash
export OLD_CLUSTER="z360-beta-addonsstack-n53yvqoutypc-dbdbcluster-1aamvltsjji0"
export NEW_CLUSTER="z360-beta-db-encrypted"
```

**ECS Services to Update:**
- z360-beta-web
- z360-beta-queue
- z360-beta-reverb

### 4. z360-agent-layer-beta

```bash
export OLD_CLUSTER="z360-agent-layer-beta-addonsstack-agentdbdbcluster-dvw0pqd83qiq"
export NEW_CLUSTER="z360-agent-layer-beta-db-encrypted"
```

**ECS Services to Update:**
- z360-agent-layer-beta-gateway
- z360-agent-layer-beta-aegra
- z360-agent-layer-beta-voice

---

## Rollback Procedure

If issues occur during migration:

1. **Stop ECS services** using new cluster
2. **Revert Secrets Manager** to old endpoint
3. **Force redeploy ECS services**
4. **Old cluster is still running** - no data loss

---

## Alternative: AWS DMS Migration

For larger databases or zero-downtime requirements, consider AWS Database Migration Service:

1. Create DMS replication instance
2. Create source endpoint (unencrypted cluster)
3. Create target endpoint (new encrypted cluster)
4. Run full load + CDC migration
5. Perform cutover when caught up

**Cost:** ~$0.25/hour for dms.t3.medium

---

## Post-Migration Verification

- [ ] Verify StorageEncrypted = true for new cluster
- [ ] Verify application health checks pass
- [ ] Verify no connection errors in logs
- [ ] Verify data integrity (row counts, checksums)
- [ ] Verify automated backups working
- [ ] Delete old cluster after 48-hour validation period

---

## Timeline Recommendation

| Day | Action |
|-----|--------|
| Day 1 | Migrate z360-staging (test process) |
| Day 2 | Migrate z360-agent-layer-staging |
| Day 3-4 | Validation period for staging |
| Day 5 | Migrate z360-beta (schedule maintenance) |
| Day 6 | Migrate z360-agent-layer-beta |
| Day 7-10 | Validation period for beta |
| Day 11 | Delete old unencrypted clusters |

---

## Contact

For questions or issues during migration, contact the infrastructure team.
