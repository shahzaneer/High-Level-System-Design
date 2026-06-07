# Jobs & CronJobs

Jobs run a Pod to completion (batch processing). A Job creates one or more Pods and ensures a specified number of them successfully terminate. CronJobs extend Jobs with a schedule (cron syntax), creating Jobs on a recurring basis.

Use Jobs for: database migrations, data processing, report generation, backup scripts. Use CronJobs for: scheduled backups, periodic cleanup, certificate renewal, nightly batch processing.

## Imperative (kubectl)

```bash
# Create a Job
kubectl create job db-migration \
  --image=migration-tool:v1 \
  -- python manage.py migrate

# Create a CronJob
kubectl create cronjob nightly-backup \
  --image=backup-tool:v1 \
  --schedule="0 2 * * *" \
  -- /backup.sh

# View
kubectl get jobs
kubectl get cronjobs
kubectl logs job/db-migration
```

## Declarative (YAML)

```yaml
# Job: runs to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  parallelism: 3          # Run 3 pods in parallel
  completions: 6          # Need 6 successful completions total
  backoffLimit: 4         # Retry up to 4 times on failure
  activeDeadlineSeconds: 300  # Kill job if not done in 5 min
  ttlSecondsAfterFinished: 600 # Auto-delete after 10 min
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: data-processor:v2
        command: ["python", "process.py"]
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
---
# CronJob: scheduled job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"           # Every day at 2 AM
  concurrencyPolicy: Forbid       # Don't start new if previous still running
  successfulJobsHistoryLimit: 7   # Keep last 7 successful jobs
  failedJobsHistoryLimit: 3       # Keep last 3 failed jobs
  startingDeadlineSeconds: 300    # Must start within 5 min of schedule
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | gzip > /backup/db-$(date +%Y%m%d-%H%M).sql.gz
            env:
            - name: DB_HOST
              valueFrom: { secretKeyRef: { name: db-credentials, key: host } }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: db-credentials, key: password } }
```

## Cron Schedule Syntax

```
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of week (0-6, Sunday=0)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)

Examples:
0 2 * * *     Every day at 2:00 AM
*/15 * * * *  Every 15 minutes
0 9 * * 1-5   Weekdays at 9:00 AM
0 0 1 * *     First day of every month at midnight
```

## Concurrency Policies

| Policy | Behavior |
|--------|----------|
| Allow (default) | Multiple jobs can run concurrently |
| Forbid | Skip new job if previous still running |
| Replace | Kill running job, start new one |

## Job Patterns

```yaml
# Parallel with work queue (each pod processes items from queue)
spec:
  parallelism: 10
  completions: 100  # Process 100 items
  
# Single pod (migration, setup)
spec:
  parallelism: 1
  completions: 1
```

## Imperative vs Declarative

Imperative `kubectl create job` is excellent for one-off operations and debugging. Declarative YAML is required for CronJobs (complex schedules, concurrency policy, history limits) and for Jobs in CI/CD pipelines.
