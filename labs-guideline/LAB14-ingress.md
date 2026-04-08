# Lab 14: Ingress - HTTP/HTTPS Routing

## Objective
Learn how to use Ingress to expose HTTP/HTTPS services with host and path-based routing.

---

## What is Ingress?

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster.

**Key features:**
- Host-based routing (e.g., api.example.com vs web.example.com)
- Path-based routing (e.g., /api → service A, /web → service B)
- TLS termination (HTTPS)
- Load balancing
- Name-based virtual hosting

**Requirements:**
- Ingress controller installed (nginx, traefik, etc.)
- Ingress resource definition

---

## Step 1: Prerequisites Check

```bash
# Check if Ingress controller is installed
kubectl get pods -n ingress-nginx
kubectl get pods -n kube-system | grep ingress

# Or check all namespaces
kubectl get pods --all-namespaces | grep -i ingress
```

For K3s, you may need to install nginx-ingress:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml
```

---

## Step 2: Create Deployment and Service

```bash
# Create sample deployment
kubectl apply -f ../k8s-training/05-deployment.yaml

# Verify
kubectl get deployment nginx-deployment -n training
```

---

## Step 3: Create Basic Ingress

Create file `14-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: training
  labels:
    lesson: "14"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: nginx.training.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-clusterip
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 14-ingress.yaml
```

---

## Step 4: Verify Ingress

```bash
# List Ingress resources
kubectl get ingress -n training
kubectl get ing -n training

# Describe Ingress
kubectl describe ingress nginx-ingress -n training
```

---

## Step 5: Get Ingress IP/Hostname

```bash
# Get Ingress address
kubectl get ingress -n training -o wide

# For bare metal/minikube, may need to add to /etc/hosts
# Find ingress controller IP
kubectl get svc -n ingress-nginx
```

---

## Step 6: Test Ingress

```bash
# Add to /etc/hosts (requires sudo)
echo "<INGRESS-IP> nginx.training.local" | sudo tee -a /etc/hosts

# Test access
curl http://nginx.training.local
```

---

## Step 7: Multiple Paths

Update Ingress for path-based routing:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
  namespace: training
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.training.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: default-service
                port:
                  number: 80
```

---

## Step 8: TLS/HTTPS

Create TLS secret and update Ingress:

```yaml
# First, create TLS secret
kubectl create secret tls nginx-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem \
  -n training

# Or create self-signed for testing
kubectl create secret tls nginx-tls \
  --cert=/dev/stdin \
  --key=/dev/stdin \
  -n training <<EOF
-----BEGIN CERTIFICATE-----
<your-cert>
-----END CERTIFICATE-----
-----BEGIN PRIVATE KEY-----
<your-key>
-----END PRIVATE KEY-----
EOF
```

Add TLS to Ingress:

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - nginx.training.local
      secretName: nginx-tls
  rules:
    - host: nginx.training.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-clusterip
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f 14-ingress.yaml

# Test HTTPS
curl -k https://nginx.training.local
```

---

## Step 9: Name-based Virtual Hosting

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

---

## Step 10: Common Ingress Annotations

```yaml
metadata:
  annotations:
    # Rewrite URL path
    nginx.ingress.kubernetes.io/rewrite-target: /

    # Enable cors
    nginx.ingress.kubernetes.io/enable-cors: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Proxy timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

---

## Step 11: Path Types

```yaml
paths:
  # Exact match (highest priority)
  - path: /exact
    pathType: Exact
    backend:
      service:
        name: exact-service
        port:
          number: 80

  # Prefix match
  - path: /prefix
    pathType: Prefix
    backend:
      service:
        name: prefix-service
        port:
          number: 80

  # Implementation specific (default)
  - path: /default
    pathType: ImplementationSpecific
    backend:
      service:
        name: default-service
        port:
          number: 80
```

---

## Step 12: Cleanup

```bash
kubectl delete ingress nginx-ingress -n training
kubectl delete secret nginx-tls -n training
```

---

## Ingress vs Service Types

| Type | Use Case | Limitations |
|------|----------|-------------|
| **ClusterIP** | Internal only | No external access |
| **NodePort** | Simple external access | Fixed port, no routing rules |
| **LoadBalancer** | Cloud provider LB | No HTTP routing, costs money |
| **Ingress** | HTTP/HTTPS routing | Requires controller |

---

## Verification Checklist

- [ ] Ingress controller installed and running
- [ ] Ingress resource created
- [ ] Host-based routing works
- [ ] Path-based routing works
- [ ] TLS termination works
- [ ] Multiple hosts on same Ingress
- [ ] /etc/hosts configured for local testing

---

## Key Commands Summary

```bash
# List/Describe
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>

# Create secret for TLS
kubectl create secret tls <name> --cert=<cert-file> --key=<key-file> -n <namespace>

# Delete
kubectl delete ingress <name> -n <namespace>
```

---

## Troubleshooting

### 404 Not Found
```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check service exists
kubectl get svc -n <namespace>
```

### Ingress not getting IP
```bash
# Check ingress controller status
kubectl get pods -n ingress-nginx

# For minikube, enable ingress addon
minikube addons enable ingress
```

---

## Next Lab

Proceed to [Lab 15: RBAC](../k8s-training/15-rbac.yaml) - Role-based access control.
