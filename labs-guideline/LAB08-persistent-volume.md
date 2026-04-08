# Lab 08: PersistentVolume & PersistentVolumeClaim - Persistent Storage

## Objective
Learn how to provision and use persistent storage in Kubernetes.

---

## What is PersistentVolume?

**PersistentVolume (PV)** - A cluster-wide storage resource provisioned by admin or dynamically provisioned.

**PersistentVolumeClaim (PVC)** - A request for storage by a user.

**Key concepts:**
- PVs are cluster resources
- PVCs consume PV resources
- PVs persist beyond pod lifecycle
- Multiple pods can share a PV (depending on access mode)

---

## Step 1: Create a PersistentVolume

Create file `08-persistent-volume.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: training-manual-pv
  labels:
    type: training
    lesson: "08"
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce    # Mount by single node
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/training-pv
```

Apply:

```bash
kubectl apply -f 08-persistent-volume.yaml
```

---

## Step 2: Verify PersistentVolume

```bash
# List PVs
kubectl get persistentvolumes
kubectl get pv

# Describe PV
kubectl describe pv training-manual-pv
```

Expected status: `Available`

---

## Step 3: Create PersistentVolumeClaim

Add to `08-persistent-volume.yaml`:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: training
  labels:
    lesson: "08"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 512Mi
  storageClassName: manual
  selector:
    matchLabels:
      type: training
```

Apply:

```bash
kubectl apply -f 08-persistent-volume.yaml
```

---

## Step 4: Verify PVC

```bash
# List PVCs in training namespace
kubectl get persistentvolumeclaims -n training
kubectl get pvc -n training

# Describe PVC
kubectl describe pvc app-pvc -n training
```

Expected: `Bound` to `training-manual-pv`

---

## Step 5: Create Pod Using PVC

Add to `08-persistent-volume.yaml`:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pv-demo
  namespace: training
  labels:
    app: pv-demo
    lesson: "08"
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: app-data
          mountPath: /usr/share/nginx/html
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
  volumes:
    - name: app-data
      persistentVolumeClaim:
        claimName: app-pvc
```

Apply:

```bash
kubectl apply -f 08-persistent-volume.yaml
```

---

## Step 6: Verify Pod

```bash
# Check pod status
kubectl get pod pv-demo -n training

# Check pod is running
kubectl get pod pv-demo -n training -o wide
```

---

## Step 7: Write Data to PV

```bash
# Create test file
kubectl exec pv-demo -n training -- \
  sh -c 'echo "Hello from PV" > /usr/share/nginx/html/index.html'

# Verify file exists
kubectl exec pv-demo -n training -- cat /usr/share/nginx/html/index.html
```

---

## Step 8: Access via Service

```bash
# Create a quick service
kubectl expose pod pv-demo -n training \
  --port=80 --target-port=80 \
  --name=pv-demo-svc

# Test
kubectl run curl-test --rm -it --image=curlimages/curl -n training -- \
  curl http://pv-demo-svc

kubectl delete svc pv-demo-svc -n training
```

---

## Step 9: PV Persists Beyond Pod

```bash
# Delete the pod
kubectl delete pod pv-demo -n training

# PVC still exists
kubectl get pvc -n training

# Create new pod with same PVC
kubectl apply -f 08-persistent-volume.yaml

# Wait for running
kubectl get pod pv-demo -n training -w

# Data still exists!
kubectl exec pv-demo -n training -- cat /usr/share/nginx/html/index.html
```

---

## Step 10: StorageClass (Dynamic Provisioning)

Check available StorageClasses:

```bash
# List storage classes
kubectl get storageclass
kubectl get sc

# For K3s, local-path is common
kubectl get sc -o yaml
```

---

## Step 11: PVC with Dynamic Provisioning

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: training
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard   # Use cluster's default StorageClass
```

```bash
# Create dynamic PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: training
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF

# Check it gets provisioned automatically
kubectl get pvc -n training
```

---

## Step 12: Cleanup

```bash
# Delete pod and PVC
kubectl delete -f 08-persistent-volume.yaml

# PVC deletion (if using dynamic storage)
kubectl delete pvc dynamic-pvc -n training

# PV deletion (if not automatically reclaimed)
kubectl delete pv training-manual-pv
```

---

## Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| **ReadWriteOnce** | RWO | Mounted by single node |
| **ReadOnlyMany** | ROX | Mounted by multiple nodes (read-only) |
| **ReadWriteMany** | RWX | Mounted by multiple nodes (read-write) |

**Note:** Most storage providers only support RWO. NFS supports RWX.

---

## Reclaim Policies

| Policy | Behavior |
|--------|----------|
| **Retain** | PV persists data, manual cleanup needed |
| **Delete** | PV deleted when PVC deleted |
| **Recycle** | Data scrubbed, PV available for reuse (deprecated) |

---

## Verification Checklist

- [ ] PersistentVolume created and in `Available` state
- [ ] PVC bound to PV
- [ ] Pod created with PVC mounted
- [ ] Data written to PV persists after pod deletion
- [ ] New pod can read existing data
- [ ] Cleanup removes all resources

---

## Key Commands Summary

```bash
# PersistentVolume
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>

# PersistentVolumeClaim
kubectl get pvc -n <namespace>
kubectl describe pvc <name> -n <namespace>
kubectl delete pvc <name> -n <namespace>

# StorageClass
kubectl get sc
kubectl get storageclass

# In pod
kubectl exec <pod> -n <namespace> -- cat /path/to/file
```

---

## Troubleshooting

### PVC stuck in Pending
```bash
kubectl describe pvc <name> -n <namespace>
# Check: storage class exists, sufficient capacity
```

### PV not binding
```bash
kubectl describe pv <name>
# Check: capacity, access modes match
```

---

## Next Lab

Proceed to [Lab 09: DaemonSets](../k8s-training/09-daemonset.yaml) - Running pods on every node.
