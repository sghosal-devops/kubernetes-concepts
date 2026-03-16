# Kubernetes Storage

## Why Storage Matters

Containers are ephemeral. If a container restarts, local filesystem changes can be lost.
Kubernetes volumes solve this by attaching storage into Pods.

Storage learning path:
1. Volume basics (this page)
2. PersistentVolume (PV)
3. PersistentVolumeClaim (PVC)
4. StorageClass (dynamic provisioning)

---

## Two Core Volume Types to Understand First

### 1) `emptyDir` (Pod-lifetime storage)
- Created when Pod starts.
- Shared across containers in the same Pod.
- Deleted when Pod is removed.
- Great for cache, scratch files, inter-container data exchange.

Manifest:
- `manifests/storage/00-storage-emptydir.yml`

Run:
```bash
kubectl apply -f manifests/storage/00-storage-emptydir.yml
kubectl get pod writer-reader-emptydir -n nginx-ns
kubectl logs writer-reader-emptydir -n nginx-ns -c reader -f
```

What to observe:
- `writer` writes lines to `/cache/data.log`
- `reader` reads same file from same `emptyDir` volume
- proves shared storage between containers in one Pod

---

### 2) `hostPath` (node filesystem mount)
- Mounts a path from the Kubernetes node into a Pod.
- Data can outlive the Pod (as long as it stays on same node and path).
- Useful for local dev/testing; avoid heavy production use unless carefully controlled.

Important for multi-node clusters:
- `hostPath` is node-local, so behavior is safest when you pin both sides.
- Pin PV to a node using PV `nodeAffinity`.
- Pin Pod to the same node using `nodeSelector` or node affinity.
- Without pinning, Pod rescheduling to another node can look like "missing data".

We will revisit this with proper examples in the Affinity/Taints section later.

Manifest:
- `manifests/storage/01-storage-hostpath.yml`

Run:
```bash
kubectl apply -f manifests/storage/01-storage-hostpath.yml
kubectl get pod hostpath-demo -n nginx-ns
kubectl exec -it hostpath-demo -n nginx-ns -- ls -la /node-data
kubectl exec -it hostpath-demo -n nginx-ns -- cat /node-data/demo.txt
```

---

## Quick Comparison

- `emptyDir`: Pod-scoped, deleted with Pod, very fast for temp data.
- `hostPath`: Node-scoped path, survives Pod recreation, node-dependent.
- PV/PVC: Cluster-managed persistence, portable and production-friendly.

---

## Next Step: Static PV + PVC Binding

Now that volume basics are clear, move to persistent storage via PV/PVC.

Manifests:
- `manifests/storage/02-pv-hostpath-static.yml`
- `manifests/storage/03-pvc-static.yml`
- `manifests/storage/04-pod-uses-pvc-static.yml`

What this teaches:
- PV is the storage supply (`1Gi`, `manual-hostpath` class).
- PVC is the app request (`512Mi`, same class).
- Pod mounts PVC, not PV directly.

Run order:

```bash
kubectl apply -f manifests/storage/02-pv-hostpath-static.yml
kubectl apply -f manifests/storage/03-pvc-static.yml

kubectl get pv pv-hostpath-static-01
kubectl get pvc pvc-static-01 -n nginx-ns
```

Expected:
- PV status: `Bound`
- PVC status: `Bound`

Attach to a Pod:

```bash
kubectl apply -f manifests/storage/04-pod-uses-pvc-static.yml
kubectl get pod pvc-consumer-static -n nginx-ns
kubectl exec -it pvc-consumer-static -n nginx-ns -- sh -c "ls -la /data; cat /data/app.log"
```

Persistence test:

```bash
kubectl delete pod pvc-consumer-static -n nginx-ns
kubectl apply -f manifests/storage/04-pod-uses-pvc-static.yml
kubectl exec -it pvc-consumer-static -n nginx-ns -- cat /data/app.log
```

If the previous log line is still there, PVC persistence is working.

---

## Deep Dive: PostgreSQL on Static PV/PVC

Use this when you want to validate real database persistence.

Manifests:
- `manifests/storage/postgres/05-db-secret.yml`
- `manifests/storage/postgres/06-pv-postgres-static.yml`
- `manifests/storage/postgres/07-pvc-postgres-static.yml`
- `manifests/storage/postgres/08-postgres-deployment-pvc.yml`
- `manifests/storage/postgres/09-postgres-service.yml`

Apply in order:

```bash
kubectl apply -f manifests/storage/postgres/05-db-secret.yml
kubectl apply -f manifests/storage/postgres/06-pv-postgres-static.yml
kubectl apply -f manifests/storage/postgres/07-pvc-postgres-static.yml
kubectl apply -f manifests/storage/postgres/08-postgres-deployment-pvc.yml
kubectl apply -f manifests/storage/postgres/09-postgres-service.yml
```

Verify bind and pod readiness:

```bash
kubectl get pv pv-postgres-static-01
kubectl get pvc pvc-postgres-static-01 -n nginx-ns
kubectl get pods -n nginx-ns -l app=postgres-pvc
kubectl get svc postgres-pvc-svc -n nginx-ns
```

Write test data:

```bash
kubectl exec -it -n nginx-ns deploy/postgres-pvc -- sh -c "psql -U appuser -d appdb -c \"create table if not exists notes(id serial primary key, msg text); insert into notes(msg) values ('hello-from-pvc'); select * from notes order by id;\""
```

Restart pod and verify persistence:

```bash
kubectl delete pod -n nginx-ns -l app=postgres-pvc
kubectl rollout status deploy/postgres-pvc -n nginx-ns
kubectl exec -it -n nginx-ns deploy/postgres-pvc -- sh -c "psql -U appuser -d appdb -c \"select * from notes order by id;\""
```

If row `hello-from-pvc` is still present after pod recreation, PV/PVC persistence is confirmed.

---

## Interview Points

- Kubernetes volumes are mounted into containers; they are not tied to container image layers.
- `emptyDir` lifecycle = Pod lifecycle.
- `hostPath` ties workload to node filesystem, so it reduces portability.
- Production persistence is typically PV/PVC + StorageClass, not raw `hostPath`.
- Pods consume PVCs; PVCs bind to PVs; StorageClass automates PV provisioning.

---

## Cleanup

```bash
kubectl delete -f manifests/storage/00-storage-emptydir.yml
kubectl delete -f manifests/storage/01-storage-hostpath.yml
kubectl delete -f manifests/storage/04-pod-uses-pvc-static.yml
kubectl delete -f manifests/storage/03-pvc-static.yml
kubectl delete -f manifests/storage/02-pv-hostpath-static.yml
kubectl delete -f manifests/storage/postgres/09-postgres-service.yml
kubectl delete -f manifests/storage/postgres/08-postgres-deployment-pvc.yml
kubectl delete -f manifests/storage/postgres/07-pvc-postgres-static.yml
kubectl delete -f manifests/storage/postgres/06-pv-postgres-static.yml
kubectl delete -f manifests/storage/postgres/05-db-secret.yml
```
