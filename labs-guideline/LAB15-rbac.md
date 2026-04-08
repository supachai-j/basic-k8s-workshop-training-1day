# Lab 15: RBAC - Role-Based Access Control

## Objective
Learn how to use Kubernetes RBAC to control who can access what resources.

---

## What is RBAC?

RBAC (Role-Based Access Control) regulates access based on roles and permissions.

**Key concepts:**

| Object | Scope | Purpose |
|--------|-------|---------|
| **Role** | Namespace | Grants permissions within a namespace |
| **ClusterRole** | Cluster-wide | Grants permissions across all namespaces |
| **RoleBinding** | Namespace | Binds Role to subjects within namespace |
| **ClusterRoleBinding** | Cluster-wide | Binds ClusterRole to subjects |

**Subjects:** Users, Groups, ServiceAccounts

---

## Step 1: Create ServiceAccount

Create file `15-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: training
---
# Role defines what can be done
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-pod-reader
  namespace: training
rules:
  # Allow reading pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  # Allow reading pod logs
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
# RoleBinding connects Role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-pod-reader-binding
  namespace: training
subjects:
  - kind: ServiceAccount
    name: app-sa
    apiGroup: ""
roleRef:
  kind: Role
  name: app-pod-reader
  apiGroup: ""
```

Apply:

```bash
kubectl apply -f 15-rbac.yaml
```

---

## Step 2: Verify RBAC Objects

```bash
# List ServiceAccounts
kubectl get serviceaccounts -n training
kubectl get sa -n training

# List Roles
kubectl get roles -n training

# List RoleBindings
kubectl get rolebindings -n training
```

---

## Step 3: Test Permissions

Create a pod using the ServiceAccount and test access:

```bash
# Create a test pod with the ServiceAccount
kubectl run rbac-test --image=nginx:1.25-alpine \
  --serviceaccount=app-sa \
  -n training

# Try to list pods (should succeed)
kubectl auth can-i get pods --as=system:serviceaccount:training:app-sa -n training

# Try to create pods (should fail)
kubectl auth can-i create pods --as=system:serviceaccount:training:app-sa -n training
```

---

## Step 4: Create ClusterRole (Cluster-wide)

Add to `15-rbac.yaml`:

```yaml
---
# ClusterRole - works across all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: training-cluster-viewer
rules:
  # Read access to pods in all namespaces
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  # Read access to nodes
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding - connects ClusterRole to subjects
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: training-cluster-viewer-binding
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: training
roleRef:
  kind: ClusterRole
  name: training-cluster-viewer
  apiGroup: ""
```

Apply:

```bash
kubectl apply -f 15-rbac.yaml
```

---

## Step 5: Verify ClusterRoleBinding

```bash
# List ClusterRoles
kubectl get clusterroles

# List ClusterRoleBindings
kubectl get clusterrolebindings

# Describe
kubectl describe clusterrolebinding training-cluster-viewer-binding
```

---

## Step 6: Test Cluster-wide Access

```bash
# Check if can list pods in different namespace
kubectl auth can-i get pods --as=system:serviceaccount:training:app-sa -n kube-system

# Check if can list nodes
kubectl auth can-i get nodes --as=system:serviceaccount:training:app-sa
```

---

## Step 7: Test From Within Pod

```bash
# Get pod token
TOKEN=$(kubectl get secret -n training -o jsonpath='{.items[?(@.metadata.annotations.kubernetes\.io/service-account\.name=="app-sa")].data.token}' | base64 -d)

# Use token inside pod
kubectl exec rbac-test -n training -- \
  sh -c "cat /var/run/secrets/kubernetes.io/serviceaccount/token" | head -c 50

# Inside pod, query k8s API
kubectl exec rbac-test -n training -- \
  sh -c 'curl -sSk -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default/api/v1/namespaces/training/pods'
```

---

## Step 8: Verb Definitions

| Verb | Operations |
|------|------------|
| **get** | Single resource (GET) |
| **list** | Collection of resources (GET) |
| **watch** | Watch for changes (WS) |
| **create** | Create new resource (POST) |
| **update** | Update existing (PUT) |
| **patch** | Partial update (PATCH) |
| **delete** | Delete resource (DELETE) |
| **deletecollection** | Delete multiple (DELETE) |

---

## Step 9: Resource Types

```yaml
rules:
  # Core resources
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]

  # Apps resources
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]

  # Networking
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]

  # Batch
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
```

---

## Step 10: Aggregation (ClusterRole labels)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aggregated-reader
  labels:
    rbac.example.com/aggregate-to-view: "true"
rules:
  # This ClusterRole will automatically aggregate
  # to any ClusterRole with label rbac.example.com/aggregate-to-view: "true"
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```

---

## Step 11: Clean up RBAC Objects

```bash
# Delete in order (bindings first, then roles)
kubectl delete rolebinding app-pod-reader-binding -n training
kubectl delete role app-pod-reader -n training
kubectl delete clusterrolebinding training-cluster-viewer-binding
kubectl delete clusterrole training-cluster-viewer
kubectl delete serviceaccount app-sa -n training
kubectl delete pod rbac-test -n training
```

---

## Verification Checklist

- [ ] ServiceAccount created
- [ ] Role grants correct permissions within namespace
- [ ] RoleBinding connects SA to Role
- [ ] ClusterRole grants cluster-wide permissions
- [ ] ClusterRoleBinding connects SA to ClusterRole
- [ ] `kubectl auth can-i` shows correct permissions
- [ ] Pod using SA can/cannot access resources as expected

---

## Key Commands Summary

```bash
# ServiceAccounts
kubectl get sa -n <namespace>
kubectl describe sa <name> -n <namespace>

# Roles/ClusterRoles
kubectl get roles -n <namespace>
kubectl get clusterroles
kubectl describe role <name> -n <namespace>

# RoleBindings/ClusterRoleBindings
kubectl get rolebindings -n <namespace>
kubectl get clusterrolebindings
kubectl describe rolebinding <name> -n <namespace>

# Check permissions
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<namespace>:<sa-name>

# Delete
kubectl delete rolebinding <name> -n <namespace>
kubectl delete role <name> -n <namespace>
kubectl delete clusterrolebinding <name>
kubectl delete clusterrole <name>
```

---

## RBAC Best Practices

1. **Least privilege** - Grant minimum permissions needed
2. **Use ServiceAccounts** - Not user accounts for applications
3. **Namespace isolation** - Use Roles, not ClusterRoles when possible
4. **Aggregate ClusterRoles** - Use labels for reusable roles
5. **Audit regularly** - Check who has what permissions

---

## Next Lab

Proceed to [Lab 16: HPA](../k8s-training/16-hpa.yaml) - Horizontal Pod Autoscaler.
