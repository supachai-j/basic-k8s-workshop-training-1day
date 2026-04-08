# Lab 01: Namespaces - Logical Isolation

## Objective
Learn how Kubernetes Namespaces provide logical isolation for resources.

---

## What is a Namespace?

A Namespace is a way to divide cluster resources between multiple users, teams, or projects.

**Default Namespaces:**
- `default` - Default namespace for user resources
- `kube-system` - System resources (DNS, scheduler, etc.)
- `kube-public` - Public resources across cluster
- `kube-node-lease` - Node heartbeat data

---

## Prerequisites

```bash
# Verify cluster is accessible
kubectl cluster-info

# Confirm kubectl is working
kubectl get nodes
```

---

## Step 1: View Existing Namespaces

```bash
# List all namespaces
kubectl get namespaces

# Alternative command (older syntax)
kubectl get ns
```

Expected output:
```
NAME              STATUS   AGE
default           Active   30d
kube-node-lease   Active   30d
kube-public       Active   30d
kube-system       Active   30d
training          Active   2h    <-- Created in this lab
```

---

## Step 2: Create the Training Namespace

### Option A: Using YAML file

Create file `01-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: training
  labels:
    purpose: k8s-training
    team: nttlab
```

Apply:
```bash
kubectl apply -f 01-namespace.yaml
```

### Option B: Using kubectl command

```bash
kubectl create namespace training
```

---

## Step 3: Verify Namespace Creation

```bash
# List namespaces
kubectl get namespaces

# Describe the namespace
kubectl describe namespace training
```

Expected output:
```
Name:         training
Labels:       purpose=k8s-training
              team=nttlab
Annotations:  <none>
Status:       Active
```

---

## Step 4: Set Default Namespace (Optional)

### Option A: Using kubens (if installed)

```bash
# Install kubens (optional)
brew install kubens

# Switch default namespace
kubens training
```

### Option B: Using kubectl config

```bash
# Set default namespace for current context
kubectl config set-context --current --namespace=training

# Verify
kubectl config view --minify | grep namespace
```

### Option C: Use -n flag explicitly

```bash
# Every command will use -n training
kubectl get pods -n training
kubectl get services -n training
```

---

## Step 5: List Resources in Namespace

```bash
# Show all resources in training namespace
kubectl get all -n training

# List pods (empty initially)
kubectl get pods -n training

# List services
kubectl get svc -n training

# List deployments
kubectl get deployment -n training
```

---

## Step 6: Apply Resources to Namespace

Let's apply the Pod from Lab 02 to see namespace isolation:

```bash
# Apply a simple pod to training namespace
kubectl apply -f ../k8s-training/02-pod.yaml

# Verify pod is in training namespace
kubectl get pods -n training

# Verify it's NOT in default namespace
kubectl get pods -n default
```

---

## Step 7: Resource Quotas (Optional)

Namespaces can have resource quotas to limit resource usage:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: training-quota
  namespace: training
spec:
  hard:
    pods: "20"
    services: "10"
    cpu: "2"
    memory: 4Gi
```

Apply:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: training-quota
  namespace: training
spec:
  hard:
    pods: "20"
    services: "10"
    cpu: "2"
    memory: 4Gi
EOF
```

Check quota:
```bash
kubectl get resourcequota -n training
kubectl describe resourcequota training-quota -n training
```

---

## Step 8: Cleanup (DO NOT RUN - Save for Lab 19)

```bash
# We'll clean up at the end of all labs
# kubectl delete namespace training
```

---

## Verification Checklist

- [ ] `kubectl get namespaces` shows `training` namespace
- [ ] `kubectl describe namespace training` shows your labels
- [ ] Pods created with `-n training` appear in training namespace
- [ ] Resources are isolated from other namespaces

---

## Key Commands Summary

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create namespace <name>
kubectl apply -f <file.yaml>

# Describe namespace
kubectl describe namespace <name>

# Set default namespace
kubectl config set-context --current --namespace=<name>

# List resources in namespace
kubectl get all -n <namespace>
kubectl get pods -n <namespace>

# Delete namespace (deletes ALL resources inside!)
kubectl delete namespace <name>
```

---

## Key Concepts Learned

1. **Namespace** - Logical isolation boundary
2. **Resource Quota** - Limits resource consumption per namespace
3. **Label-based isolation** - Resources tagged with namespace
4. **Context switching** - `-n` flag or `kubens`

---

## Next Lab

Proceed to [Lab 02: Pods](../k8s-training/02-pod.yaml) - The smallest deployable unit.
