# Basic Kubernetes Workshop Training - 1 Day

A hands-on Kubernetes workshop covering core concepts and practical labs for beginners.

---

## 📋 Course Overview

This workshop provides a comprehensive introduction to Kubernetes fundamentals through 21 practical labs. Participants will learn by doing, deploying real applications and managing Kubernetes resources.

---

## 📚 Lab Contents

| Lab | Topic | Description |
|-----|-------|-------------|
| **LAB00** | **Setup** | **Local Lab Environment Setup (Minikube/Docker/Podman/K3s)** |
| LAB01 | Namespace | Create and manage Kubernetes namespaces |
| LAB02 | Pod | Deploy and manage Pods - the smallest deployable units |
| LAB03 | Multi-container | Work with multi-container Pod patterns |
| LAB04 | ReplicaSet | Ensure high availability with ReplicaSet |
| LAB05 | Deployment | Manage application rolling updates and rollbacks |
| LAB06 | Service | Expose applications with Kubernetes Services |
| LAB07 | ConfigMap & Secret | Manage configuration and sensitive data |
| LAB08 | Persistent Volume | Add persistent storage to applications |
| LAB09 | DaemonSet | Deploy logging and monitoring agents on every node |
| LAB10 | Resource Management | Configure CPU/memory resource limits |
| LAB11 | Probes | Implement health checks (liveness, readiness, startup) |
| LAB12 | StatefulSet | Manage stateful applications |
| LAB13 | Job & CronJob | Run batch jobs and scheduled tasks |
| LAB14 | Ingress | Configure HTTP routing with Ingress controller |
| LAB15 | RBAC | Secure access with Role-Based Access Control |
| LAB16 | HPA | Auto-scale applications based on metrics |
| LAB17 | Network Policy | Control network traffic between pods |
| LAB18 | Affinity, Taint & Toleration | Control pod scheduling |
| LAB19 | Cleanup | Clean up resources after lab completion |
| LAB20 | Helm | Package manager for Kubernetes applications |
| LAB21 | CNI (Flannel) | Container Network Interface with Flannel |

---

## ✅ Prerequisites

- Basic Linux command line knowledge
- Docker fundamentals understanding
- Access to a Kubernetes cluster (v1.24+)
- kubectl installed and configured

---

## 🛠 Environment Setup

### Required Tools

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# helm (for LAB20)
brew install helm
```

### Verify Installation

```bash
kubectl version --client
helm version
```

---

## 📖 How to Use

1. Navigate to `labs-guideline/` folder
2. Read the README.md in each lab folder for instructions
3. Follow the step-by-step exercises
4. Complete the verification commands to check your work
5. Clean up resources using LAB19 when done

---

## 📁 Repository Structure

```
basic-k8s-workshop-training-1day/
├── README.md                 # This file
├── labs-guideline/
│   ├── README.md           # Lab guidelines overview
│   ├── **LAB00-setup.md**  # **Setup local lab environment**
│   ├── LAB01-namespace.md
│   ├── LAB02-pod.md
│   ├── LAB03-multi-container.md
│   ├── LAB04-replicaset.md
│   ├── LAB05-deployment.md
│   ├── LAB06-service.md
│   ├── LAB07-configmap-secret.md
│   ├── LAB08-persistent-volume.md
│   ├── LAB09-daemonset.md
│   ├── LAB10-resource-management.md
│   ├── LAB11-probes.md
│   ├── LAB12-statefulset.md
│   ├── LAB13-job-cronjob.md
│   ├── LAB14-ingress.md
│   ├── LAB15-rbac.md
│   ├── LAB16-hpa.md
│   ├── LAB17-network-policy.md
│   ├── LAB18-affinity-taint.md
│   ├── LAB19-cleanup.md
│   ├── LAB20-helm.md
│   └── LAB21-cni-flannel.md
└── .gitignore
```

---

## 🎯 Workshop Goals

By the end of this workshop, you will be able to:

- [ ] Deploy and manage applications on Kubernetes
- [ ] Configure networking and services
- [ ] Implement storage solutions
- [ ] Secure applications with RBAC and Network Policies
- [ ] Scale applications automatically
- [ ] Use Helm for application deployment

---

## 📝 Notes

- Labs should be completed in order (LAB01 → LAB21)
- Each lab builds upon concepts from previous labs
- Always clean up resources after completing labs to avoid resource leaks

---

## 🔗 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [Helm Documentation](https://helm.sh/docs/)

---

*Last updated: 2026-04-12*
