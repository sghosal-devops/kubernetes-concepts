# Kubernetes DaemonSets

## What is a DaemonSet?

A DaemonSet ensures **exactly one pod runs on every eligible node**.
If a new node joins, DaemonSet automatically schedules a pod there.
If a node is removed, the pod on that node is removed too.

Use cases:
- Log collectors (Fluent Bit, Filebeat)
- Node monitoring agents (Node Exporter)
- Security agents
- CNI or node-level networking components

---

## DaemonSet vs Deployment

- `Deployment`: You choose replica count (for app workloads).
- `DaemonSet`: Kubernetes chooses count = number of eligible nodes.

In a kind cluster with 1 node, a DaemonSet normally runs 1 pod.

---

## Manifest Used

File: `manifests/daemonset.yml`

- Name: `node-agent-ds`
- Namespace: `nginx-ns`
- Image: `busybox:latest`
- Behavior: prints a heartbeat log every 15 seconds

---

## Commands to Run

```bash
# 1. Apply daemonset
kubectl apply -f manifests/daemonset.yml

# 2. Check daemonset status
kubectl get daemonset -n nginx-ns
kubectl describe daemonset node-agent-ds -n nginx-ns

# 3. Check pods created by daemonset
kubectl get pods -n nginx-ns -l app=node-agent -o wide

# 4. Watch logs from daemonset pod
kubectl logs -n nginx-ns -l app=node-agent --tail=20 --follow
```

Expected in kind single-node:
- `DESIRED = 1`
- `CURRENT = 1`
- `READY = 1`

---

## Useful Tests

### A) Delete daemonset pod (self-healing)
```bash
kubectl delete pod -n nginx-ns -l app=node-agent
kubectl get pods -n nginx-ns -l app=node-agent -w
```
A new pod should be recreated automatically.

### B) Update image (rolling update style)
```bash
kubectl set image daemonset/node-agent-ds node-agent=busybox:1.36 -n nginx-ns
kubectl rollout status daemonset/node-agent-ds -n nginx-ns
kubectl get pods -n nginx-ns -l app=node-agent
```

### C) Rollback
```bash
kubectl rollout undo daemonset/node-agent-ds -n nginx-ns
kubectl rollout status daemonset/node-agent-ds -n nginx-ns
```

---

## Cleanup

```bash
kubectl delete -f manifests/daemonset.yml
```

---

## Quick Revision

- DaemonSet = one pod per node.
- Great for node-level agents.
- Pod count scales with nodes, not replicas.
- Supports rollout, status, and rollback commands similar to Deployments.
