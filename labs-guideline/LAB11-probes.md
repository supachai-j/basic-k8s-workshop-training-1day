# Lab 11: Health Probes - Liveness, Readiness, and Startup

## Objective
Learn how to configure health checks to keep applications healthy and serving traffic.

---

## What are Health Probes?

Health probes let Kubernetes detect when a container is unhealthy and take action:

**Types of Probes:**

| Probe | Question | Failure Action |
|-------|----------|----------------|
| **Liveness** | "Is container alive?" | Restart container |
| **Readiness** | "Can container receive traffic?" | Remove from Service |
| **Startup** | "Has app finished starting?" | Disables other probes |

---

## Step 1: Create Deployment with Liveness Probe

Create file `11-probes.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
  namespace: training
  labels:
    app: probe-demo
    lesson: "11"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80

          # Liveness Probe - HTTP check
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

          # Readiness Probe - Exec check
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - "nginx -t"
            initialDelaySeconds: 5
            periodSeconds: 3
            failureThreshold: 1
```

Apply:

```bash
kubectl apply -f 11-probes.yaml
```

---

## Step 2: Verify Probes

```bash
# Check pod status
kubectl get pods -n training -l app=probe-demo

# Describe to see probe configuration
kubectl describe pod -n training -l app=probe-demo | grep -A 30 "Liveness"
```

---

## Step 3: Create a Service for Probe Testing

```bash
# Expose deployment
kubectl expose deployment probe-demo -n training \
  --port=80 --target-port=80 \
  --name=probe-demo-svc
```

---

## Step 4: Test Liveness Probe Failure

Create a pod that fails liveness probe:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      # Liveness probe pointing to non-existent endpoint
      livenessProbe:
        httpGet:
          path: /nonexistent
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
        failureThreshold: 1
EOF

# Watch the pod restart
kubectl get pod liveness-fail -n training -w
```

The pod will restart repeatedly (CrashLoopBackOff pattern).

---

## Step 5: Test Readiness Probe Failure

Create a pod that fails readiness but passes liveness:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: readiness-fail
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      # Readiness fails until we create the marker file
      readinessProbe:
        exec:
          command:
            - sh
            - -c
            - "cat /tmp/ready"
        initialDelaySeconds: 2
        periodSeconds: 2
      # Liveness always passes
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
EOF

# Check status - Ready should be 0/1
kubectl get pod readiness-fail -n training

# Create the marker file to make it ready
kubectl exec readiness-fail -n training -- sh -c 'echo ready > /tmp/ready'

# Check again - Ready should become 1/1
kubectl get pod readiness-fail -n training

# Remove marker to see it go NotReady
kubectl exec readiness-fail -n training -- rm /tmp/ready
kubectl get pod readiness-fail -n training -w
```

---

## Step 6: Startup Probe (Slow Starting Apps)

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      # Startup probe - app takes 10 seconds to start
      startupProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 0
        periodSeconds: 2
        failureThreshold: 10   # 10 * 2s = 20s max startup
      # Liveness probe waits for startup to complete
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 5
      # Readiness waits for startup
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
EOF

# Watch it start
kubectl get pod startup-probe -n training -w
```

---

## Step 7: Probe Parameters

```bash
# All probe parameters:
kubectl explain pod.spec.containers.livenessProbe
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Seconds to wait before first check |
| `periodSeconds` | 10 | How often to check |
| `timeoutSeconds` | 1 | Check timeout |
| `failureThreshold` | 3 | Failures before action |
| `successThreshold` | 1 | Successes to recover |

---

## Step 8: Cleanup

```bash
kubectl delete -f 11-probes.yaml
kubectl delete pod liveness-fail readiness-fail startup-probe -n training
kubectl delete svc probe-demo-svc -n training
```

---

## Probe Types

### httpGet
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
      - name: Custom-Header
        value: Alive
```

### tcpSocket
```yaml
readinessProbe:
  tcpSocket:
    port: 5432
```

### exec
```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "redis-cli ping"
```

---

## Verification Checklist

- [ ] Liveness probe configured and working
- [ ] Readiness probe correctly marks pod ready/not ready
- [ ] Startup probe delays liveness during slow startup
- [ ] Failed liveness probe triggers restart
- [ ] Failed readiness probe removes from service endpoints
- [ ] Parameters (initialDelaySeconds, periodSeconds) tuned correctly

---

## Key Commands Summary

```bash
# Check probe status
kubectl describe pod <name> -n <namespace> | grep -A 20 "Liveness"
kubectl describe pod <name> -n <namespace> | grep -A 20 "Readiness"

# Check container status
kubectl get pod <name> -n <namespace> -o jsonpath='{.status.conditions[*]}'

# Test probe manually
kubectl exec <pod> -n <namespace> -- <probe-command>
```

---

## Best Practices

1. **Liveness:** Check if app process is alive, not if it's ready to serve
2. **Readiness:** Check if app can handle traffic (dependencies, loading)
3. **initialDelaySeconds:** Set higher than app startup time
4. **periodSeconds:** Balance between quick detection vs overhead
5. **failureThreshold:** 3-5 for most cases (don't restart too quickly)

---

## Next Lab

Proceed to [Lab 12: StatefulSets](../k8s-training/12-statefulset.yaml) - For stateful applications.
