# Lab 05: Deployments - Declarative Updates

## Objective
Learn how to use Deployments for managing application updates, scaling, and rollbacks.

---

## What is a Deployment?

A **Deployment** provides declarative updates for Pods and ReplicaSets.

**Key capabilities:**
- Rolling updates (zero-downtime)
- Rollback to previous versions
- Scale up/down easily
- Pause/resume deployments
- View deployment history

---

## Step 1: Create a Deployment

Create file `05-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: training
  labels:
    app: nginx-deploy
    lesson: "05"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: nginx-deploy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # At most 1 Pod unavailable during update
      maxSurge: 1           # At most 1 extra Pod during update
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
              name: http
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
kubectl apply -f 05-deployment.yaml
```

---

## Step 2: Verify Deployment

```bash
# List deployments
kubectl get deployments -n training
kubectl get deploy -n training

# Get details
kubectl describe deployment nginx-deployment -n training
```

Expected output:
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

**Columns:**
- `READY` - Available pods / Desired pods
- `UP-TO-DATE` - Pods updated to latest version
- `AVAILABLE` - Pods available to serve traffic

---

## Step 3: Check ReplicaSet Created

```bash
# Deployment creates a ReplicaSet automatically
kubectl get rs -n training -l app=nginx-deploy
```

---

## Step 4: Rolling Update

Update the nginx image version:

```bash
# Update image to nginx:1.26-alpine
kubectl set image deployment/nginx-deployment nginx=nginx:1.26-alpine -n training

# Watch the rollout
kubectl rollout status deployment/nginx-deployment -n training
```

---

## Step 5: Observe Rolling Update Process

```bash
# Watch pods being replaced
kubectl get pods -n training -l app=nginx-deploy -w
```

You should see:
1. New pod created (`nginx-xxxx-v2`)
2. Old pod terminated (`nginx-xxxx-v1`)
3. Process continues until all are updated

---

## Step 6: View Deployment History

```bash
# View revision history
kubectl rollout history deployment/nginx-deployment -n training

# View specific revision details
kubectl rollout history deployment/nginx-deployment -n training --revision=2
```

---

## Step 7: Rollback to Previous Version

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment -n training

# Watch rollback
kubectl rollout status deployment/nginx-deployment -n training

# Verify image is back to 1.25-alpine
kubectl get deployment nginx-deployment -n training -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## Step 8: Rollback to Specific Revision

```bash
# First, update to newer version again
kubectl set image deployment/nginx-deployment nginx=nginx:1.27-alpine -n training
kubectl rollout status deployment/nginx-deployment -n training

# Rollback to revision 1
kubectl rollout undo deployment/nginx-deployment -n training --to-revision=2
```

---

## Step 9: Scale the Deployment

```bash
# Scale up to 5 replicas
kubectl scale deployment/nginx-deployment -n training --replicas=5

# Verify
kubectl get deployment nginx-deployment -n training

# Scale down to 2
kubectl scale deployment/nginx-deployment -n training --replicas=2
```

---

## Step 10: Pause and Resume Deployment

Useful for multiple changes without triggering rollout:

```bash
# Pause deployment
kubectl rollout pause deployment/nginx-deployment -n training

# Make changes while paused (no rollout triggered)
kubectl set image deployment/nginx-deployment nginx=nginx:1.27-alpine -n training
kubectl scale deployment/nginx-deployment -n training --replicas=10

# Check: No rollout happening
kubectl get pods -n training -l app=nginx-deploy

# Resume deployment
kubectl rollout resume deployment/nginx-deployment -n training

# Watch rollout proceed
kubectl rollout status deployment/nginx-deployment -n training
```

---

## Step 11: Restart Deployment

```bash
# Rolling restart (useful for refreshing config)
kubectl rollout restart deployment/nginx-deployment -n training

# Watch pods get recreated
kubectl get pods -n training -l app=nginx-deploy -w
```

---

## Step 12: Cleanup

```bash
kubectl delete deployment nginx-deployment -n training

# Verify ReplicaSet and pods are also deleted
kubectl get all -n training -l app=nginx-deploy
```

---

## Verification Checklist

- [ ] Deployment created with 3 replicas
- [ ] Rolling update replaces pods gradually
- [ ] `kubectl rollout history` shows revisions
- [ ] `kubectl rollout undo` reverts to previous version
- [ ] Scale up/down works correctly
- [ ] Pause/resume prevents partial updates

---

## Deployment Strategy Options

### RollingUpdate (Default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1    # How many can be unavailable
    maxSurge: 1          # How many extra can be created
```

### Recreate
```yaml
strategy:
  type: Recreate
```
Kills ALL old pods BEFORE creating new ones (causes downtime!)

---

## Key Commands Summary

```bash
# Create/Apply
kubectl apply -f 05-deployment.yaml

# Status
kubectl get deployment -n training
kubectl describe deployment <name> -n training
kubectl rollout status deployment/<name> -n training

# Updates
kubectl set image deployment/<name> <container>=<image:tag> -n training
kubectl rollout restart deployment/<name> -n training

# Rollback
kubectl rollout history deployment/<name> -n training
kubectl rollout undo deployment/<name> -n training
kubectl rollout undo deployment/<name> -n training --to-revision=<N>

# Scaling
kubectl scale deployment/<name> -n training --replicas=5

# Pause/Resume
kubectl rollout pause deployment/<name> -n training
kubectl rollout resume deployment/<name> -n training

# Delete
kubectl delete deployment <name> -n training
```

---

## Common Issues

### Old pods not terminating
```bash
# Check rollout status
kubectl rollout status deployment/<name> -n training

# Check if image exists
kubectl describe pod <pod-name> -n training | grep -i image
```

### Deployment stuck
```bash
# Check events
kubectl describe deployment <name> -n training

# Check replica set
kubectl get rs -n training -l app=<label>
```

---

## Next Lab

Proceed to [Lab 06: Services](../k8s-training/06-service.yaml) - Exposing pods to the network.
