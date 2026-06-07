# Backup Strategies

## Introduction
Backup strategies are the foundation of data durability and disaster recovery. In an era where data is the most valuable business asset, the question is never whether to back up—it's how to design a backup architecture that meets Recovery Point Objectives (RPO) and Recovery Time Objectives (RTO) within budget constraints. The 3-2-1 rule (three copies, two different media, one off-site) has been a gold standard for decades, but cloud-native architectures have evolved backup strategies beyond simple file copies to continuous replication, point-in-time recovery, and immutable backups that survive ransomware attacks.

For solution architects, backup design is a multi-dimensional optimization problem: balancing cost, recovery speed, data loss tolerance, compliance retention requirements, and the operational complexity of managing backup lifecycles across databases, object storage, container volumes, and configuration state.

## Definition

**Backup** is the process of copying data from a primary source to a secondary location to enable recovery in the event of data loss, corruption, or disaster. Key metrics:

- **RPO (Recovery Point Objective)**: Maximum acceptable data loss measured in time. RPO of 1 hour means you can lose up to 1 hour of data. Zero RPO requires synchronous replication.
- **RTO (Recovery Time Objective)**: Maximum acceptable time to restore service. RTO of 4 hours means systems must be operational within 4 hours of a disaster declaration.

## Concept Explanation

### The 3-2-1 Rule (and Evolution)

```
3-2-1 Rule:
  ✓ 3 copies of data
  ✓ 2 different storage media
  ✓ 1 copy off-site

3-2-1-1-0 Rule (Modern / Ransomware-Resilient):
  ✓ 3 copies of data
  ✓ 2 different media
  ✓ 1 copy off-site
  ✓ 1 copy immutable/air-gapped
  ✓ 0 errors (verified recoverability)
```

### Backup Types

#### Full Backup
Complete copy of all data. Foundation for all other backup types.

```
Size: 100% of dataset
Time: Longest to create and restore
Frequency: Weekly (or longer)
```

#### Incremental Backup
Copies only data changed since the last backup (any type).

```
Size: Smallest, only changes
Time: Fastest to create; slowest to restore (must replay chain)
Frequency: Daily or hourly

Restore: Full + I1 + I2 + I3 + I4 + I5 (long chain = slow restore)
```

#### Differential Backup
Copies all changes since the last full backup.

```
Size: Grows over time (between full and incremental)
Time: Moderate to create; faster to restore than incremental
Frequency: Daily

Restore: Full + Latest Differential (only 2 pieces)
```

#### Continuous / CDP (Continuous Data Protection)
Every change is replicated in real-time to the backup target.

```
Provides: Point-in-time recovery to any second
Overhead: Higher (constant replication)
Use: Zero or near-zero RPO requirements
```

### Snapshot-Based Backups (Cloud-Native)

```python
# Conceptual snapshot lifecycle
class SnapshotManager:
    def __init__(self):
        self.policy = {
            'hourly':  {'retention': 24,   'schedule': '0 * * * *'},
            'daily':   {'retention': 7,    'schedule': '0 2 * * *'},
            'weekly':  {'retention': 4,    'schedule': '0 3 * * 0'},
            'monthly': {'retention': 12,   'schedule': '0 4 1 * *'},
            'yearly':  {'retention': 7,    'schedule': '0 5 1 1 *'}
        }
    
    def create_snapshot(self, volume_id, snapshot_type):
        snapshot = cloud_api.create_snapshot(
            VolumeId=volume_id,
            Description=f"{snapshot_type}-{datetime.utcnow().isoformat()}",
            Tags=[{'Key': 'Type', 'Value': snapshot_type}]
        )
        
        # Apply retention: delete snapshots beyond retention count
        snapshots = cloud_api.describe_snapshots(
            Filters=[{'Name': 'tag:Type', 'Values': [snapshot_type]}]
        )
        retention = self.policy[snapshot_type]['retention']
        if len(snapshots) > retention:
            oldest = sorted(snapshots, key=lambda s: s.start_time)[0]
            cloud_api.delete_snapshot(SnapshotId=oldest.id)
```

### Database Backup Strategies

#### PostgreSQL Continuous Archiving + PITR
```sql
-- Enable WAL archiving (postgresql.conf)
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://backups/wal/%f'

-- Full backup (pg_basebackup)
pg_basebackup -D /backup/current -Ft -z -P

-- Point-in-time recovery (recovery.conf)
restore_command = 'aws s3 cp s3://backups/wal/%f %p'
recovery_target_time = '2024-06-15 14:30:00'
```

#### MySQL Binary Log + mysqldump
```bash
# Full backup
mysqldump --single-transaction --all-databases --master-data=2 \
  | gzip > full_backup_$(date +%Y%m%d).sql.gz

# Point-in-time recovery
mysqlbinlog --start-datetime="2024-06-15 14:00:00" \
  --stop-datetime="2024-06-15 14:29:59" \
  binlog.000001 binlog.000002 | mysql
```

### Immutable Backups (Ransomware Defense)

```bash
# Write-Once-Read-Many (WORM) storage
aws s3api put-object --bucket backups \
  --key db-2024-06-15.sql.gz \
  --body db-2024-06-15.sql.gz \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date 2024-12-15

# Object cannot be deleted or modified until retention expires
# Even root/administrator cannot override COMPLIANCE mode
```

### Backup Verification

```python
def verify_backup(backup_id):
    """Automated backup verification prevents silent corruption"""
    results = {
        'integrity': False,
        'restore_tested': False,
        'size_match': False,
        'timestamp': datetime.utcnow()
    }
    
    # 1. Checksum verification
    results['integrity'] = (
        calculate_checksum(restore_backup(backup_id)) ==
        stored_checksum[backup_id]
    )
    
    # 2. Test restore to isolated environment
    test_env = spin_up_test_environment()
    try:
        restore_to_environment(backup_id, test_env)
        results['restore_tested'] = run_validation_queries(test_env)
    finally:
        tear_down_test_environment(test_env)
    
    # 3. Size sanity check
    results['size_match'] = (
        backup_metadata[backup_id].size > MIN_EXPECTED_SIZE
    )
    
    return results
```

### Backup Storage Tiers

| Tier | Access Latency | Cost | Best For |
|------|---------------|------|----------|
| Hot (immediate) | Milliseconds | $$$ | Last 7 days of backups |
| Warm | Minutes | $$ | 7-30 day retention |
| Cold | Hours | $ | 30-90 day compliance |
| Archive/Glacier | 12-48 hours | $ (cents/GB) | 7+ year regulatory retention |

## Layman's Explanation

### The Family Photo Album
Your family photos (business data) are irreplaceable:

- **No backup**: The house burns down. All photos gone forever. Business-ending event.
- **Full backup on external drive**: You copy all photos to an external hard drive once a week. If your laptop dies on Thursday, you lose Monday-Thursday photos (RPO = 4 days).
- **Cloud sync (continuous)**: Every photo you take is instantly uploaded to Google Photos. Laptop stolen at a coffee shop? All photos safe. (RPO = seconds)
- **3-2-1**: You have photos on your laptop, on an external drive at home, and on cloud storage. Laptop dies + house fire? Cloud still has them.
- **Immutable backup**: Ransomware encrypts your laptop AND your cloud backup. But the backup from last month is WORM-protected. You recover from there. Lose the last month, but keep years of history.
- **Verification**: You actually open the external drive once a month and check the photos aren't corrupted. Untested backups aren't backups—they're hopes.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **RPO/RTO-Driven Architecture**: The business defines acceptable data loss and downtime; the architect designs backup strategy to meet it. RPO of zero = synchronous replication (costly, impacts write latency). RPO of 24 hours = daily snapshots (cheap). Architecture follows business requirements.
- **Backup Scope Complexity**: Not just databases. Consider: infrastructure-as-code (Terraform state), container images, Kubernetes etcd, CI/CD configurations, DNS records, secrets, and configuration files. A complete restore requires ALL of these.
- **Restore Testing**: The most neglected aspect. Quarterly automated restore tests to isolated environments. Without testing, you don't know if your backups work until you desperately need them.
- **Cross-Region/Corss-Cloud**: Backups should not live in the same region or cloud provider as production. A region-wide outage or account compromise shouldn't also destroy backups.

### Business Impact
- **Ransomware Survival**: 66% of organizations were hit by ransomware in 2023. Immutable, air-gapped backups are the only guaranteed recovery path. Organizations with verified, immutable backups recover without paying ransom.
- **Compliance**: SOC 2 (CC7.1), PCI DSS (Req 9.5.1), HIPAA (164.308(a)(7)), GDPR (Art 32) all require backup and recovery capabilities. Specific retention periods are mandated.
- **Business Continuity**: A database corruption from a buggy deployment can be recovered in minutes from a recent snapshot, vs. days of manual reconstruction from logs.
- **RTO-Sensitive SLAs**: Customer contracts often specify RTO SLAs (99.9% uptime = 8.76 hours downtime/year). The backup/restore architecture must support the RTO committed to customers.

## On-Premises Examples

### rsync + cron (Simple File Backup)
```bash
# Daily incremental backup with rsync
rsync -avz --delete /data/app/ backup-server:/backups/app/daily/

# Weekly full with hard linking for deduplication
rsync -avz --delete --link-dest=/backups/app/latest \
  /data/app/ backup-server:/backups/app/$(date +%Y%m%d)/
rm /backups/app/latest
ln -s /backups/app/$(date +%Y%m%d) /backups/app/latest
```

### BorgBackup (Deduplicated, Encrypted)
```bash
# Initialize backup repository
borg init --encryption=repokey-blake2 backup-server:/backups/app

# Create backup (deduplicated, compressed, encrypted)
borg create --compression lz4 \
  backup-server:/backups/app::app-{now:%Y-%m-%d_%H:%M} \
  /data/app /etc/app

# List backups
borg list backup-server:/backups/app

# Restore
borg extract backup-server:/backups/app::app-2024-06-15_14:30
```

### Restic (Cloud-Native Backup)
```bash
# Backup to S3
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
restic -r s3:s3.amazonaws.com/my-backup-bucket init

# Create snapshot
restic -r s3:s3.amazonaws.com/my-backup-bucket backup /data/app

# Restore
restic -r s3:s3.amazonaws.com/my-backup-bucket restore latest --target /restore
```

## AWS Examples

### AWS Backup (Centralized)
```hcl
resource "aws_backup_vault" "main" {
  name = "app-backup-vault"
}

resource "aws_backup_plan" "app" {
  name = "app-backup-plan"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 * * ? *)"

    lifecycle {
      delete_after = 30  # delete after 30 days
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
    }
  }

  rule {
    rule_name         = "monthly-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 1 * ? *)"

    lifecycle {
      delete_after = 365
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
    }
  }

  advanced_backup_setting {
    resource_type = "EC2"
    backup_options = {
      WindowsVSS = "enabled"
    }
  }
}

resource "aws_backup_selection" "app" {
  iam_role_arn = aws_iam_role.backup.arn
  name         = "app-resources"
  plan_id      = aws_backup_plan.app.id

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "daily"
  }

  resources = [
    aws_db_instance.main.arn,
    aws_efs_file_system.app.arn,
  ]
}
```

### RDS Automated Backups + Snapshots
```hcl
resource "aws_db_instance" "main" {
  identifier = "orders-db"
  
  backup_retention_period = 30        # Automated daily backups, 30 days
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  # Snapshots: manually triggered, retained indefinitely
  final_snapshot_identifier = "orders-db-final-snapshot"
  skip_final_snapshot       = false
  copy_tags_to_snapshot     = true
  
  # Point-in-time recovery
  deletion_protection = true
}
```

### DynamoDB PITR
```hcl
resource "aws_dynamodb_table" "orders" {
  name     = "Orders"
  billing_mode = "PAY_PER_REQUEST"

  point_in_time_recovery {
    enabled = true  # Continuous backup to any second in last 35 days
  }
}
```

### S3 Cross-Region Replication + Object Lock
```hcl
resource "aws_s3_bucket" "backup" {
  bucket = "app-backups-primary"
}

resource "aws_s3_bucket_versioning" "backup" {
  bucket = aws_s3_bucket.backup.bucket
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_object_lock_configuration" "backup" {
  bucket = aws_s3_bucket.backup.bucket
  
  rule {
    default_retention {
      mode = "GOVERNANCE"  # Overridable with special permissions
      days = 90
    }
  }
}

resource "aws_s3_bucket_replication_configuration" "crr" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.backup.bucket

  rule {
    id     = "cross-region-replication"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
    }
    
    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

## GCP Examples

### Cloud Storage Object Versioning + Retention
```bash
# Create bucket with versioning and retention
gcloud storage buckets create gs://app-backups \
  --location=us-central1 \
  --uniform-bucket-level-access

gcloud storage buckets update gs://app-backups \
  --versioning

# Set retention policy (WORM)
gcloud storage buckets update gs://app-backups \
  --retention-period=90d

# Place object hold (prevent deletion)
gcloud storage objects update gs://app-backups/db-2024-06-15.sql.gz \
  --event-based-hold
```

### Cloud SQL Automated Backups
```bash
gcloud sql instances create orders-db \
  --backup-start-time=03:00 \
  --retained-backups-count=30 \
  --enable-point-in-time-recovery \
  --retained-transaction-log-days=7
```

### Filestore Backup (NFS for GKE)
```bash
gcloud filestore backups create app-backup-$(date +%Y%m%d) \
  --instance=app-filestore \
  --file-share=data \
  --region=us-central1
```

## Azure Examples

### Azure Backup (Centralized)
```bash
# Create Recovery Services Vault
az backup vault create \
  --name appBackupVault \
  --resource-group myResourceGroup \
  --location eastus

# Backup policy: daily at 2 AM, 30 days retention
az backup policy create \
  --name daily-policy \
  --vault-name appBackupVault \
  --resource-group myResourceGroup \
  --backup-management-type AzureIaasVM \
  --policy '{
    "schedulePolicy": {
      "scheduleRunFrequency": "Daily",
      "scheduleRunTimes": ["2024-01-01T02:00:00Z"]
    },
    "retentionPolicy": {
      "dailySchedule": {
        "retentionDuration": {"count": 30, "durationType": "Days"}
      }
    }
  }'

# Enable backup for VM
az backup protection enable-for-vm \
  --vm myVM \
  --policy-name daily-policy \
  --vault-name appBackupVault \
  --resource-group myResourceGroup
```

### Azure SQL Automated Backups
```sql
-- Azure SQL automatically:
-- Full backup: weekly
-- Differential backup: every 12 hours
-- Transaction log backup: every 5-10 minutes
-- 7-35 days retention
-- Long-term retention (LTR) up to 10 years in blob storage

-- Configuring LTR
ALTER DATABASE orders
SET LONG_TERM_RETENTION_POLICY (
    WEEKLY = 'P4W',    -- 4 weeks
    MONTHLY = 'P12M',  -- 12 months
    YEARLY = 'P7Y'     -- 7 years
);
```

## Summary Decision Matrix

| RPO Requirement | Backup Strategy | Cost |
|----------------|----------------|------|
| Zero (no data loss) | Synchronous replication + CDP | $$$$ |
| Seconds | Async replication + WAL streaming | $$$ |
| Minutes | Transaction log backup (5-10 min) + snapshots | $$ |
| Hours | Hourly incremental snapshots | $ |
| 24 hours | Daily automated snapshots/backups | $ |
| Days | Manual/scripted weekly full backups | Minimal |

Backup is insurance you hope to never use but must be able to rely on when needed. The architect's job is not just to configure backups but to design the entire backup lifecycle: what gets backed up, how often, where it's stored, who can access it, how long it's retained, and—critically—how to restore it. A backup strategy without tested restore procedures is not a strategy; it's a prayer.
