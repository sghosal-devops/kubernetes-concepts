# Kubernetes Jobs

## What is a Job?

A Job is for one-off or finite work that must run to completion.
The Job controller keeps retrying failed Pods until success criteria are met or failure limits are reached.

Unlike Deployments and DaemonSets, Jobs are not for always-on services.

---

## Official Baseline We Will Use

We are using the Kubernetes docs pattern (pi example) in our namespace.

File: `manifests/job.yml`

- Name: `pi`
- Namespace: `nginx-ns`
- Image: `perl:5.34.0`
- Command: prints pi to 2000 digits
- `restartPolicy: Never`
- `backoffLimit: 4`

---

## Run and Verify

```bash
kubectl apply -f manifests/job.yml

kubectl get jobs -n nginx-ns
kubectl describe job pi -n nginx-ns

kubectl get pods -n nginx-ns --selector=batch.kubernetes.io/job-name=pi
kubectl logs job/pi -n nginx-ns
```

Expected:
- Job reaches `COMPLETIONS 1/1`
- Pod reaches `Completed`

---

## Key Concepts From Official Docs

1. Restart policy
- For Jobs, only `Never` or `OnFailure` are valid.

2. Retry behavior
- `backoffLimit` controls retry attempts before Job becomes `Failed`.

3. Parallel execution modes
- Non-parallel: usually one Pod to completion.
- Fixed completions: set `completions` and optional `parallelism`.
- Work queue: set `parallelism`, leave `completions` unset.

4. Completion modes
- `NonIndexed` (default): any successful Pod counts.
- `Indexed`: each index must complete; useful for static partitioned work.

5. Time-based stop
- `activeDeadlineSeconds` fails a Job if it runs too long.

6. Auto-cleanup
- `ttlSecondsAfterFinished` auto-deletes completed/failed Jobs and their Pods.

7. Suspension
- `spec.suspend: true` pauses Job execution; resume with `false`.

---

## Practical Commands

```bash
# Watch Job progress
kubectl get jobs -n nginx-ns -w

# List all pods for this job
kubectl get pods -n nginx-ns --selector=batch.kubernetes.io/job-name=pi

# Re-run same job name
kubectl delete job pi -n nginx-ns
kubectl apply -f manifests/job.yml

# One-off cloned run
kubectl create job --from=job/pi pi-manual-1 -n nginx-ns
```

---

## Cleanup

```bash
kubectl delete -f manifests/job.yml
```

---

## Quick Revision

- Job = finite task that must complete.
- `backoffLimit` = retry cap.
- `parallelism` = concurrent Pods.
- `completions` = required successful Pods.
- Use `ttlSecondsAfterFinished` in real clusters to avoid API clutter.

---

## Quick Interview Point (Important)

### Why use `suspend` in Jobs?

Common use cases:
- Pre-create Jobs but run later.
- Pause batch workloads during peak traffic windows.
- Let an external controller/queue decide when Jobs should start.
- Validate and debug Job specs safely before execution.

Kubernetes 1.24+ highlight:
- You can suspend and resume Jobs dynamically using `.spec.suspend` without deleting and recreating the Job.
- This is very useful for long-running batch pipelines.

---

## Practical Production Scenario (Interview Friendly)

Scenario:
- Your nightly ETL Job starts at 1:00 AM.
- At 1:20 AM, production API latency spikes because database load is high.
- Instead of deleting the Job, you suspend it to reduce pressure.

Commands:

```bash
# Suspend active Job
kubectl patch job/myjob -n nginx-ns --type=strategic --patch '{"spec":{"suspend":true}}'

# Resume when system is stable
kubectl patch job/myjob -n nginx-ns --type=strategic --patch '{"spec":{"suspend":false}}'
```

Why interviewers like this answer:
- Shows production awareness (cost/performance control).
- Shows you understand graceful workload control beyond create/delete.
- Demonstrates knowledge of modern Job lifecycle features.
