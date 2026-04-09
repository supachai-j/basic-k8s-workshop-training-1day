# Lab 21: Kubernetes CNI - Network Basics with Flannel

## Objective
Understand how Kubernetes networking works and learn about CNI (Container Network Interface) plugins using Flannel as an example.

---

## What is CNI?

**CNI (Container Network Interface)** is a standard that defines how container runtimes (like containerd, cri-o) communicate with network plugins.

**CNI Responsibilities:**
- Assign IP addresses to pods
- Connect pods to the network
- Manage network namespaces
- Clean up network resources when containers are deleted

**Popular CNI Plugins:**
- **Flannel** - Simple, Layer 3 network (VXLAN)
- **Calico** - More features, Layer 3 with network policies
- **Weave** - Multi-host networking
- **Cilium** - eBPF-based, high performance

---

## What is Flannel?

**Flannel** is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

**Key features:**
- Allocates a subnet lease to each host out of a preconfigured address space
- Uses VXLAN backend by default
- Stores network configuration in Kubernetes API (or etcd)
- Does NOT control container-to-host networking, only host-to-host traffic

**Official source:** https://github.com/flannel-io/flannel

---

## Step 1: Install Flannel (Reference)

For Kubernetes v1.17+, install Flannel using:

```bash
# Standard installation
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# If using custom podCIDR (not 10.244.0.0/16), first create namespace:
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Or using Helm
helm repo add flannel https://flannel-io.github.io/flannel/
helm install flannel --set podCidr="10.42.0.0/16" --namespace kube-flannel flannel/flannel
```

> **Note:** Flannel runs in the `kube-flannel` namespace (NOT `kube-system`).

---

## Step 2: Check Current CNI Configuration

```bash
# Check CNI binaries directory
ls -la /etc/cni/net.d/

# View current CNI config
cat /etc/cni/net.d/10-flannel.conflist
```

---

## Step 3: Check Nodes and Cluster Info

```bash
# Get cluster nodes
kubectl get nodes -o wide

# Check node IP addresses
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

---

## Step 4: Check Flannel Pods and Namespace

```bash
# Flannel runs in kube-flannel namespace (NOT kube-system)
kubectl get pods -n kube-flannel

# Check Flannel logs
kubectl logs -n kube-flannel -l app=flannel --tail=20

# List all resources in kube-flannel namespace
kubectl get all -n kube-flannel
```

---

## Step 5: Check Pod Network Range

```bash
# Check Flannel ConfigMap
kubectl get configmap -n kube-flannel

# View Flannel configuration
kubectl describe configmap -n kube-flannel kube-flannel-cfg

# Or view the full configmap
kubectl get configmap -n kube-flannel -o yaml
```

---

## Step 6: Understand Pod Networking

When a pod is created, the CNI plugin:
1. Creates a network namespace
2. Creates a virtual ethernet (veth) pair
3. Assigns an IP address from the pod CIDR
4. Connects the pod to the network

```bash
# Create a test pod to explore networking
kubectl run test-pod --image=busybox:1.36 -n training --restart=Never -- sleep 3600

# Wait for pod to be ready
kubectl get pods test-pod -n training

# Check pod details including IP
kubectl get pod test-pod -n training -o wide
```

---

## Step 7: Inspect Pod Network Namespace

On a node (if you have direct access):

```bash
# Get the pod's node
kubectl get pod test-pod -n training -o jsonpath='{.spec.nodeName}'

# SSH to the node (if possible)
# Find the pod's process ID
# crictl inspect <pod-id> | grep pid

# Check network interfaces on the node
# ip addr show

# Inside the pod namespace:
# ip addr
# The pod will have an eth0 interface with its pod IP
```

---

## Step 8: Test Pod-to-Pod Communication

```bash
# Get pod IP addresses
kubectl get pods -n training -o wide

# Create another test pod
kubectl run test-pod-2 --image=busybox:1.36 -n training --restart=Never -- sleep 3600

# Wait for it to be ready
kubectl get pods -n training -l run=test-pod

# From inside a pod, test connectivity to another pod
kubectl exec test-pod -n training -- ping -c 3 <other-pod-ip>
```

---

## Step 9: Test DNS Resolution

```bash
# Check if CoreDNS is running (CoreDNS is in kube-system)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# From a pod, test DNS resolution
kubectl exec test-pod -n training -- nslookup kubernetes.default

# Test service name resolution
kubectl exec test-pod -n training -- nslookup nginx.training.svc.cluster.local
```

---

## Step 10: Understand Services and Network Policies

Services provide stable IP addresses for pods:

```bash
# Create a service
kubectl expose pod test-pod -n training --port=80 --name=test-service

# Check service
kubectl get svc test-service -n training

# Service IP is different from pod IP
kubectl get pods -n training -l run=test-pod -o wide
```

---

## Step 11: Flannel Architecture

Flannel works by creating a **VXLAN overlay network**:

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                   │
│                                                         │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐  │
│   │ Node 1  │◄───────►│ Node 2  │◄───────►│ Node 3  │  │
│   │10.200.0.0/24│   │10.200.1.0/24│   │10.200.2.0/24│  │
│   └─────────┘         └─────────┘         └─────────┘  │
│        │                   │                   │        │
│   ┌────┴────┐         ┌────┴────┐         ┌────┴────┐  │
│   │ Flannel │         │ Flannel │         │ Flannel │  │
│   │ (VXLAN) │◄───────►│ (VXLAN) │◄───────►│ (VXLAN) │  │
│   └─────────┘         └─────────┘         └─────────┘  │
│                                                         │
│   Namespace: kube-flannel                               │
└─────────────────────────────────────────────────────────┘

Pod Network: 10.42.0.0/16
Node Subnets: 10.200.0.0/24, 10.200.1.0/24, 10.200.2.0/24
```

---

## Step 12: Network Communication Paths

**Same Node Pod-to-Pod:**
```
Pod A (eth0) ←→ veth pair ←→ cni0 (bridge) ←→ Pod B (eth0)
```

**Different Node Pod-to-Pod:**
```
Pod A (eth0) ←→ veth ←→ cni0 ←→ flannel.1 (VXLAN) ←→ eth0 ←→ Network ←→ eth0 ←→ flannel.1 ←→ cni0 ←→ veth ←→ Pod B
```

---

## Step 13: Check Network Routes

On a node (if you have access):

```bash
# Check routing table
# ip route

# You should see routes like:
# 10.200.0.0/24 via 10.200.0.x dev flannel.1
# 10.200.1.0/24 via 10.200.1.x dev flannel.1
```

---

## Step 14: Test External Connectivity

```bash
# From a pod, test internet connectivity
kubectl exec test-pod -n training -- wget -qO- google.com

# Test DNS to external addresses
kubectl exec test-pod -n training -- nslookup www.google.com
```

---

## Step 15: Network Policy Basics

Network policies control pod-to-pod traffic:

```bash
# Create a network policy to isolate a pod
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-nginx
  namespace: training
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
    - Egress
EOF

# List network policies
kubectl get networkpolicies -n training
```

---

## Step 16: Cleanup

```bash
# Delete test pods
kubectl delete pod test-pod test-pod-2 -n training

# Delete test service
kubectl delete svc test-service -n training

# Delete network policy
kubectl delete networkpolicy isolate-nginx -n training
```

---

## Flannel Namespace vs kube-system

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| **Flannel** | `kube-flannel` | CNI network plugin |
| **CoreDNS** | `kube-system` | DNS service |
| **kube-proxy** | `kube-system` | Service networking |
| **kube-flannel-cfg** | `kube-flannel` | Flannel ConfigMap |

> **Important:** Flannel pods run in `kube-flannel` namespace, NOT `kube-system`.

---

## Verification Checklist

- [ ] Understand CNI role in Kubernetes networking
- [ ] Can explain how pods get IP addresses
- [ ] Verified pod-to-pod connectivity
- [ ] Tested DNS resolution
- [ ] Understand Flannel VXLAN overlay
- [ ] Know difference between pod IP and service IP
- [ ] Basic understanding of network policies
- [ ] Know Flannel runs in `kube-flannel` namespace

---

## CNI vs Kubernetes Networking

| Concept | Description |
|---------|-------------|
| **Pod IP** | Unique IP per pod, assigned by CNI |
| **Service IP** | Virtual IP for stable service endpoint |
| **ClusterIP** | Internal service access |
| **NodePort** | Expose service via node port |
| **LoadBalancer** | External load balancer integration |

---

## Key Commands Summary

```bash
# Check CNI config
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist

# Check Flannel (kube-flannel namespace!)
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel --tail=50
kubectl get configmap -n kube-flannel

# Check CoreDNS (kube-system namespace)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Network diagnostics in pod
kubectl exec <pod> -n <ns> -- ip addr
kubectl exec <pod> -n <ns> -- ip route
kubectl exec <pod> -n <ns> -- cat /etc/resolv.conf

# Test connectivity
kubectl exec <pod> -n <ns> -- ping -c 3 <target-ip>
kubectl exec <pod> -n <ns> -- nslookup <service-name>
kubectl exec <pod> -n <ns> -- wget -qO- <url>

# Network policies
kubectl get networkpolicies -n <ns>
kubectl describe networkpolicy <name> -n <ns>
```

---

## Common Issues

### Pod stuck in Pending
```bash
# Check if CNI plugin is working
kubectl get pods -n kube-flannel

# Check node conditions
kubectl describe node <node-name> | grep -A 5 Conditions
```

### Pod cannot reach other pods
```bash
# Check pod IP is assigned
kubectl get pod <pod> -n <ns> -o wide

# Check Flannel is running (in kube-flannel!)
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel -l app=flannel --tail=20
```

### DNS not working
```bash
# Check CoreDNS pods (in kube-system)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check DNS service
kubectl get svc kube-dns -n kube-system

# Test with full FQDN
kubectl exec <pod> -n <ns> -- nslookup kubernetes.default.svc.cluster.local
```

### Network policy blocking traffic
```bash
# List network policies
kubectl get networkpolicies -n <ns>

# Check policy rules
kubectl describe networkpolicy <name> -n <ns>
```

---

## Further Reading

- [Kubernetes Networking Explained](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Flannel GitHub](https://github.com/flannel-io/flannel)
- [Flannel Kubernetes Integration](https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CNI Specification](https://github.com/containernetworking/cni)

---


