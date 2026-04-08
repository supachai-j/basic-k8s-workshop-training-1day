# Lab 10: Resource Management - Requests, Limits, and Quotas

## Objective
Learn how to manage compute resources with requests, limits, ResourceQuota, and LimitRange.

---

## What are Resource Requests and Limits?

**Requests** - Minimum resources guaranteed to a container. Scheduler uses this to find suitable nodes.

**Limits** - Maximum resources a container can use. Enforced at runtime.

**Key concepts:**
- Requests != Limits (container can burst above requests)
- Without limits, container can consume all node resources
- Scheduler places pods based on Requests, not Limits
- Limits enforced by container runtime (cgroups)

---

## Step 1: Create Deployment with Resources

Create file `10-resource-management.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: training
  labels:
    app: resource-demo
    lesson: "10"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80

          # Resource management
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

Apply:

```bash
kubectl apply -f 10-resource-management.yaml
```

---

## Step 2: Verify Resources

```bash
# Check pod resource status
kubectl get pods -n training -l app=resource-demo

# Describe pod to see allocated resources
kubectl describe pod -n training -l app=resource-demo | grep -A 10 "Resources"

# Check actual resource usage
kubectl top pods -n training -l app=resource-demo
```

---

## Step 3: Create LimitRange

LimitRange sets default limits for pods in a namespace if not specified:

```yaml
---
apiVersion: v1
kind: LimitRange
metadata:
  name: training-limits
  namespace: training
spec:
  limits:
    - type: Container
      default:
        cpu: "100m"
        memory: "128Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "500m"
        memory: "512Mi"
      min:
        cpu: "10m"
        memory: "16Mi"
```

Apply:

```bash
kubectl apply -f 10-resource-management.yaml
```

---

## Step 4: Verify LimitRange

```bash
# List LimitRanges
kubectl get limitrange -n training
kubectl describe limitrange training-limits -n training
```

---

## Step 5: Create Pod Without Resources (LimitRange Default)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-resources
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
```

```bash
# Create pod without resource specs
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: default-resources
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
EOF

# Check if LimitRange defaults were applied
kubectl describe pod default-resources -n training | grep -A 5 "Resources"
```

---

## Step 6: Test Resource Limits

Try to create a pod exceeding max limits:

```bash
# Try to create pod with exceeds max limit
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: exceeds-limits
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      resources:
        requests:
          cpu: "10m"
          memory: "16Mi"
        limits:
          cpu: "1"      # Exceeds max of 500m
          memory: "1Gi"  # Exceeds max of 512Mi
EOF

# This should be rejected
```

---

## Step 7: Create ResourceQuota

ResourceQuota limits total resources in a namespace:

```yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: training-quota
  namespace: training
spec:
  hard:
    pods: "10"
    services: "5"
    cpu: "2"
    memory: "4Gi"
```

Apply:

```bash
kubectl apply -f 10-resource-management.yaml
```

---

## Step 8: Verify ResourceQuota

```bash
# List ResourceQuotas
kubectl get resourcequota -n training
kubectl describe resourcequota training-quota -n training
```

---

## Step 9: Test Quota Limits

Try to create more pods than quota allows:

```bash
# Check current pod count
kubectl get pods -n training

# Try to scale deployment beyond quota
kubectl scale deployment resource-demo -n training --replicas=20
```

---

## Step 10: QoS Classes

Kubernetes assigns QoS based on resource specification:

```yaml
# Guaranteed QoS (Best priority)
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "100m"    # Equal to requests
    memory: "128Mi"

# Burstable (Medium priority)
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"    # Higher than requests
    memory: "256Mi"

# BestEffort (Lowest priority)
# No requests or limits specified
```

---

## Step 11: Check QoS Class

```bash
# Check QoS for pods
kubectl get pods -n training -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# Most training lab pods are Burstable (have requests but limits != requests)
```

---

## Step 12: Cleanup

```bash
kubectl delete -f 10-resource-management.yaml
```

---

## Resource Types

| Resource | Description | Unit |
|----------|-------------|------|
| `cpu` | Compute resources | cores (m = millicores = 0.001) |
| `memory` | RAM usage | bytes (Mi, Gi) |
| `ephemeral-storage` | Temporary storage | bytes (Mi, Gi) |
| `storage` | Persistent storage | bytes (Mi, Gi) |
| `nvidia.com/gpu` | GPU cards | integer |

---

## Verification Checklist

- [ ] Pod created with specified requests/limits
- [ ] LimitRange provides defaults to pods without specs
- [ ] Pod exceeding LimitRange max is rejected
- [ ] ResourceQuota limits namespace totals
- [ ] QoS class correctly assigned based on resource specs
- [ ] `kubectl top` shows actual resource usage

---

## Key Commands Summary

```bash
# Check resources
kubectl describe pod <name> -n <ns> | grep -A 5 Resources
kubectl top pods -n <namespace>
kubectl top nodes

# LimitRange
kubectl get limitrange -n <namespace>
kubectl describe limitrange <name> -n <namespace>

# ResourceQuota
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>

# QoS
kubectl get pods -n <namespace> -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

---

## Best Practices

1. **Always set resource requests** - Scheduler needs this for placement
2. **Set limits slightly higher than requests** - Allow burst capability
3. **Use LimitRange defaults** - Prevents pods without specs
4. **Monitor actual usage** - Adjust based on real data
5. **Set appropriate limits** - Prevent noisy neighbor problems

---

## Next Lab

Proceed to [Lab 11: Health Probes](../k8s-training/11-probes.yaml) - Liveness, Readiness, and Startup probes.
