# LAB00 - Local Lab Environment Setup

## Objective
Set up a local Kubernetes cluster for hands-on practice labs.

---

## Overview

This lab guides you through setting up a local Kubernetes environment using one of the following options:

| Option | Best For | Resource Usage |
|--------|----------|----------------|
| **Minikube** | macOS/Linux users | Medium |
| **Docker Desktop** | Docker users already | Medium |
| **Podman** | Podman fans | Medium |
| **K3s** | Lightweight/Single-node | Low |

---

## Option 1: Minikube

### Prerequisites
- macOS, Linux, or Windows with WSL2
- 2+ CPUs, 2GB+ RAM, 20GB+ disk
- Docker or container runtime installed

### Installation

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --driver=docker --cpus=2 --memory=4g

# Enable ingress addon
minikube addons enable ingress

# Verify
kubectl get nodes
kubectl get pods -A
```

### Stop/Start Cluster

```bash
# Stop cluster
minikube stop

# Start cluster
minikube start

# Delete cluster
minikube delete
```

---

## Option 2: Docker Desktop

### Prerequisites
- Docker Desktop installed (v4.0+)
- 4GB+ RAM allocated to Docker

### Installation

```bash
# Enable Kubernetes in Docker Desktop
# 1. Open Docker Desktop
# 2. Settings → Kubernetes
# 3. Enable "Enable Kubernetes"
# 4. Click "Apply & Restart"

# Verify
kubectl get nodes
kubectl get pods -A
```

### Configure kubectl

```bash
# Docker Desktop auto-configures kubectl
# Just verify
kubectl config current-context
kubectl get nodes
```

---

## Option 3: Podman

### Prerequisites
- Podman installed (v4.0+)
- 4GB+ RAM

### Installation

```bash
# macOS
brew install podman
podman machine init --cpus=2 --memory=4096
podman machine start

# Linux
sudo apt-get install podman

# Start Kubernetes (using kind or minikube with podman driver)
brew install kind
kind create cluster --name k8s-lab

# Verify
kubectl get nodes
```

---

## Option 4: K3s (Lightweight)

### Prerequisites
- Single node (VM or bare metal)
- Ubuntu 20.04+ or macOS

### Installation

```bash
# Ubuntu/Linux
curl -sfL https://get.k3s.io | sh -

# Verify
kubectl get nodes
sudo k3s kubectl get pods -A

# For macOS, use k3d (K3s in Docker)
brew install k3d
k3d cluster create mycluster --agents 1
kubectl config use-context k3d-mycluster
```

---

## Verify Your Setup

### Check Cluster Status

```bash
# List nodes
kubectl get nodes

# Expected output:
# NAME             STATUS   ROLES                  AGE   VERSION
# minikube         Ready    control-plane,master  5m    v1.28.0

# List all pods
kubectl get pods -A

# Expected output:
# NAMESPACE     NAME                               READY   STATUS
# kube-system   coredns-xxxxx                      1/1     Running
# kube-system   etcd-minikube                      1/1     Running
# kube-system   kube-apiserver-minikube             1/1     Running
# kube-system   kube-controller-manager-minikube   1/1     Running
# kube-system   kube-proxy-xxxxx                   1/1     Running
# kube-system   storage-provisioner                 1/1     Running
```

### Check kubectl Context

```bash
# Show current context
kubectl config current-context

# Show all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>
```

---

## Install Additional Tools

### kubectl Autocomplete

```bash
# bash
source <(kubectl completion bash)

# zsh
source <(kubectl completion zsh)

# Add to ~/.bashrc or ~/.zshrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

### helm (for LAB20)

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

### Additional kubectl plugins

```bash
# Install krew (kubectl plugin manager)
(
  set -x; cd "$(mktemp -d)"
  OS="$(uname | tr '[:upper:]' '[:lower:]')"
  Arch="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')"
  KREW="krew-${OS}_${Arch}"
  curl -fsSL "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" | tar xzvf -
  ./"${KREW}" install krew
)
export PATH="${KREW_PATH:-$HOME/.krew/bin}:$PATH"
kubectl krew install ctx
kubectl krew install ns
```

---

## Troubleshooting

### Minikube Issues

```bash
# Check minikube status
minikube status

# View logs
minikube logs

# Restart cluster
minikube delete && minikube start --driver=docker
```

### Docker Desktop Issues

```bash
# Reset Kubernetes
# Docker Desktop → Settings → Kubernetes → Reset Kubernetes Cluster

# Check Docker status
docker info
```

### kubectl Issues

```bash
# Recreate kubectl config
rm ~/.kube/config
kubectl config view

# For Docker Desktop, export config
kubectl config view --flatten
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `kubectl get nodes` | List all nodes |
| `kubectl get pods -A` | List all pods |
| `kubectl cluster-info` | Show cluster endpoint |
| `minikube status` | Minikube status |
| `minikube stop` | Stop cluster |
| `minikube start` | Start cluster |

---

## Next Steps

✅ Lab environment is ready! Proceed to **LAB01 - Namespace**

---

*Last updated: 2026-04-12*
