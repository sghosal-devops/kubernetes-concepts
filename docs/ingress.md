# Kubernetes Ingress

## What Ingress Solves

Ingress provides HTTP/HTTPS routing to services based on host/path rules.
Unlike Service, Ingress needs an Ingress Controller (for example ingress-nginx).

---

## File

- manifests/ingress/00-nginx-ingress.yml
- manifests/ingress/01-api-backend.yml

This routes:
- host: `nginx.local`
- path `/` -> service `nginx-svc:80`
- path `/api` -> service `api-svc:80`

---

## Prerequisite: Ingress Controller

Check controller:

```bash
kubectl get pods -n ingress-nginx
```

If not installed in kind, install ingress-nginx for kind (official deploy):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=180s
```

---

## Apply and Test

```bash
kubectl apply -f manifests/ingress/01-api-backend.yml
kubectl apply -f manifests/service/00-nginx-clusterip.yml
kubectl apply -f manifests/ingress/00-nginx-ingress.yml
kubectl get ingress -n nginx-ns
kubectl describe ingress nginx-ingress -n nginx-ns
```

Add host mapping on your machine:
- `127.0.0.1 nginx.local`

Then test:

```bash
curl -H "Host: nginx.local" http://127.0.0.1
curl -H "Host: nginx.local" http://127.0.0.1/api
```

In kind, you may need port-forward to ingress controller service:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
curl -H "Host: nginx.local" http://127.0.0.1:8080
curl -H "Host: nginx.local" http://127.0.0.1:8080/api
```

---

## Interview Points

- Ingress is L7 HTTP routing, Service is L4 networking abstraction.
- Ingress needs a controller; Ingress resource alone does nothing.
- One Ingress can route many hosts/paths to different services.
- TLS is typically terminated at Ingress.

---

## Cleanup

```bash
kubectl delete -f manifests/ingress/00-nginx-ingress.yml
```
