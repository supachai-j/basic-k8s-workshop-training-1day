# Lab 16: Horizontal Pod Autoscaler (HPA)

## Objective
Learn how to automatically scale pods based on CPU/memory metrics.

---

## What is HPA?

Horizontal Pod Autoscaler (HPA) automatically scales the number of pods in a Deployment, ReplicaSet, or StatefulSet based on observed resource utilization.

**Key features:**
- Scale based on CPU/memory metrics
- Configurable min/max replicas
- Scale-up and scale-down policies
- Stabilization windows to prevent flapping

**Requirements:**
- Metrics server installed
- Pods must have resource requests set

---

## Step 1: Prerequisites Check

```bash
# Check metrics-server is installed
kubectl get pods -n kube-system | grep metrics

# Or check API availability
kubectl get apiservices | grep metrics
```

For K3s, metrics-server should be pre-installed. If not:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Step 2: Create Deployment with Resources

Create file `16-hpa.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-hpa
  namespace: training
  labels:
    app: web-hpa
    lesson: "16"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-hpa
  template:
    metadata:
      labels:
        app: web-hpa
    spec:
      containers:
        - name: web
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          # Resource requests REQUIRED for HPA
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web-hpa-svc
  namespace: training
spec:
  selector:
    app: web-hpa
  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f 16-hpa.yaml
```

---

## Step 3: Create HPA

Add to `16-hpa.yaml`:

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: training
  labels:
    lesson: "16"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-hpa
  # Scaling bounds
  minReplicas: 1
  maxReplicas: 10
  # Metrics to scale on
  metrics:
    # Scale when average CPU exceeds 50% of request
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    # Also consider memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  # Scaling behavior
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 10
      policies:
        - type: Pods
          value: 2
          periodSeconds: 30
```

Apply:

```bash
kubectl apply -f 16-hpa.yaml
```

---

## Step 4: Verify HPA

```bash
# List HPAs
kubectl get hpa -n training
kubectl get hpa -n training -w

# Describe HPA
kubectl describe hpa web-hpa -n training
```

Expected output:
```
NAME      REFERENCE            TARGETS                MINPODS   MAXPODS   REPLICAS   AGE
web-hpa   Deployment/web-hpa   7%/50%, 35%/70%        1         10        1          1m
```

---

## Step 5: Check Current Metrics

```bash
# View current pod metrics
kubectl top pods -n training -l app=web-hpa

# View node metrics
kubectl top nodes
```

---

## Step 6: Generate Load to Trigger Scale

Open two terminals:

**Terminal 1:** Generate load
```bash
kubectl run load-gen --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -n training -- \
  sh -c 'while true; do wget -q -O- http://web-hpa-svc:80 > /dev/null; done'
```

**Terminal 2:** Watch HPA
```bash
kubectl get hpa -n training -w
kubectl get pods -n training -l app=web-hpa -w
```

---

## Step 7: Observe Scaling

Watch as:
1. CPU usage increases
2. HPA detects high utilization (>50%)
3. Pod replicas increase (2, 3, 4...)
4. Load is distributed across more pods

Stop the load generator when you see scaling (Ctrl+C).

---

## Step 8: Stop Load and Observe Scale Down

After stopping load:
1. CPU usage drops
2. HPA waits 60 seconds (stabilization window)
3. Pod replicas gradually decrease
4. Returns to minimum replicas

---

## Step 9: HPA API Versions

```bash
# autoscaling/v1 - basic (CPU only)
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
spec:
  targetCPUUtilizationPercentage: 50

# autoscaling/v2 - modern (CPU, memory, custom)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

# autoscaling/v2beta2 - adds custom metrics
```

---

## Step 10: Scale Manually

Override HPA for maintenance:

```bash
# Set minimum replicas
kubectl patch hpa web-hpa -n training -p '{"spec":{"minReplicas":5}}'

# Temporarily disable HPA
kubectl patch hpa web-hpa -n training -p '{"spec":{"minReplicas":10,"maxReplicas":10}}'

# Restore HPA
kubectl patch hpa web-hpa -n training -p '{"spec":{"minReplicas":1,"maxReplicas":10}}'
```

---

## Step 11: HPA Behavior Policies

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300   # 5 minutes
    policies:
      - type: Percent
        value: 100                    # Remove 100% of pods possible
        periodSeconds: 15
      - type: Pods
        value: 4                      # Remove at most 4 pods per period
        periodSeconds: 60
    selectPolicy: Min               # Use the policy that scales fewer pods

  scaleUp:
    stabilizationWindowSeconds: 0    # Scale up immediately
    policies:
      - type: Pods
        value: 4                      # Add at most 4 pods per 15 seconds
        periodSeconds: 15
      - type: Percent
        value: 100                    # Add up to 100% of current pods
        periodSeconds: 15
    selectPolicy: Max               # Use policy that scales more pods
```

---

## Step 12: Cleanup

```bash
kubectl delete -f 16-hpa.yaml
```

---

## HPA Requirements Checklist

- [ ] metrics-server installed and running
- [ ] Deployment has resource requests
- [ ] HPA minReplicas <= maxReplicas
- [ ] HPA targets existing Deployment

---

## Verification Checklist

- [ ] HPA created with correct min/max
- [ ] `kubectl top pods` shows metrics
- [ ] Load generation triggers scale-out
- [ ] Stopping load triggers scale-in
- [ ] Scale-down respects stabilization window
- [ ] Cleanup removes HPA and deployment

---

## Key Commands Summary

```bash
# List/Describe HPA
kubectl get hpa -n <namespace>
kubectl describe hpa <name> -n <namespace>

# View metrics
kubectl top pods -n <namespace>
kubectl top nodes

# Manual scale (override HPA temporarily)
kubectl scale deployment <name> -n <namespace> --replicas=5

# Patch HPA
kubectl patch hpa <name> -n <namespace> -p '{"spec":{"minReplicas":3}}'

# Delete HPA (doesn't delete deployment)
kubectl delete hpa <name> -n <namespace>
```

---

## Troubleshooting

### HPA not scaling
```bash
# Check metrics-server
kubectl get pods -n kube-system | grep metrics

# Check pod has resource requests
kubectl describe pod <pod> -n <namespace> | grep -A 5 "Resources"

# Check HPA events
kubectl describe hpa <name> -n <namespace> | grep -A 10 Events
```

### Unknown metric
```bash
# Verify metrics available
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/training/pods | jq
```

---

## Next Lab

Proceed to [Lab 17: Network Policies](../k8s-training/17-network-policy.yaml) - Pod-to-pod traffic control.
