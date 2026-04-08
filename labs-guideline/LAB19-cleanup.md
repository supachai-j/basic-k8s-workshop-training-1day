# Lab 19: Cleanup - Remove All Training Resources

## Objective
Learn how to properly clean up all resources created during the training labs.

---

## Overview

After completing the labs, it's important to clean up resources to:
- Free up cluster resources
- Avoid resource conflicts in future sessions
- Maintain a clean training environment

---

## Step 1: List All Training Resources

Before deleting, let's see what we created:

```bash
# List all resources in training namespace
kubectl get all -n training

# List persistent volume claims
kubectl get pvc -n training

# List configmaps and secrets
kubectl get configmap,secret -n training

# List persistent volumes (may have training PVs)
kubectl get pv | grep training
```

---

## Step 2: Delete All Workloads

```bash
# Delete all pods, services, deployments, etc.
kubectl delete all --all -n training

# Verify
kubectl get all -n training
```

---

## Step 3: Delete ConfigMaps and Secrets

```bash
# Delete configmaps
kubectl delete configmap --all -n training

# Delete secrets
kubectl delete secret --all -n training

# Verify
kubectl get configmap,secret -n training
```

---

## Step 4: Delete PersistentVolumeClaims

```bash
# List PVCs
kubectl get pvc -n training

# Delete all PVCs (WARNING: This deletes data!)
kubectl delete pvc --all -n training

# Verify
kubectl get pvc -n training
```

---

## Step 5: Delete PersistentVolumes (If Created Manually)

```bash
# List PVs
kubectl get pv | grep training

# Delete training PVs
kubectl delete pv training-manual-pv

# Verify
kubectl get pv | grep training
```

---

## Step 6: Delete Namespaces

```bash
# Delete the training namespace
kubectl delete namespace training

# Verify namespace is gone
kubectl get namespaces | grep training
```

---

## Step 7: Clean Up ClusterRoleBindings

```bash
# List clusterrolebindings related to training
kubectl get clusterrolebindings | grep training

# Delete training clusterrolebindings
kubectl delete clusterrolebinding training-cluster-viewer-binding
```

---

## Step 8: Clean Up ClusterRoles

```bash
# List clusterroles related to training
kubectl get clusterroles | grep training

# Delete training clusterroles
kubectl delete clusterrole training-cluster-viewer
```

---

## Step 9: Verify Complete Cleanup

```bash
# Check for any remaining training resources
echo "=== Namespaces ==="
kubectl get namespaces | grep training

echo "=== All resources in training namespace ==="
kubectl get all -n training 2>&1

echo "=== PVCs ==="
kubectl get pvc -n training 2>&1

echo "=== PVs ==="
kubectl get pv | grep training

echo "=== ClusterRoles ==="
kubectl get clusterroles | grep training

echo "=== ClusterRoleBindings ==="
kubectl get clusterrolebindings | grep training
```

---

## Step 10: Alternative - Use Lab's Cleanup File

The k8s-training folder includes a cleanup file:

```bash
# Navigate to k8s-training
cd ../k8s-training

# Apply cleanup file (if it exists)
kubectl apply -f 19-cleanup.yaml

# Or use the cleanup commands from README
kubectl delete -f .
```

---

## Quick Cleanup Commands Reference

```bash
# Single command cleanup (if everything is in training namespace)
kubectl delete namespace training

# Or delete resources one by one
kubectl delete deployment --all -n training
kubectl delete service --all -n training
kubectl delete configmap --all -n training
kubectl delete secret --all -n training
kubectl delete pvc --all -n training
kubectl delete job --all -n training
kubectl delete cronjob --all -n training
kubectl delete hpa --all -n training
kubectl delete networkpolicy --all -n training

# Finally, delete namespace
kubectl delete namespace training
```

---

## Verification Checklist

- [ ] No resources in training namespace
- [ ] Training namespace deleted (or empty)
- [ ] No training-related PVs remain
- [ ] No training-related ClusterRoles remain
- [ ] No training-related ClusterRoleBindings remain
- [ ] Cluster resources are freed

---

## Congratulations!

You've completed all 19 Kubernetes Training Labs! 🎉

**Topics covered:**
1. ✅ Namespaces - Logical isolation
2. ✅ Pods - Basic unit
3. ✅ Multi-container Pods - Sidecars
4. ✅ ReplicaSets - Pod replication
5. ✅ Deployments - Rolling updates
6. ✅ Services - Network exposure
7. ✅ ConfigMap & Secret - Configuration
8. ✅ PersistentVolume - Storage
9. ✅ DaemonSets - Node-level pods
10. ✅ Resource Management - Requests/limits
11. ✅ Health Probes - Liveness/Readiness
12. ✅ StatefulSets - Stateful apps
13. ✅ Jobs & CronJobs - Batch tasks
14. ✅ Ingress - HTTP routing
15. ✅ RBAC - Access control
16. ✅ HPA - Auto-scaling
17. ✅ Network Policies - Traffic control
18. ✅ Affinity & Taints - Pod placement
19. ✅ Cleanup - Resource management

---

## Next Steps

1. **Practice** - Run through labs again on your own
2. **Explore** - Set up a local cluster (k3s, minikube, kind)
3. **Learn more:**
   - Helm (package manager)
   - ArgoCD/Flux (GitOps)
   - Prometheus/Grafana (monitoring)
   - Istio/Linkerd (service mesh)
   - Vault (secrets management)

---

## Thank You!

**Training resources location:**
```
/Users/tumz/workspace/nttlab-trueminsert/
├── k8s-training/         # YAML lab files
├── presentation/         # Slide content
└── labs-guideline/       # Step-by-step guides
```
