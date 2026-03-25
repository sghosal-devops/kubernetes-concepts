# nginx_scaling — Kubernetes Autoscaling Learning Lab

All manifests use `nginx:1.24` as the base workload.

## Files

| File | What it teaches |
|------|----------------|
| `00-deployment.yml` | Base deployment with proper resource requests (required for autoscaling) |
| `01-hpa.yml` | HPA v2 — scale replicas on CPU + memory with custom behavior |
| `02-vpa.yml` | VPA Auto mode — let K8s right-size your container resources |
| `03-vpa-recommendation-only.yml` | VPA Off mode — safe start, just get recommendations |
| `04-hpa-custom-metrics.yml` | HPA with custom metrics via Prometheus Adapter |

## Order to apply

```bash
# 1. Deploy the workload first
kubectl apply -f 00-deployment.yml

# 2. Apply HPA (needs metrics-server)
kubectl apply -f 01-hpa.yml

# 3. Watch HPA in action
kubectl get hpa -w

# 4. Apply VPA recommender (safe, no pod changes)
kubectl apply -f 03-vpa-recommendation-only.yml
kubectl describe vpa nginx-vpa-recommender
```

## Key concepts

- HPA = scales OUT (more pods), needs metrics-server
- VPA = scales UP (bigger pods), needs VPA controller
- Don't use HPA + VPA (Auto) on the same CPU/memory metric — they conflict
- Always set resource `requests` on containers, autoscalers depend on them

## Coming next
- KEDA (event-driven autoscaling)
- Cluster Autoscaler (node-level scaling)
