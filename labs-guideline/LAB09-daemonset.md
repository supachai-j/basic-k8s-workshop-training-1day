# Lab 09: DaemonSets - Pods on Every Node

## Objective
Learn how DaemonSets ensure a copy of a Pod runs on every node in the cluster.

---

## What is a DaemonSet?

A **DaemonSet** ensures that all (or some) nodes run a copy of a specific Pod.

**Use cases:**
- Log collectors (fluentd, logstash)
- Monitoring agents (prometheus node exporter)
- Storage daemons (glusterd, ceph)
- Network plugins (kube-proxy, CNI)

**Key characteristics:**
- One Pod per node (automatic)
- New nodes added to cluster get the DaemonSet Pod automatically
- Pods removed from nodes when DaemonSet is deleted
- Can use node selectors and tolerations to control placement

---

## Step 1: Create a DaemonSet

Create file `09-daemonset.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: training
  labels:
    app: node-monitor
    lesson: "09"
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      # Tolerate master/control-plane taints so it runs on ALL nodes
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: monitor
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              while true; do
                echo "[$(date)] Node: $(hostname) | Uptime: $(cat /proc/uptime | awk '{print $1}')s | Load: $(cat /proc/loadavg)"
                sleep 60
              done
          resources:
            requests:
              cpu: "10m"
              memory: "16Mi"
            limits:
              cpu: "50m"
              memory: "32Mi"
          # Mount host filesystem read-only for monitoring
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
```

Apply:

```bash
kubectl apply -f 09-daemonset.yaml
```

---

## Step 2: Verify DaemonSet

```bash
# List DaemonSets
kubectl get daemonsets -n training
kubectl get ds -n training

# Describe DaemonSet
kubectl describe daemonset node-monitor -n training

# View logs from DaemonSet
kubectl logs -l app=node-monitor -n training
```

Expected: `NUMBER AVAILABLE` should equal number of nodes

---

## Step 3: Verify Pods per Node

```bash
# List pods - should have one pod per node
kubectl get pods -n training -l app=node-monitor -o wide

# Count nodes
kubectl get nodes

# Count daemon pods
kubectl get pods -n training -l app=node-monitor --no-headers | wc -l
```

---

## Step 4: Use Node Selectors

Create a DaemonSet that only runs on specific nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds-selector
  namespace: training
spec:
  selector:
    matchLabels:
      app: nginx-ds-selector
  template:
    metadata:
      labels:
        app: nginx-ds-selector
    spec:
      # Only run on nodes with this label
      nodeSelector:
        disktype: ssd
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
```

---

## Step 5: Label a Node

```bash
# Get node names
kubectl get nodes

# Add label to specific node
kubectl label node <node-name> disktype=ssd

# Verify label
kubectl get nodes --show-labels | grep disktype
```

---

## Step 6: Deploy Selective DaemonSet

```bash
# Apply selective DaemonSet
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds-selector
  namespace: training
spec:
  selector:
    matchLabels:
      app: nginx-ds-selector
  template:
    metadata:
      labels:
        app: nginx-ds-selector
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
EOF

# Check - only runs on nodes with disktype=ssd label
kubectl get pods -n training -l app=nginx-ds-selector -o wide
```

---

## Step 7: Tolerations with DaemonSets

DaemonSets automatically schedule on all nodes including master. Use tolerations to control:

```yaml
spec:
  tolerations:
    # Tolerate all taints (schedules on all nodes including master)
    - operator: Exists

    # Only tolerate specific taint
    - key: "node-role.kubernetes.io/master"
      operator: Exists
      effect: "NoSchedule"

    # Tolerate pods not ready
    - key: "node.kubernetes.io/not-ready"
      operator: Exists
      effect: "NoExecute"
      tolerationSeconds: 300
```

---

## Step 8: Rolling Update DaemonSet

```bash
# Update image
kubectl set image daemonset/nginx-daemonset nginx=nginx:1.26-alpine -n training

# Watch rollout
kubectl get pods -n training -l app=nginx-ds -w

# Check DaemonSet status
kubectl rollout status daemonset/nginx-daemonset -n training
```

---

## Step 9: Rollback DaemonSet

```bash
# View history
kubectl rollout history daemonset nginx-daemonset -n training

# Rollback
kubectl rollout undo daemonset/nginx-daemonset -n training
```

---

## Step 10: Cleanup

```bash
# Delete DaemonSets
kubectl delete daemonset node-monitor -n training
kubectl delete daemonset nginx-ds-selector -n training

# Remove node label
kubectl label node <node-name> disktype-
```

---

## DaemonSet vs Deployment

| Feature | Deployment | DaemonSet |
|---------|------------|-----------|
| **Purpose** | Stateless apps | Node-level daemons |
| **Replicas** | Configurable count | One per node |
| **Scaling** | You specify replicas | Automatic (nodes count) |
| **Scheduling** | Any available node | Every node |
| **Use Case** | Web servers, APIs | Log collectors, agents |

---

## Verification Checklist

- [ ] DaemonSet created successfully
- [ ] One pod exists per cluster node
- [ ] New nodes automatically get DaemonSet pods
- [ ] nodeSelector limits pods to specific nodes
- [ ] DaemonSet can be updated and rolled back
- [ ] Cleanup removes all pods

---

## Key Commands Summary

```bash
# Create/Apply
kubectl apply -f 09-daemonset.yaml

# List/Describe
kubectl get daemonsets -n training
kubectl get ds -n training
kubectl describe daemonset <name> -n training

# Update image
kubectl set image daemonset/<name> <container>=<image:tag> -n training

# Rollback
kubectl rollout undo daemonset/<name> -n training

# Delete
kubectl delete daemonset <name> -n training

# Node labels
kubectl label node <name> <key>=<value>
kubectl label node <name> <key>-
```

---

## Troubleshooting

### DaemonSet pod not on some nodes
```bash
# Check node labels
kubectl get nodes --show-labels

# Check pod events
kubectl describe pod <pod-name> -n training | grep -A 5 Events

# Check DaemonSet selector matches pod labels
kubectl describe daemonset <name> -n training
```

---

## Next Lab

Proceed to [Lab 10: Resource Management](../k8s-training/10-resource-management.yaml) - Requests, limits, and quotas.
