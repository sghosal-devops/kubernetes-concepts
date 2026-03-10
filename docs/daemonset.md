# Kubernetes DaemonSets

## What is a DaemonSet?

A DaemonSet ensures one pod runs on every eligible node.
In this repo, we use it for a real node-level logging use case: **Fluentd shipping logs toward Elasticsearch**.

---

## Fluentd + Elasticsearch DaemonSet

File: `manifests/daemonset.yml`

This manifest deploys:
- DaemonSet name: `fluentd-elasticsearch`
- Namespace: `kube-system`
- Container image: `quay.io/fluentd_elasticsearch/fluentd:v5.0.1`
- Host log mount: node `/var/log` mounted into pod `/var/log`

Why this pattern:
- DaemonSet gives one Fluentd pod per node.
- `hostPath: /var/log` lets Fluentd read node logs.
- Running in `kube-system` is common for cluster-level agents.

---

## Important Fields Explained

1. `selector.matchLabels` and `template.metadata.labels`
- Must match exactly.
- This is how DaemonSet identifies its pods.

2. `tolerations`
- Includes tolerations for control-plane and master taints.
- This allows Fluentd pods on control-plane nodes too.

3. `resources`
- Requests and limits are set to control memory/cpu usage for log agents.

4. `terminationGracePeriodSeconds: 30`
- Gives Fluentd time to flush buffered logs before stop.

5. `volumes.hostPath`
- Mounts node filesystem path `/var/log` into the container.
- Powerful and required for this use case, but use carefully.

---

## Commands to Run

```bash
# Apply
kubectl apply -f manifests/daemonset.yml

# Verify DaemonSet
kubectl get daemonset -n kube-system
kubectl describe daemonset fluentd-elasticsearch -n kube-system

# Verify pods
kubectl get pods -n kube-system -l name=fluentd-elasticsearch -o wide

# Logs
kubectl logs -n kube-system -l name=fluentd-elasticsearch --tail=50 --follow
```

Expected on a single-node kind cluster:
- `DESIRED = 1`
- `CURRENT = 1`
- `READY = 1`

---

## Useful Tests

### A) Self-healing
```bash
kubectl delete pod -n kube-system -l name=fluentd-elasticsearch
kubectl get pods -n kube-system -l name=fluentd-elasticsearch -w
```

### B) Rolling update image
```bash
kubectl set image daemonset/fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v5.0.2 -n kube-system
kubectl rollout status daemonset/fluentd-elasticsearch -n kube-system
```

### C) Rollback
```bash
kubectl rollout undo daemonset/fluentd-elasticsearch -n kube-system
kubectl rollout status daemonset/fluentd-elasticsearch -n kube-system
```

---

## Cleanup

```bash
kubectl delete -f manifests/daemonset.yml
```

---

## Quick Revision

- DaemonSet = one pod per node.
- Fluentd DaemonSet is used for node log collection.
- `hostPath /var/log` is the key enabler for log scraping.
- Tolerations allow running on control-plane nodes.
