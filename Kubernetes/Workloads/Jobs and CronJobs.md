---
tags:
  - kubernetes
  - kubernetes/workloads
topic: Workloads
---

# Jobs and CronJobs

## Job Basics

A Job creates one or more Pods and ensures that a specified number of them **successfully terminate** (exit code 0). Unlike Deployments, which keep Pods running indefinitely, Jobs are for **run-to-completion** workloads: batch processing, data migrations, report generation, backups, etc.

```
Job: generate-report
┌────────────────────────────────────┐
│  Creates Pod → Pod runs → Pod     │
│  completes with exit 0 → Done     │
│                                    │
│  If Pod fails → retry (up to      │
│  backoffLimit) → eventually       │
│  succeed or mark Job as Failed    │
└────────────────────────────────────┘
```

When a Job completes, the Pods are **not deleted** — they remain in `Completed` status so you can check logs. The Job object also persists until you delete it (or a TTL controller cleans it up).

## Job YAML Manifest

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1                    # number of successful completions required
  parallelism: 1                    # number of Pods running concurrently
  backoffLimit: 4                   # max retries before marking the Job as Failed
  activeDeadlineSeconds: 600        # max total runtime — kills the Job after this
  ttlSecondsAfterFinished: 3600     # auto-delete Job 1 hour after completion
  template:
    metadata:
      labels:
        job: db-migration
    spec:
      restartPolicy: OnFailure     # OnFailure or Never (not Always)
      containers:
        - name: migrate
          image: my-app:1.0.0
          command: ["./migrate", "--up"]
          env:
            - name: DB_HOST
              value: "db-service"
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

## Parallelism and Completions

These two fields control how Pods run:

| Field | Default | Meaning |
|---|---|---|
| `completions` | 1 | Total number of Pods that must successfully complete |
| `parallelism` | 1 | Maximum number of Pods running at the same time |

Examples:

| completions | parallelism | Behavior |
|---|---|---|
| 1 | 1 | Single Pod runs, Job done when it succeeds (default) |
| 5 | 1 | 5 Pods run sequentially, one at a time |
| 5 | 3 | Up to 3 Pods run concurrently until 5 total have succeeded |
| nil | 3 | Work queue mode: 3 Pods run in parallel, Job completes when any one succeeds |

## backoffLimit and activeDeadlineSeconds

### backoffLimit

Controls how many times a Pod can **fail and be retried** before the Job is marked as Failed. Default is 6. The retry delay uses exponential back-off (10s, 20s, 40s, ...) capped at 6 minutes.

```yaml
spec:
  backoffLimit: 4     # allow 4 retries, then fail the Job
```

The behavior depends on `restartPolicy`:

| restartPolicy | On failure |
|---|---|
| `OnFailure` | The same Pod is restarted in-place. Each restart counts toward `backoffLimit`. |
| `Never` | The Pod is left in Failed state. A new Pod is created. Each new Pod counts toward `backoffLimit`. |

### activeDeadlineSeconds

A hard time limit for the entire Job. Once exceeded, all running Pods are terminated and the Job is marked as Failed. This takes precedence over `backoffLimit`.

```yaml
spec:
  activeDeadlineSeconds: 300    # kill everything after 5 minutes
```

## Job Patterns

### Single Job

One Pod, one completion. The simplest pattern.

```yaml
spec:
  completions: 1
  parallelism: 1
```

### Parallel with Fixed Completion Count

Run a fixed number of work items, multiple at a time.

```yaml
spec:
  completions: 10     # process 10 items total
  parallelism: 3      # 3 at a time
```

Each Pod processes one work item. The Job tracks how many have succeeded and creates new Pods until `completions` is reached.

### Parallel with Work Queue

Multiple Pods consume from a shared work queue. The Job completes when a Pod exits successfully (signaling the queue is empty).

```yaml
spec:
  # completions is unset (or null) — this activates work queue mode
  parallelism: 5
```

Each Pod pulls tasks from an external queue (RabbitMQ, Redis, SQS, etc.). When a Pod finds the queue empty, it exits 0. The Job considers this a signal that all work is done.

### Indexed Job

Each Pod gets a unique index (0, 1, 2, ...) via the `JOB_COMPLETION_INDEX` environment variable. Useful when each Pod needs to process a different partition of data.

```yaml
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed     # each pod gets JOB_COMPLETION_INDEX (0-4)
```

## CronJob Basics

A CronJob creates **Jobs on a schedule**, like cron on Linux. It manages the lifecycle of Jobs and their cleanup.

```
CronJob (schedule: "0 2 * * *")
│
├── 2:00 AM → creates Job → creates Pod → runs → completes
├── 2:00 AM → creates Job → creates Pod → runs → completes
└── 2:00 AM → creates Job → creates Pod → runs → completes
```

### Schedule Syntax

Uses standard cron format with five fields:

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * *
```

| Expression | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour on the hour |
| `0 2 * * *` | Daily at 2:00 AM |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 9 1 * *` | 9:00 AM on the 1st of every month |
| `30 8 * * 1-5` | 8:30 AM on weekdays |

## CronJob YAML Manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"                 # daily at 2:00 AM UTC
  timeZone: "America/New_York"          # optional, defaults to kube-controller-manager timezone
  concurrencyPolicy: Forbid             # Allow | Forbid | Replace
  startingDeadlineSeconds: 200          # max seconds late a Job can start
  suspend: false                        # set to true to pause scheduling
  successfulJobsHistoryLimit: 3         # keep last 3 successful Jobs
  failedJobsHistoryLimit: 1             # keep last 1 failed Job
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 86400    # clean up Job after 24 hours
      template:
        metadata:
          labels:
            job: nightly-backup
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0.0
              command: ["./backup.sh"]
              env:
                - name: BACKUP_BUCKET
                  value: "s3://my-backups"
              resources:
                requests:
                  cpu: "500m"
                  memory: "256Mi"
                limits:
                  cpu: "1"
                  memory: "512Mi"
```

## Concurrency Policies

Controls what happens when a new scheduled run triggers while a previous Job is still running:

| Policy | Behavior |
|---|---|
| **Allow** (default) | Multiple Jobs can run concurrently. No protection against overlap. |
| **Forbid** | Skip the new run if the previous Job is still active. The missed run is counted against `startingDeadlineSeconds`. |
| **Replace** | Kill the currently running Job and start a new one. |

```yaml
spec:
  concurrencyPolicy: Forbid    # skip if previous job is still running
```

Choose based on your workload:

- **Allow** — independent jobs that can safely overlap (e.g., health checks)
- **Forbid** — jobs that must not run concurrently (e.g., database backups)
- **Replace** — the latest run is always more important than the previous one

## startingDeadlineSeconds

If the CronJob controller misses a scheduled time (e.g., the controller was down), it can still start the Job as long as the delay is within `startingDeadlineSeconds`. If the deadline is exceeded, the run is skipped and counted as a miss.

```yaml
spec:
  startingDeadlineSeconds: 200   # allow up to 200 seconds late
```

If more than **100 missed schedules** accumulate (and `startingDeadlineSeconds` is set), the CronJob stops scheduling and logs an error. This prevents a flood of Jobs after a long outage.

Without `startingDeadlineSeconds` set, the CronJob controller counts all misses since `status.lastScheduleTime`.

## History Limits

Control how many completed Jobs are kept for inspection:

```yaml
spec:
  successfulJobsHistoryLimit: 3   # keep 3 successful Jobs (default: 3)
  failedJobsHistoryLimit: 1       # keep 1 failed Job (default: 1)
```

Setting these to `0` deletes Jobs immediately after completion. Old Jobs (and their Pods) consume resources, so keep these values low. The `ttlSecondsAfterFinished` field on the Job spec provides more granular control.

## Suspend Field

Temporarily pause a CronJob without deleting it. No new Jobs are created while suspended, but already-running Jobs continue:

```yaml
spec:
  suspend: true    # pause — no new Jobs will be created
```

Toggle via kubectl:

```bash
# Pause
kubectl patch cronjob nightly-backup -p '{"spec": {"suspend": true}}'

# Resume
kubectl patch cronjob nightly-backup -p '{"spec": {"suspend": false}}'
```

Suspended CronJobs still count missed schedules. If many runs are missed while suspended, be aware of the 100-miss limit when resuming if `startingDeadlineSeconds` is set.
