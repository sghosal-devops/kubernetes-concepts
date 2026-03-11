# Kubernetes CronJobs

## What is a CronJob?

A CronJob creates Jobs on a repeating schedule.
Think of one CronJob as one line in a Unix crontab.

---

## Files in this repo

- `manifests/cronjob/00-cronjob-database-backup-simple.yml`
- `manifests/cronjob/01-cronjob-database-backup-advanced.yml`
- `manifests/cronjob/02-cronjob-log-archive-advanced.yml`

---

## Key fields from official docs

- `schedule`: required cron expression.
- `jobTemplate`: required Job spec template.
- `concurrencyPolicy`: `Allow` (default), `Forbid`, or `Replace`.
- `startingDeadlineSeconds`: max lateness allowed for missed schedules.
- `suspend`: pause scheduling of new Jobs.
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`: retained history size.
- `timeZone` (v1.27+): interpret schedule in a specific TZ, for example `Etc/UTC`.

---

## Run order (recommended)

### 1) Simple database backup (foundational)

```bash
kubectl apply -f manifests/cronjob/00-cronjob-database-backup-simple.yml
kubectl get cronjob database-backup-simple -n nginx-ns
kubectl describe cronjob database-backup-simple -n nginx-ns
```

What this includes:
- easy-to-read shell workflow
- `concurrencyPolicy: Forbid`
- basic retries/history behavior

### 2) Database backup (advanced, PostgreSQL tooling)

```bash
kubectl apply -f manifests/cronjob/01-cronjob-database-backup-advanced.yml
kubectl get cronjob database-backup -n nginx-ns
kubectl describe cronjob database-backup -n nginx-ns
```

What this includes:
- daily schedule (`0 2 * * *`)
- `concurrencyPolicy: Forbid`
- `startingDeadlineSeconds: 300`
- PostgreSQL tooling via `postgres:16-alpine`
- backup checksum generation and retention cleanup
- history retention controls + TTL

Manual trigger for immediate testing:

```bash
kubectl create job --from=cronjob/database-backup db-backup-manual-1 -n nginx-ns
kubectl logs -n nginx-ns -l job-name=db-backup-manual-1 --tail=100
```

### 3) Log archive (advanced, Python pipeline)

```bash
kubectl apply -f manifests/cronjob/02-cronjob-log-archive-advanced.yml
kubectl get cronjob log-archive -n nginx-ns
kubectl describe cronjob log-archive -n nginx-ns
```

What this includes:
- daily schedule (`30 1 * * *`)
- `timeZone: Etc/UTC`
- `concurrencyPolicy: Replace`
- multi-stage pipeline (`initContainer` + Python archiver)
- tar.gz + sha256 + summary report generation
- Job-level `ttlSecondsAfterFinished` for cleanup

Manual trigger for immediate testing:

```bash
kubectl create job --from=cronjob/log-archive log-archive-manual-1 -n nginx-ns
kubectl logs -n nginx-ns -l job-name=log-archive-manual-1 --tail=100
```

---

## Quick interview points

- CronJob creates Jobs; Jobs create Pods.
- Cron schedule is approximate, workloads must be idempotent.
- `Forbid` avoids overlap, `Replace` kills old and starts new.
- `suspend` pauses future runs without deleting the object.
- Use history limits or Job TTL to reduce API clutter.
- Use `initContainers` to prepare data before main batch processing.
- Prefer workload-specific images (for example, postgres/python) over generic shells for production tasks.

---

## Cleanup

```bash
kubectl delete -f manifests/cronjob/00-cronjob-database-backup-simple.yml
kubectl delete -f manifests/cronjob/01-cronjob-database-backup-advanced.yml
kubectl delete -f manifests/cronjob/02-cronjob-log-archive-advanced.yml
```
