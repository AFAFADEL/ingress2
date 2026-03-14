# Kubernetes Ingress Lab — Path-based & Host-based Routing

## 🚀 Part 1 — Setup & First Ingress (20 pts)

### 1. Enable Ingress Controller & Deploy Backend Apps

```bash
# Enable Ingress Controller (Minikube)
minikube addons enable ingress

# Verify controller pod is running
kubectl get pods -n ingress-nginx

# Verify IngressClass exists
kubectl get ingressclass

# Apply backend deployments and ClusterIP services
kubectl apply -f 01-deployments-and-services.yaml

# Verify deployments and services
kubectl get deploy,svc
```
Answers:

IngressClass name: nginx

Why ClusterIP not NodePort: Because Ingress routes traffic internally to services, NodePort is not required. ClusterIP is enough for internal routing.
2-
File: 02-ingress-basic-path.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```
# Apply Ingress
kubectl apply -f 02-ingress-basic-path.yaml

# Get Ingress IP
kubectl get ingress basic-ingress

# Describe routing
kubectl describe ingress basic-ingress

# Test paths (replace <ADDRESS> with Ingress IP)
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api
Answer:
```
Why one Ingress replaces 2 NodePort services: One Ingress can handle multiple paths and routes to internal ClusterIP services, so NodePort is not needed for external access
3-
File: 06-ingress-full-lab.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```
              # Delete previous Ingress
kubectl delete ingress basic-ingress

# Apply full lab Ingress
kubectl apply -f 06-ingress-full-lab.yaml

# Test all paths (replace <ADDRESS> with Ingress IP)
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/random
```
answer:
/random returns 404 Not Found

Routing table shows 3 rules for /, /api, /admin.
4-

File: 03-ingress-host-based.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
  - host: api.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
  - host: admin.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80

```
          # Delete previous Ingress
kubectl delete ingress lab-ingress

# Apply host-based Ingress
kubectl apply -f 03-ingress-host-based.yaml

# Describe Ingress
kubectl describe ingress host-ingres
# Test subdomains (replace <ADDRESS> with Ingress IP)
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local
curl --resolve "api.myapp.local:80:<ADDRESS>" http://api.myapp.local
curl --resolve "admin.myapp.local:80:<ADDRESS>" http://admin.myapp.local
curl --resolve "other.myapp.local:80:<ADDRESS>" http://other.myapp.local

Answers:

Host entries in describe output: 3

Unknown host: 404 Not Found

Difference path vs host-based: Path-based routes by URL path, host-based routes by domain/subdomain
5-
File: 05-ingress-path-types.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pathtype-demo
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /admin
        pathType: Exact
        backend:
          service:
            name: admin-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```
              # Delete host-ingress first
kubectl delete ingress host-ingress

# Apply PathType demo
kubectl apply -f 05-ingress-path-types.yaml
# Test
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api/users     # Prefix matches /api
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin         # Exact match
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin/settings # Exact fails
```
Answers:

/api/users → yes

/admin/settings → no (Exact only matches /admin)

Use Exact when you want only the specific path to match, not subpaths.
6-
File: 04-ingress-default-backend.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: webapp-svc
      port:
        number: 80
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80
```

# Delete previous Ingress
kubectl delete ingress pathtype-demo

# Apply Default Backend Ingress
kubectl apply -f 04-ingress-default-backend.yaml
```
# Test paths
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/api      # api-svc
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/admin    # admin-svc
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local/anything # webapp-svc (defaultBackend)
curl --resolve "myapp.local:80:<ADDRESS>" http://myapp.local           # webapp-svc (defaultBackend)
```
Answers:

/anything → webapp-svc

/ → webapp-svc

Use case for defaultBackend: Catch-all for unmatched requests (404 replacement, SPA routing, etc.)
