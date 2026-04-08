# Lab 04: ReplicaSets - Maintaining Pod Count

## Objective
Learn how ReplicaSet ensures the specified number of pod replicas are always running.

---

## What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of pod replicas are running at any given time.

**Key characteristics:**
- Maintains stable set of replica pods
- Automatically replaces failed or deleted pods
- Usually not used directly - Deployment manages ReplicaSets
- Uses label selectors to find pods it should manage

---

## Step 1: Create a ReplicaSet

Create file `04-replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  namespace: training
  labels:
    app: nginx-rs
    lesson: "04"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
```

Apply:

```bash
kubectl apply -f 04-replicaset.yaml
```

---

## Step 2: Verify ReplicaSet

```bash
# List ReplicaSets
kubectl get replicasets -n training
kubectl get rs -n training

# More details
kubectl get rs -n training -o wide

# Describe ReplicaSet
kubectl describe rs nginx-replicaset -n training
```

Expected output:
```
NAME                  DESIRED   CURRENT   READY   AGE
nginx-replicaset      3         3         3       30s
```

---

## Step 3: Verify Pods Created

```bash
# List pods
kubectl get pods -n training -l app=nginx-rs

# Check labels
kubectl get pods -n training --show-labels
```

Expected: 3 pods with `app=nginx-rs` label

---

## Step 4: Test Self-Healing

**Delete one pod manually:**

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n training -l app=nginx-rs -o jsonpath='{.items[0].metadata.name}')
echo "Deleting: $POD_NAME"

# Delete the pod
kubectl delete pod $POD_NAME -n training

# Watch recovery (new pod should be created automatically)
kubectl get pods -n training -l app=nginx-rs -w
```

ReplicaSet should automatically create a new pod to maintain 3 replicas.

---

## Step 5: Test Scale Up

```bash
# Scale up to 5 replicas
kubectl scale rs nginx-replicaset -n training --replicas=5

# Verify
kubectl get rs -n training
kubectl get pods -n training -l app=nginx-rs
```

---

## Step 6: Test Scale Down

```bash
# Scale down to 2 replicas
kubectl scale rs nginx-replicaset -n training --replicas=2

# Verify (should have 2 pods, 3 terminating)
kubectl get pods -n training -l app=nginx-rs
```

---

## Step 7: Pod Selector Mismatch

Create a pod with the same label but outside ReplicaSet management:

```bash
# Create a pod with matching label
kubectl run orphan-pod --image=nginx:1.25-alpine \
  --labels=app=nginx-rs \
  -n training

# Check ReplicaSet status
kubectl get rs -n training

# Notice: Still shows 2 ready (not 3!)
```

---

## Step 8: Verify Orphan Pod Behavior

```bash
# Check which pods are managed
kubectl get pods -n training -l app=nginx-rs

# Describe ReplicaSet events
kubectl describe rs nginx-replicaset -n training | grep -A 10 Events

```

---



## Step 9: Cleanup

```bash
# Delete ReplicaSet (this also deletes all managed pods)
kubectl delete rs nginx-replicaset -n training

# Verify all pods are deleted
kubectl get pods -n training -l app=nginx-replica
```

---

## ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| **Purpose** | Maintain pod count | Declarative updates |
| **Rolling Updates** | Manual | Automatic |
| **Rollback** | Manual | Built-in |
| **Use Case** | Rarely used directly | Most common |

**Best Practice:** Always use Deployment, not ReplicaSet directly.

---

## Verification Checklist

- [ ] ReplicaSet created with 3 desired replicas
- [ ] 3 pods are running
- [ ] Deleting a pod creates a replacement automatically
- [ ] Scale up/down works
- [ ] Cleanup removes all pods

---

## Key Commands Summary

```bash
# Create/Apply
kubectl apply -f 04-replicaset.yaml

# List/Describe
kubectl get replicasets -n training
kubectl get rs -n training
kubectl describe rs <name> -n training

# Scale
kubectl scale rs <name> -n training --replicas=5

# Update image
kubectl set image rs/<name> <container>=<new-image> -n training

# Delete (also deletes pods)
kubectl delete rs <name> -n training
```

---

## Common Issues

### Selector matches existing pods
```bash
# Warning: ReplicaSet picks up existing pods!
# Solution: Use unique labels for each ReplicaSet
```

### Image pull errors
```bash
kubectl describe rs <name> -n training
# Check Events for image-related errors
```

---

## Next Lab

Proceed to [Lab 05: Deployments](../k8s-training/05-deployment.yaml) - Declarative updates with rolling strategy.
