# Kubernetes Training Lab Guidelines

## Overview

This folder contains step-by-step lab guides for learning Kubernetes basics.

**Lab Structure:** 22 labs progressing from environment setup to advanced topics.

---

## Prerequisites Check

Before starting labs, verify your environment:

```bash
# 1. Check Kubernetes cluster is running
kubectl cluster-info

# 2. Verify nodes are ready
kubectl get nodes

# 3. Check kubectl version
kubectl version --client

# 4. Verify kubectl can communicate with cluster
kubectl get all
```

Expected output for `kubectl get nodes`:
```
NAME        STATUS   ROLES           AGE   VERSION
k8s-node    Ready    control-plane   10d   v1.29.x
```

---

## ⚠️ Start Here: LAB00 - Environment Setup

**Before doing any labs, you must complete LAB00 to set up your local Kubernetes environment.**

Choose one option: **Minikube**, **Docker Desktop**, **Podman**, or **K3s**

📄 See: [LAB00-setup.md](LAB00-setup.md)

---

## Lab Environment Variables

Set up convenient aliases for training:

```bash
# Add to ~/.bashrc or ~/.zshrc
export NS="training"
alias kns='kubectl config set-context --current --namespace=$NS'
alias kgp='kubectl get pods -n $NS'
alias kgs='kubectl get svc -n $NS'
alias kgd='kubectl get deployment -n $NS'
alias kga='kubectl get all -n $NS'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'

# Reload shell
source ~/.bashrc
```

---

## Lab Index

| Lab | File | Topic | Estimated Time |
|-----|------|-------|----------------|
| **00** | **[LAB00-setup.md](LAB00-setup.md)** | **Local Lab Setup** | **30 min** |
| 01 | [LAB01-namespace.md](LAB01-namespace.md) | Namespaces | 10 min |
| 02 | [LAB02-pod.md](LAB02-pod.md) | Pods | 15 min |
| 03 | [LAB03-multi-container.md](LAB03-multi-container.md) | Multi-Container Pods | 15 min |
| 04 | [LAB04-replicaset.md](LAB04-replicaset.md) | ReplicaSets | 10 min |
| 05 | [LAB05-deployment.md](LAB05-deployment.md) | Deployments | 20 min |
| 06 | [LAB06-service.md](LAB06-service.md) | Services | 20 min |
| 07 | [LAB07-configmap-secret.md](LAB07-configmap-secret.md) | ConfigMap & Secret | 20 min |
| 08 | [LAB08-persistent-volume.md](LAB08-persistent-volume.md) | PersistentVolume | 20 min |
| 09 | [LAB09-daemonset.md](LAB09-daemonset.md) | DaemonSets | 15 min |
| 10 | [LAB10-resource-management.md](LAB10-resource-management.md) | Resource Management | 20 min |
| 11 | [LAB11-probes.md](LAB11-probes.md) | Health Probes | 20 min |
| 12 | [LAB12-statefulset.md](LAB12-statefulset.md) | StatefulSets | 20 min |
| 13 | [LAB13-job-cronjob.md](LAB13-job-cronjob.md) | Jobs & CronJobs | 20 min |
| 14 | [LAB14-ingress.md](LAB14-ingress.md) | Ingress | 25 min |
| 15 | [LAB15-rbac.md](LAB15-rbac.md) | RBAC | 20 min |
| 16 | [LAB16-hpa.md](LAB16-hpa.md) | Horizontal Pod Autoscaler | 25 min |
| 17 | [LAB17-network-policy.md](LAB17-network-policy.md) | Network Policies | 20 min |
| 18 | [LAB18-affinity-taint.md](LAB18-affinity-taint.md) | Affinity & Taints | 25 min |
| 19 | [LAB19-cleanup.md](LAB19-cleanup.md) | Cleanup | 5 min |
| 20 | [LAB20-helm.md](LAB20-helm.md) | Helm Package Manager | 25 min |
| 21 | [LAB21-cni-flannel.md](LAB21-cni-flannel.md) | CNI (Flannel) | 20 min |

---

## Lab Order Flow

```
START
  │
  ▼
Lab 00: Local Environment Setup
  │  (Minikube / Docker Desktop / Podman / K3s)
  │
  ▼
Lab 01: Namespace
  │
  ▼
Lab 02: Pod
  │
  ▼
Lab 03: Multi-Container Pods
  │
  ▼
Lab 04: ReplicaSet
  │
  ▼
Lab 05: Deployment
  │
  ▼
Lab 06: Service ──► Lab 14: Ingress
  │                      │
  ▼                      │
Lab 07: ConfigMap/Secret ◄──┘
  │
  ▼
Lab 08: PersistentVolume
  │
  ▼
Lab 09: DaemonSet
  │
  ▼
Lab 10: Resource Management
  │
  ▼
Lab 11: Probes
  │
  ▼
Lab 12: StatefulSet
  │
  ▼
Lab 13: Job/CronJob
  │
  ▼
Lab 15: RBAC
  │
  ▼
Lab 16: HPA
  │
  ▼
Lab 17: Network Policy
  │
  ▼
Lab 18: Affinity/Taints
  │
  ▼
Lab 19: Cleanup
  │
  ▼
Lab 20: Helm
  │
  ▼
Lab 21: CNI (Flannel)
  │
  ▼
END
```

---

## Common kubectl Commands Reference

```bash
# Apply resources
kubectl apply -f <file.yaml>
kubectl apply -f .                    # Apply all in current directory

# List resources
kubectl get pods
kubectl get pods -n <namespace>
kubectl get all -n <namespace>
kubectl get services
kubectl get deployments
kubectl get replicasets
kubectl get ingress
kubectl get pvc
kubectl get pv

# Describe resources (detailed info)
kubectl describe pod <pod-name>
kubectl describe service <service-name>
kubectl describe deployment <deployment-name>

# View resource YAML
kubectl get pod <pod-name> -o yaml
kubectl get service <service-name> -o yaml

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f                    # Follow logs
kubectl logs <pod-name> --previous           # Previous container logs

# Exec into container
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# Delete resources
kubectl delete -f <file.yaml>
kubectl delete pod <pod-name>
kubectl delete service <service-name>
kubectl delete deployment <deployment-name>
kubectl delete namespace <namespace>

# Scale
kubectl scale deployment <deployment-name> --replicas=3

# Watch resources
kubectl get pods -w
kubectl get pods -n <namespace> --watch

# Port forward (access pod locally)
kubectl port-forward pod/<pod-name> 8080:80

# Resource usage
kubectl top pods
kubectl top nodes
```

---

## Troubleshooting Common Issues

### Pod Issues

```bash
# Pod stuck in Pending
kubectl describe pod <pod-name>
# Check: resources available, node selectors, taints

# Pod stuck in ContainerCreating
kubectl describe pod <pod-name>
# Check: image pulled, volume mounted correctly

# Pod in CrashLoopBackOff
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
# Check: application startup, health checks

# ImagePullBackOff
kubectl describe pod <pod-name>
# Check: image name correct, registry accessible
```

### Service Issues

```bash
# Service cannot find pods
kubectl describe service <service-name>
# Check: selector labels match pod labels

# Cannot access service
kubectl get endpoints <service-name>
# If 0 endpoints: selector mismatch or pods not running
```

### Network Issues

```bash
# Test DNS resolution
kubectl run dnsutils --rm -it --image=tutum/dnsutils -n <ns> -- /bin/sh
# Inside container: nslookup <service-name>

# Test connectivity
kubectl run curl-test --rm -it --image=curlimages/curl -n <ns> -- sh
# Inside container: curl http://<service-name>
```

---

## Tips for Success

1. **Complete LAB00 first** - Set up your local Kubernetes environment
2. **Read the YAML files** - Understand what each field does before applying
3. **Check status** - Always run `kubectl get pods` after applying
4. **Use describe** - `kubectl describe` gives detailed error messages
5. **Check logs** - Application logs often explain failures
6. **Start fresh** - Run cleanup between labs if issues persist
7. **Take notes** - Record commands that work well for you

---

## Next Steps After Training

1. **Practice** - Run these labs on your own cluster
2. **Explore** - Set up a local cluster with k3s or minikube
3. **Learn Helm** - Package manager for Kubernetes
4. **Study RBAC** - Security and access control
5. **Try GitOps** - ArgoCD or Flux for deployments
6. **Explore Operators** - Kubernetes Operators pattern

---

## Lab Duration

| Session | Labs | Time |
|---------|------|------|
| Setup | Lab 00 | 30 min |
| Morning Block 1 | Lab 01-05 | 2.5 hours |
| Morning Block 2 | Lab 07 | 1 hour |
| Afternoon Block 1 | Lab 06, Lab 14 | 1.5 hours |
| Afternoon Block 2 | Lab 08, Lab 12, Lab 13 | 2 hours |
| Afternoon Block 3 | Lab 11, Lab 16 | 1 hour |
| Afternoon Block 4 | Lab 17, Lab 18 | 1.5 hours |
| Final Block | Lab 20, Lab 21 | 1 hour |
| **Total** | **All 22 labs** | **~10 hours** |
