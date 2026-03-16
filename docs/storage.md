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

## Interview Points

- Kubernetes volumes are mounted into containers; they are not tied to container image layers.
- `emptyDir` lifecycle = Pod lifecycle.
- `hostPath` ties workload to node filesystem, so it reduces portability.
- Production persistence is typically PV/PVC + StorageClass, not raw `hostPath`.

---

## Cleanup

```bash
kubectl delete -f manifests/storage/00-storage-emptydir.yml
kubectl delete -f manifests/storage/01-storage-hostpath.yml
```
