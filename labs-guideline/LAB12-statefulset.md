# Lab 12: StatefulSets - For Stateful Applications

## Objective
Learn how StatefulSets provide stable network identities and persistent storage for stateful applications.

---

## What is StatefulSet?

StatefulSet is a workload API object for managing stateful applications.

**Key characteristics:**
- **Stable network identity** - Predictable, persistent hostnames
- **Stable storage** - Persistent Volume per pod (PVC)
- **Ordered deployment/scaling** - Pods created/destroyed in order
- **Ordered graceful deletion** - Pods terminate in reverse order

**Use cases:**
- Databases (MySQL, PostgreSQL, MongoDB)
- Key-value stores (Redis, etcd)
- Message queues (Kafka, RabbitMQ)

---

## Step 1: Create StatefulSet

Create file `12-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: training
  labels:
    app: mysql
    lesson: "12"
spec:
  serviceName: mysql-headless    # Must match headless service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  podManagementPolicy: OrderedReady   # Pods created in order
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpassword"
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
  # Each pod gets its own PVC
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Apply:

```bash
kubectl apply -f 12-statefulset.yaml
```

---

## Step 2: Create Headless Service

```yaml
# Required for StatefulSet - provides stable DNS
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: training
spec:
  clusterIP: None    # Headless!
  selector:
    app: mysql
  ports:
    - port: 3306
```

Apply:

```bash
kubectl apply -f 12-statefulset.yaml
```

---

## Step 3: Verify StatefulSet

```bash
# List StatefulSets
kubectl get statefulsets -n training
kubectl get sts -n training

# Describe StatefulSet
kubectl describe statefulset mysql-statefulset -n training
```

---

## Step 4: Verify Pods (Ordered Creation)

```bash
# Watch pods be created in order
kubectl get pods -n training -l app=mysql -w

# After running, list pods
kubectl get pods -n training -l app=mysql -o wide
```

Expected order:
```
mysql-statefulset-0   Running   (created first)
mysql-statefulset-1   Running   (created second)
mysql-statefulset-2   Running   (created third)
```

---

## Step 5: Verify Stable Network Identity

Each pod has a stable, predictable hostname:

```bash
# Check pod hostnames
for i in 0 1 2; do
  kubectl exec mysql-statefulset-$i -n training -- tail -n 1 /etc/hosts | awk "{print \$2}"
done

# DNS names work too
# mysql-statefulset-0.mysql-headless.training.svc.cluster.local
# mysql-statefulset-1.mysql-headless.training.svc.cluster.local
```

---

## Step 6: Verify Persistent Volumes

Each pod has its own PVC:

```bash
# List PVCs
kubectl get pvc -n training | grep mysql

# Each pod has its own PVC
kubectl get pvc -n training -l app=mysql
```

Expected:
```
NAME                             STATUS   VOLUME
mysql-data-mysql-statefulset-0   Bound    ...
mysql-data-mysql-statefulset-1   Bound    ...
mysql-data-mysql-statefulset-2   Bound    ...
```

---

## Step 7: Write Data to One Pod

```bash
# Write to pod-0
kubectl exec mysql-statefulset-0 -n training -- \
  sh -c 'echo "Hello from MySQL Pod 0" > /var/lib/mysql/test.txt'

# Verify data in pod-0
kubectl exec mysql-statefulset-0 -n training -- cat /var/lib/mysql/test.txt
```

---

## Step 8: Test Pod Deletion (Data Persistence)

```bash
# Delete pod-0
kubectl delete pod mysql-statefulset-0 -n training

# Watch it be recreated (same name, new pod)
kubectl get pods -n training -l app=mysql -w

# Check data still exists (same PVC!)
kubectl exec mysql-statefulset-0 -n training -- cat /var/lib/mysql/test.txt
```

**Key insight:** Data persists because PVC is NOT deleted when pod is deleted.

---

## Step 9: Scale StatefulSet

```bash
# Scale up
kubectl scale statefulset mysql-statefulset -n training --replicas=4

# Scale down (pods terminated in reverse order: 3, 2, 1)
kubectl scale statefulset mysql-statefulset -n training --replicas=2
```

---

## Step 10: Rolling Update

```yaml
# Update strategy
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

```bash
# Update image
kubectl set image statefulset/mysql-statefulset mysql=mysql:8.0 -n training

# Watch update (pods updated in order: 0, 1, 2)
kubectl get pods -n training -l app=mysql -w
```

---

## Step 11: StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| **Pod Names** | Random hash | Stable ordinal (name-0, name-1) |
| **DNS** | Shared service | Headless + individual DNS |
| **Storage** | Shared or none | Each pod gets own PVC |
| **Creation Order** | Any order | Ordinal order |
| **Deletion Order** | Any order | Reverse ordinal |
| **Scaling** | Any pod replaced | Ordinal order |

---

## Step 12: Cleanup

```bash
# Delete StatefulSet (PVCs remain!)
kubectl delete statefulset mysql-statefulset -n training

# Delete PVCs (data will be lost!)
kubectl delete pvc -n training -l app=mysql

# Delete service
kubectl delete service mysql-headless -n training
```

---

## Verification Checklist

- [ ] StatefulSet created with 3 replicas
- [ ] Pods created in ordinal order (0, 1, 2)
- [ ] Each pod has unique, stable hostname
- [ ] Each pod has its own PVC
- [ ] Deleting a pod preserves PVC (data persists)
- [ ] New pod attaches to same PVC
- [ ] Scaling works correctly
- [ ] Cleanup removes pods and optionally PVCs

---

## Key Commands Summary

```bash
# Create/Apply
kubectl apply -f 12-statefulset.yaml

# List/Describe
kubectl get statefulsets -n training
kubectl get sts -n training
kubectl describe statefulset <name> -n training

# Scale
kubectl scale statefulset <name> -n training --replicas=3

# Update image
kubectl set image statefulset/<name> <container>=<image:tag> -n training

# Delete (PVCs NOT deleted automatically)
kubectl delete statefulset <name> -n training

# PVC operations
kubectl get pvc -n training
kubectl delete pvc <pvc-name> -n training
```

---

## Troubleshooting

### Pod stuck in Terminating
```bash
# Force delete (careful with StatefulSets!)
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### StatefulSet not creating pods
```bash
# Check service name matches spec.serviceName
kubectl describe statefulset <name> -n training | grep "Service Name"
```

---

## Next Lab

Proceed to [Lab 13: Jobs & CronJobs](../k8s-training/13-job-cronjob.yaml) - Batch task execution.
