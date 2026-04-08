# Lab 02: Pods - The Smallest Deployable Unit

## Objective
Learn how to create, inspect, and manage Pods in Kubernetes.

---

## What is a Pod?

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share:
- Network namespace (same IP, localhost)
- Storage (shared volumes)
- Process namespace (see each other's processes)

**Key characteristics:**
- Ephemeral (can be destroyed and recreated)
- Usually contains 1 container (sidecar pattern除外)
- Has a unique IP within the cluster

---

## Prerequisites

```bash
# Verify training namespace exists
kubectl get namespace training

# If not, create it
kubectl create namespace training
```

---

## Step 1: Create a Pod from YAML

Create file `02-pod.yaml` in current directory:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-basic
  namespace: training
  labels:
    app: nginx
    tier: frontend
    lesson: "02"
  annotations:
    description: "Basic Pod example for K8s training"
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
          name: http
      # Resource requests and limits (covered in detail in Lesson 10)
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
```

Apply the Pod:

```bash
kubectl apply -f 02-pod.yaml
```

---

## Step 2: Verify Pod Status

```bash
# List pods in training namespace
kubectl get pods -n training

# Check pod status
kubectl get pods -n training -o wide
```

Expected output:
```
NAME          READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-basic   1/1     Running   0          30s   10.42.0.15   k8s-node
```

**STATUS meanings:**
- `Pending` - Pod is being scheduled or downloading image
- `Running` - Pod is running
- `ContainerCreating` - Containers are being created
- `CrashLoopBackOff` - Container keeps crashing
- `Error` - Pod encountered an error

---

## Step 3: Describe the Pod

Get detailed information about the Pod:

```bash
kubectl describe pod nginx-basic -n training
```

This shows:
- Pod details, labels, annotations
- Containers configuration
- Events (scheduling, image pulling, restarts)
- Resource usage

---

## Step 4: View Pod Logs

```bash
# View nginx logs
kubectl logs nginx-basic -n training

# Follow logs in real-time
kubectl logs -f nginx-basic -n training

# View previous logs (if container restarted)
kubectl logs nginx-basic -n training --previous
```

---

## Step 5: Exec into Pod Container

Access the container's shell:

```bash
# Get shell access
kubectl exec -it nginx-basic -n training -- /bin/sh

# Once inside, try these commands:
# ls -la /usr/share/nginx/html
# cat /etc/os-release
# nginx -v
# exit
```

**Alternative - run a single command:**

```bash
# Run hostname inside pod
kubectl exec nginx-basic -n training -- hostname

# Run with specific container (for multi-container pods)
kubectl exec -it nginx-basic -n training -c nginx -- /bin/sh
```

---

## Step 6: Access Pod Network

```bash
# Get Pod IP
kubectl get pod nginx-basic -n training -o jsonpath='{.status.podIP}'

# Curl from within cluster (create temporary pod)
kubectl run curl-test --rm -it --image=curlimages/curl -n training -- \
  curl http://10.42.0.15:80
```

---

## Step 7: Port Forward (Local Access)

Forward local port to Pod port:

```bash
# In terminal 1: Run port forward
kubectl port-forward nginx-basic -n training 8081:80

# In terminal 2: Access nginx
curl http://localhost:8081
```

---

## Step 8: Inspect Pod YAML

Get the full Pod specification:

```bash
# View full YAML
kubectl get pod nginx-basic -n training -o yaml

# View in JSON
kubectl get pod nginx-basic -n training -o json

# Export to file
kubectl get pod nginx-basic -n training -o yaml > my-pod.yaml
```

---

## Step 9: Update Pod (Immutability)

**Pods are immutable** - you cannot update most fields directly.
Instead, you delete and recreate:

```bash
# Delete the Pod
kubectl delete pod nginx-basic -n training

# Modify YAML and recreate
kubectl apply -f 02-pod.yaml

# Verify
kubectl get pods -n training
```

---

## Step 10: Delete the Pod

```bash
kubectl delete pod nginx-basic -n training

# Verify deletion
kubectl get pods -n training
```

---

## Verification Checklist

- [ ] Pod is in `Running` status
- [ ] `kubectl describe pod` shows events without errors
- [ ] `kubectl logs` shows nginx access logs
- [ ] `kubectl exec` can access container shell
- [ ] Port forward works from localhost

---

## Key Commands Summary

```bash
# Create/Update
kubectl apply -f 02-pod.yaml

# List/Describe
kubectl get pods -n training
kubectl get pods -n training -o wide
kubectl describe pod <name> -n training

# Logs
kubectl logs <name> -n training
kubectl logs -f <name> -n training
kubectl logs <name> -n training --previous

# Exec
kubectl exec -it <name> -n training -- /bin/sh
kubectl exec <name> -n training -- <command>

# Port forward
kubectl port-forward <name> -n training <local-port>:<pod-port>

# Delete
kubectl delete pod <name> -n training

# Export
kubectl get pod <name> -n training -o yaml > file.yaml
```

---

## Common Issues

### ImagePullBackOff
```bash
# Check if image exists
kubectl describe pod <name> -n training | grep -i image

# Fix: Check image name and registry
# Try pulling manually: docker pull nginx:1.25-alpine
```

### CrashLoopBackOff
```bash
# Check logs
kubectl logs <name> -n training
kubectl logs <name> -n training --previous

# Fix: Debug application inside container
kubectl exec -it <name> -n training -- /bin/sh
```

### Pending forever
```bash
# Check events
kubectl describe pod <name> -n training

# Check resources: maybe no nodes with enough CPU/memory
kubectl describe nodes
```

---

## Next Lab

Proceed to [Lab 03: Multi-Container Pods](../k8s-training/03-multi-container-pod.yaml) - Sidecars and init containers.
