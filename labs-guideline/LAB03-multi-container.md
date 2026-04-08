# Lab 03: Multi-Container Pods

## Objective
Learn how to run multiple containers in a single Pod, including sidecar and init container patterns.

---

## What are Multi-Container Pods?

Multi-container Pods contain **multiple containers that:**
- Share the same network namespace (localhost)
- Share storage volumes
- Can communicate via IPC
- Are scheduled on the same node
- Share the same lifecycle

**Common patterns:**
1. **Sidecar** - Helper container (log aggregator, sync tool)
2. **Ambassador** - Proxy for external services
3. **Adapter** - Transform output for consumption

---

## Step 1: Create a Multi-Container Pod

Create file `03-multi-container-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
  namespace: training
  labels:
    app: multi-container-demo
    lesson: "03"
spec:
  # --- Init Container: runs first, must complete successfully ---
  initContainers:
    - name: init-config
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Init container: preparing config..."
          echo '<!DOCTYPE html><html><body><h1>Hello from K8s Training!</h1><p>Served by multi-container pod</p></body></html>' > /shared/index.html
          echo "Config ready."
      volumeMounts:
        - name: shared-data
          mountPath: /shared

  # --- App Containers ---
  containers:
    # Main application container
    - name: app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        - name: log-volume
          mountPath: /var/log/nginx
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"

    # Sidecar: tails nginx access logs
    - name: log-sidecar
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Sidecar started: tailing access log..."
          tail -F /var/log/nginx/access.log 2>/dev/null || tail -f /dev/null
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/nginx
          readOnly: true
      resources:
        requests:
          cpu: "10m"
          memory: "16Mi"
        limits:
          cpu: "50m"
          memory: "32Mi"

  # Shared volumes between containers
  volumes:
    - name: shared-data
      emptyDir: {}
    - name: log-volume
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f 03-multi-container-pod.yaml
```

---

## Step 2: Verify Multi-Container Pod

```bash
# Check pod status
kubectl get pod multi-container-pod -n training

# Watch the startup
kubectl get pod multi-container-pod -n training -w
```

Expected sequence:
1. `Pending` (init container starting)
2. `Pending` (init container completed)
3. `ContainerCreating` (main containers starting)
4. `Running` (all containers running)

---

## Step 3: Check Init Container Logs

```bash
# View init container logs
kubectl logs multi-container -n training -c init-config
```

---

## Step 4: Check All Container Logs

```bash
# View nginx logs
kubectl logs multi-container -n training -c app

# View sidecar logs
kubectl logs multi-container -n training -c log-sidecar

# Tail all logs simultaneously
kubectl logs multi-container -n training -c app -f &
kubectl logs multi-container -n training -c log-sidecar -f
```

---

## Step 5: Exec into Specific Container

```bash
# Exec into nginx
kubectl exec -it multi-container -n training -c app -- /bin/sh

# Inside nginx container:
# ls -la /var/log/nginx
# exit

# Exec into log-agent
kubectl exec -it multi-container -n training -c log-sidecar -- /bin/sh

# Inside log-agent:
# cat /var/log/nginx/access.log
# cat /var/log/nginx/error.log
# exit
```

---

## Step 6: Test Shared Volume

```bash
# Create test file from nginx container
kubectl exec multi-container -n training -c app -- \
  sh -c 'echo "Test from nginx" > /var/log/nginx/test.log'

# Read from log-agent container
kubectl exec multi-container -n training -c log-sidecar -- \
  cat /var/log/nginx/test.log
```

---

## Step 7: Network Communication Between Containers

```bash
# From nginx, curl localhost (same pod, different container)
kubectl exec multi-container -n training -c app -- \
  curl http://localhost:80

# Containers share network - nginx is accessible on localhost:80
```

---

## Step 8: Describe Pod with Multiple Containers

```bash
kubectl describe pod multi-container -n training
```

Notice:
- Shows all containers in Events
- Shows Init Containers separately
- Shows restart count per container

---

## Step 9: Container Restart Behavior

```bash
# Check which container restarted
kubectl describe pod multi-container -n training | grep -A 5 "State:"

# Restart only log-agent container
kubectl exec multi-container -n training -c log-sidecar -- \
  sh -c 'kill 1'

# Watch recovery
kubectl get pod multi-container -n training -w
```

---

## Step 10: Cleanup

```bash
kubectl delete pod multi-container -n training
```

---

## Verification Checklist

- [ ] Pod shows `Init:1/1` then `Running` with `2/2` containers
- [ ] Init container logs show initialization completed
- [ ] Both containers can read/write to shared volume
- [ ] `kubectl exec -c <container>` selects specific container
- [ ] Containers can communicate via localhost

---

## Key Commands Summary

```bash
# List all containers in pod
kubectl get pod <name> -n training -o jsonpath='{.spec.containers[*].name}'

# Logs from specific container
kubectl logs <name> -n training -c <container-name>

# Exec into specific container
kubectl exec -it <name> -n training -c <container-name> -- /bin/sh

# Check init containers
kubectl get pod <name> -n training -o jsonpath='{.spec.initContainers[*].name}'
```

---

## When to Use Multi-Container Pods

| Pattern | Use Case | Example |
|---------|----------|---------|
| **Sidecar** | Extend main container | Log aggregator, sync tool, metrics exporter |
| **Ambassador** | Proxy connections | Redis proxy, service discovery |
| **Adapter** | Transform output | Format data for consumption |
| **Shared process** | Tight coupling | Sidecar for config reload |

**Tip:** Most pods should be single-container. Use multi-container only when containers are tightly coupled and must share resources.

---

## Next Lab

Proceed to [Lab 04: ReplicaSets](../k8s-training/04-replicaset.yaml) - Maintaining pod replicas.
