# Lab 06: Services - Exposing Pods to the Network

## Objective
Learn how Kubernetes Services provide stable network endpoints for accessing Pods.

---

## What is a Service?

A **Service** provides a stable IP address and DNS name to access a group of Pods, even when Pods are recreated.

**Key characteristics:**
- Stable endpoint (doesn't change when pods die)
- Load balances across multiple pods
- Supports multiple access types (ClusterIP, NodePort, LoadBalancer)
- Uses label selectors to find target pods

---

## Prerequisites

```bash
# First, create a deployment to expose
kubectl apply -f ../k8s-training/05-deployment.yaml

# Verify deployment is running
kubectl get deployment nginx-deployment -n training
```

---

## Step 1: Create ClusterIP Service

Create file `06-service.yaml` (or apply from k8s-training folder):

```yaml
# ClusterIP Service - Internal only
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  namespace: training
  labels:
    lesson: "06"
spec:
  type: ClusterIP
  selector:
    app: nginx-deploy
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

Apply:

```bash
kubectl apply -f 06-service.yaml
```

---

## Step 2: Verify Service

```bash
# List services
kubectl get services -n training
kubectl get svc -n training

# Get service details
kubectl describe svc nginx-clusterip -n training
```

Expected output:
```
NAME             TYPE        CLUSTER-IP     PORT(S)   AGE
nginx-clusterip  ClusterIP   10.96.45.123   80/TCP    30s
```

---

## Step 3: Test ClusterIP from Within Cluster

```bash
# Create a test pod
kubectl run curl-test --rm -it --image=curlimages/curl -n training -- \
  sh -c 'curl http://nginx-clusterip:80'

# Or using busybox
kubectl run dns-test --rm -it --image=busybox:1.36 -n training -- \
  sh -c 'wget -qO- http://nginx-clusterip:80'
```

---

## Step 4: Get Endpoints

Endpoints show which Pod IPs are behind the Service:

```bash
# Get endpoints
kubectl get endpoints nginx-clusterip -n training

# Compare with pod IPs
kubectl get pods -n training -l app=nginx-deploy -o wide
```

---

## Step 5: Create NodePort Service

Add to `06-service.yaml`:

```yaml
# NodePort Service - External via Node IP
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: training
  labels:
    lesson: "06"
spec:
  type: NodePort
  selector:
    app: nginx-deploy
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f 06-service.yaml
```

---

## Step 6: Access via NodePort

```bash
# Get node IPs
kubectl get nodes -o wide

# Access service
curl http://<NODE-IP>:30080
```

**NodePort range:** 30000-32767

---

## Step 7: Create Headless Service

Add to `06-service.yaml`:

```yaml
# Headless Service - No load balancing, returns pod IPs directly
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
  namespace: training
  labels:
    lesson: "06"
spec:
  clusterIP: None  # This makes it headless!
  selector:
    app: nginx-deploy
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f 06-service.yaml
```

---

## Step 8: Test Headless Service

```bash
# Query DNS
kubectl run dnsutils --rm -it --image=tutum/dnsutils -n training -- /bin/sh

# Inside the pod:
nslookup nginx-headless.training.svc.cluster.local

# Exit: exit
```

For headless services, DNS returns individual Pod IPs instead of Service IP.

---

## Step 9: Service Discovery

Services are discoverable via:

```bash
# Full DNS: <service>.<namespace>.svc.<cluster-domain>
nginx-clusterip.training.svc.cluster.local

# Within namespace: <service>
nginx-clusterip

# Short form: works within same namespace
nginx-clusterip:80
```

---

## Step 10: Service Without Selector

Services can point to external endpoints:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
  namespace: training
spec:
  ports:
    - port: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
  namespace: training
subsets:
  - addresses:
      - ip: 203.0.113.10
    ports:
      - port: 80
```

---

## Step 11: Port Forward to Service

```bash
# Port forward service to local
kubectl port-forward svc/nginx-clusterip 8080:80 -n training

# Test
curl http://localhost:8080
```

---

## Step 12: Cleanup

```bash
kubectl delete -f 06-service.yaml
```

---

## Service Types Summary

| Type | Access | Use Case |
|------|--------|----------|
| **ClusterIP** | Internal only | Internal microservices |
| **NodePort** | `<NodeIP>:<30000-32767>` | Development, simple external access |
| **LoadBalancer** | Cloud provider LB | Production external access |
| **Headless** | Direct pod IPs | StatefulSets, custom discovery |

---

## Verification Checklist

- [ ] ClusterIP service created and accessible internally
- [ ] `kubectl get endpoints` shows pod IPs
- [ ] NodePort service accessible via Node IP
- [ ] Headless service returns pod IPs in DNS
- [ ] Service selector correctly matches pod labels

---

## Key Commands Summary

```bash
# Create/Apply
kubectl apply -f 06-service.yaml

# List/Describe
kubectl get services -n training
kubectl get svc -n training
kubectl describe svc <name> -n training

# Endpoints
kubectl get endpoints <name> -n training

# Access
kubectl run test --rm -it --image=curlimages/curl -n training -- sh
curl http://<service-name>:<port>

# Port forward
kubectl port-forward svc/<name> <local>:<remote> -n training

# Delete
kubectl delete svc <name> -n training
```

---

## Troubleshooting

### Service cannot find pods
```bash
# Check selector matches pod labels
kubectl describe svc <name> -n training | grep Selector

# Verify pod labels
kubectl get pods --show-labels -n training
```

### No endpoints
```bash
# Pods might not be running
kubectl get pods -n training

# Or selector mismatch
kubectl describe svc <name> -n training
```

---

## Next Lab

Proceed to [Lab 07: ConfigMap & Secret](../k8s-training/07-configmap-secret.yaml) - Externalized configuration.
