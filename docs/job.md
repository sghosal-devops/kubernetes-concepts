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
