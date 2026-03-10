# Kubernetes Deployments

## What is a Deployment?

A Deployment is a higher-level abstraction that manages **ReplicaSets**, which in turn manage **Pods**.
You describe the desired state in the Deployment spec, and the Deployment Controller continuously works to match it.

```
Deployment  →  ReplicaSet  →  Pods
```

You never create ReplicaSets or Pods directly when using Deployments — the Deployment owns them.

---

## Manifest Breakdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx-ns          # scope to a namespace; omit = default namespace
  labels:
    app: nginx                 # labels on the Deployment object itself
spec:
  replicas: 3                  # desired number of pods

  strategy:
    type: RollingUpdate        # default; alternative: Recreate
    rollingUpdate:
      maxSurge: 1              # max extra pods allowed during rollout
      maxUnavailable: 1        # max pods that can be down during rollout

  revisionHistoryLimit: 3      # how many old ReplicaSets to keep (default: 10)

  selector:
    matchLabels:
      app: nginx               # must match template.metadata.labels

  template:                    # Pod template — everything below here is a Pod spec
    metadata:
      labels:
        app: nginx             # must match selector.matchLabels
    spec:
      containers:
        - name: nginx
          image: nginx:1.25    # always pin a version, never use :latest in production
          ports:
            - containerPort: 80
          resources:
            requests:          # minimum guaranteed resources
              memory: "64Mi"
              cpu: "100m"      # 100 millicores = 0.1 CPU
            limits:            # hard cap; pod killed if exceeded
              memory: "128Mi"
              cpu: "500m"
```

---

## Namespace

A Namespace is a virtual cluster inside the physical cluster. It isolates resources.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-ns
  labels:
    env: learning
```

```bash
kubectl apply -f nginx/namespace.yml
kubectl get namespaces
```

> All subsequent commands need `-n <namespace>` flag, or set a default context namespace.

---

## Rolling Update Strategy

### How it works
1. You change the PodTemplateSpec (e.g., image version).
2. Kubernetes creates a **new ReplicaSet** and starts scaling it up.
3. Simultaneously it scales down the **old ReplicaSet**.
4. `maxSurge` / `maxUnavailable` control the pace.

### maxSurge vs maxUnavailable

| Field            | Meaning                                             |
|------------------|-----------------------------------------------------|
| `maxSurge: 1`    | At most 1 extra pod can exist above desired count   |
| `maxUnavailable: 1` | At most 1 pod can be unavailable at any time    |

With `replicas: 3, maxSurge: 1, maxUnavailable: 1`:
- Max pods during rollout = 4 (3 + 1 surge)
- Min pods available during rollout = 2 (3 - 1)

### Trigger a rolling update
```bash
# Option 1: edit manifest file → apply (preferred / GitOps way)
# edit deployment.yml → change image: nginx:1.26
kubectl apply -f nginx/deployment.yml

# Option 2: imperative (quick testing only, manifest goes out of sync)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26 -n nginx-ns
```

### Watch rollout live
```bash
kubectl rollout status deployment/nginx-deployment -n nginx-ns
kubectl get pods -n nginx-ns -w
```

---

## Rollback

Every change to PodTemplateSpec creates a new **revision**. Rollback reactivates an old ReplicaSet.

```bash
# View history
kubectl rollout history deployment/nginx-deployment -n nginx-ns

# Undo last rollout (go to previous revision)
kubectl rollout undo deployment/nginx-deployment -n nginx-ns

# Undo to a specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1 -n nginx-ns
```

> **Key insight:** The old ReplicaSet is never deleted during rollout (until revisionHistoryLimit is hit).
> Rollback is instant because the old RS is just scaled back up.

---

## Scaling

Scaling only changes replica count — does **not** create a new ReplicaSet or revision.

```bash
kubectl scale deployment/nginx-deployment --replicas=6 -n nginx-ns
```

Or declaratively in the manifest:
```yaml
spec:
  replicas: 6
```
then `kubectl apply`.

---

## Pause & Resume

Use when you want to batch multiple changes into a single rollout:

```bash
kubectl rollout pause deployment/nginx-deployment -n nginx-ns

# Make multiple changes (image, env vars, resource limits, etc.)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26 -n nginx-ns
kubectl set resources deployment/nginx-deployment -c nginx --limits=memory=256Mi -n nginx-ns

# Apply all changes at once
kubectl rollout resume deployment/nginx-deployment -n nginx-ns
```

---

## Stuck Rollout Detection

If a new pod can't start (bad image, OOM, missing secret, etc.), the rollout hangs.

```bash
kubectl rollout status deployment/nginx-deployment -n nginx-ns
# exits non-zero if stuck — pipe this in CI/CD to fail the pipeline

kubectl describe pod <pod-name> -n nginx-ns   # see exact error reason
```

---

## Essential Commands

```bash
# Apply / create
kubectl apply -f nginx/deployment.yml

# List
kubectl get deployments -n nginx-ns
kubectl get replicasets -n nginx-ns       # see RS created by deployment
kubectl get pods -n nginx-ns

# Describe (detailed info)
kubectl describe deployment nginx-deployment -n nginx-ns
kubectl describe pod <pod-name> -n nginx-ns

# Logs
kubectl logs <pod-name> -n nginx-ns

# Check image on running pod
kubectl describe pod <pod-name> -n nginx-ns | grep image

# Rollout commands
kubectl rollout status  deployment/nginx-deployment -n nginx-ns
kubectl rollout history deployment/nginx-deployment -n nginx-ns
kubectl rollout undo    deployment/nginx-deployment -n nginx-ns
kubectl rollout pause   deployment/nginx-deployment -n nginx-ns
kubectl rollout resume  deployment/nginx-deployment -n nginx-ns

# Delete
kubectl delete deployment nginx-deployment -n nginx-ns
kubectl delete namespace nginx-ns          # deletes everything inside it
```

---

## Manifest File vs Live Cluster State

| | Manifest File (your .yml) | Live Cluster (etcd) |
|---|---|---|
| Changed by | `kubectl apply` | `kubectl set image`, `kubectl rollout undo`, etc. |
| Source of truth | Should be (GitOps) | Is always current |
| After `rollout undo` | Still shows old version | Shows rolled-back version |

**Rule:** Always update your manifest file first, then apply. Never rely on imperative commands as the source of truth.

---

## Deployment vs ReplicaSet vs StatefulSet (Quick Reference)

| | Deployment | ReplicaSet | StatefulSet |
|---|---|---|---|
| Use for | Stateless apps | Rarely used directly | Stateful apps (DBs, queues) |
| Rolling update | Yes | No | Yes (ordered) |
| Pod identity | Random names | Random names | Stable names (pod-0, pod-1) |
| Persistent storage per pod | No | No | Yes |
| Manages | ReplicaSet | Pods | Pods |
