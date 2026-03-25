# Kubernetes Services

## Why Service is Needed

Pods are ephemeral and their IPs change.
A Service gives a stable virtual IP (ClusterIP) and DNS name for a set of pods.

---

## Files

- manifests/service/00-nginx-clusterip.yml
- manifests/service/01-nginx-nodeport.yml

---

## 1) ClusterIP Service

File:
- manifests/service/00-nginx-clusterip.yml

Purpose:
- Internal-only access inside the cluster.
- Stable DNS: `nginx-svc.nginx-ns.svc.cluster.local`.

Apply and test:

```bash
kubectl apply -f manifests/service/00-nginx-clusterip.yml
kubectl get svc -n nginx-ns
kubectl describe svc nginx-svc -n nginx-ns

kubectl run curl-test --rm -it --image=curlimages/curl:8.9.1 -n nginx-ns -- sh
# inside pod:
# curl http://nginx-svc
```

---

## 2) NodePort Service

File:
- manifests/service/01-nginx-nodeport.yml

Purpose:
- Expose service through `<NodeIP>:30080`.
- Useful for quick local/dev access.

Apply and test:

```bash
kubectl apply -f manifests/service/01-nginx-nodeport.yml
kubectl get svc nginx-nodeport-svc -n nginx-ns
```

If using kind, check node container IP and curl from host or inside node container.

---

## Interview Points

- Service selects pods using labels.
- Service load balances across matching pods.
- ClusterIP is default and internal.
- NodePort opens fixed port on every node.
- Service does not create pods; it fronts existing pods.

---

## Cleanup

```bash
kubectl delete -f manifests/service/01-nginx-nodeport.yml
kubectl delete -f manifests/service/00-nginx-clusterip.yml
```
