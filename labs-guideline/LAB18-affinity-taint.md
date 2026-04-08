# Lab 18: Affinity, Taints, and Tolerations

## Objective
Learn how to control pod scheduling using node affinity, taints, and tolerations.

---

## Overview

**Taints** - Applied to nodes to repel pods (node says "don't schedule here unless tolerated")

**Tolerations** - Applied to pods to allow scheduling on tainted nodes (pod says "I can handle these conditions")

**Affinity** - Pod preferences/requirements for nodes or other pods (pod says "I prefer/demand these conditions")

---

## Part 1: Taints and Tolerations

### What are Taints?

Taints are applied to nodes to prevent pods from being scheduled unless the pod has a matching toleration.

**Taint Effect Types:**
| Effect | Behavior |
|--------|----------|
| **NoSchedule** | Pod not scheduled, existing pods not evicted |
| **PreferNoSchedule** | Scheduler tries to avoid, but not enforced |
| **NoExecute** | Pod not scheduled AND existing pods may be evicted |

---

### Step 1: Add Taint to Node

```bash
# Get node name
kubectl get nodes

# Add taint to node
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule

# Verify taint
kubectl describe node <node-name> | grep -A 5 Taints
```

---

### Step 2: Create Pod Without Toleration

```bash
# Try to schedule pod on tainted node
kubectl run no-toleration --image=nginx:1.25-alpine -n training

# Check where it landed (should NOT be on tainted node)
kubectl get pods -n training -o wide
```

---

### Step 3: Create Pod With Toleration

Add to `18-affinity-taint.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: training
spec:
  containers:
    - name: gpu
      image: nginx:1.25-alpine
  # Tolerate the taint
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: training
spec:
  containers:
    - name: gpu
      image: nginx:1.25-alpine
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
EOF
```

---

### Step 4: Verify Scheduling

```bash
# Check where gpu-pod landed (should be on tainted node)
kubectl get pods gpu-pod -n training -o wide

# Verify no-toleration pod avoided the tainted node
kubectl get pods no-toleration -n training -o wide
```

---

### Step 5: Remove Taint

```bash
# Remove taint from node
kubectl taint nodes <node-name> dedicated=gpu:NoSchedule-

# Verify taint removed
kubectl describe node <node-name> | grep Taints
```

---

## Part 2: Node Affinity

### Node Affinity Types

| Type | Behavior |
|------|----------|
| **requiredDuringSchedulingIgnoredDuringExecution** | Must meet requirements (hard) |
| **preferredDuringSchedulingIgnoredDuringExecution** | Try to meet preferences (soft) |
| **requiredDuringSchedulingRequiredDuringExecution** | Must meet, pods evicted if not (hard + enforcement) |

---

### Step 6: Create Pod with Node Affinity

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "node-role.kubernetes.io/control-plane"
                operator: Exists
EOF
```

---

### Step 7: Preferred Node Affinity

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: "disktype"
                operator: In
                values:
                  - ssd
EOF
```

---

## Part 3: Pod Affinity and Anti-Affinity

### Pod Anti-Affinity - Spread Pods

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  namespace: training
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - web-server
          topologyKey: kubernetes.io/hostname
EOF
```

This ensures web-server pods are spread across different nodes.

---

### Pod Affinity - Co-location

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: logging-sidecar
  namespace: training
spec:
  containers:
    - name: logger
      image: busybox:1.36
      command: ["sh", "-c", "while true; do sleep 10; done"]
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - web-server
          topologyKey: kubernetes.io/hostname
EOF
```

This ensures logging-sidecar is scheduled on the same node as web-server pods.

---

## Part 4: Common Use Cases

### Use Case 1: GPU Nodes

```yaml
# Taint GPU nodes
kubectl taint nodes gpu-node dedicated=gpu:NoSchedule

# Pod tolerates GPU taint
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

---

### Use Case 2: High Memory Nodes

```yaml
# Taint high-memory nodes
kubectl taint nodes highmem-node high-memory=big:PreferNoSchedule

# Pod prefers high-memory nodes
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: "high-memory"
              operator: Exists
```

---

### Use Case 3: Zone Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: "topology.kubernetes.io/zone"
              operator: In
              values:
                - us-east-1a
```

---

## Step 8: Cleanup

```bash
# Delete test pods
kubectl delete pod no-toleration gpu-pod affinity-pod preferred-affinity web-server logging-sidecar -n training

# Remove taints
kubectl taint nodes <node-name> dedicated-
```

---

## Verification Checklist

- [ ] Taint applied to node
- [ ] Pod without toleration avoids tainted node
- [ ] Pod with toleration schedules on tainted node
- [ ] Node affinity rules respected
- [ ] Pod anti-affinity spreads pods across nodes
- [ ] Pod affinity co-locates pods
- [ ] Taints removed after testing

---

## Key Commands Summary

```bash
# Taints
kubectl taint nodes <node> <key>=<value>:<effect>
kubectl taint nodes <node> <key>=<value>:<effect>-   # Remove

# Pod placement
kubectl get pods -o wide
kubectl describe pod <name> | grep -A 10 Node
```

---

## Troubleshooting

### Pod stuck in Pending
```bash
# Check node affinity
kubectl describe pod <name> | grep -A 15 "Node Selectors\|Node Affinity"

# Check taints
kubectl describe node <node-name> | grep Taints

# Check events
kubectl describe pod <name> | grep -A 5 Events
```

---

## Next Lab

Proceed to [Lab 19: Cleanup](../k8s-training/19-cleanup.yaml) - Clean up all training resources.
