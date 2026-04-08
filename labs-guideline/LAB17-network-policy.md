# Lab 17: Network Policies - Pod-to-Pod Traffic Control

## Objective
Learn how to restrict network traffic between pods using Network Policies.

---

## What are Network Policies?

Network Policies are Kubernetes resources that control traffic to/from pods. They act as a firewall for pods.

**Key concepts:**
- **Ingress** - Inbound traffic to pod
- **Egress** - Outbound traffic from pod
- **Pod selectors** - Select which pods are affected
- **Namespace selectors** - Select which namespaces
- **Port specifications** - Specify ports and protocols

**Default behavior:** All traffic is ALLOWED when no policy exists

---

## Step 1: Prerequisites

```bash
# Check if NetworkPolicy is supported
kubectl get networkpolicies -n training

# Check CNI plugin supports policies
kubectl get nodes -o wide
```

**Note:** Network Policies require a CNI plugin that supports them:
- Calico
- Cilium
- Weave Net
- OVN-Kubernetes

---

## Step 2: Create Test Deployment

```bash
# Create a test nginx deployment
kubectl run nginx --image=nginx:1.25-alpine -n training
kubectl run client --image=busybox:1.36 -n training

# Expose nginx
kubectl expose nginx --port=80 -n training
```

---

## Step 3: Test Default Connectivity (Before Policy)

```bash
# From client pod, curl nginx (should work)
kubectl exec client -n training -- wget -qO- http://nginx.training:80 | head -5

# This should succeed - no policy exists yet
```

---

## Step 4: Create Deny-All Ingress Policy

Create file `17-network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: training
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Apply:

```bash
kubectl apply -f 17-network-policy.yaml
```

---

## Step 5: Verify Policy Blocks Traffic

```bash
# Try to access nginx from client (should fail now)
kubectl exec client -n training -- wget -qO- http://nginx.training:80

# Verify pods without policy are still accessible
kubectl run other-nginx --image=nginx:1.25-alpine -n training
kubectl expose other-nginx --port=80 -n training

# This one should still work (not affected by policy)
kubectl exec client -n training -- wget -qO- http://other-nginx.training:80
```

---

## Step 6: Allow Specific Ingress Traffic

Update `17-network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-ingress
  namespace: training
spec:
  podSelector:
    matchLabels:
      run: nginx
  policyTypes:
    - Ingress
  ingress:
    # Allow traffic from client pod
    - from:
        - podSelector:
            matchLabels:
              run: client
      ports:
        - protocol: TCP
          port: 80
```

Apply:

```bash
kubectl apply -f 17-network-policy.yaml
```

---

## Step 7: Verify Selective Access

```bash
# From client pod (should work)
kubectl exec client -n training -- wget -qO- http://nginx.training:80 | head -3

# From other pod (should fail)
kubectl exec other-nginx -n training -- wget -qO- http://nginx.training:80
```

---

## Step 8: Create Egress Policy

Add to `17-network-policy.yaml`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: training
spec:
  podSelector:
    matchLabels:
      run: client
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    # Allow traffic to nginx
    - to:
        - podSelector:
            matchLabels:
              run: nginx
      ports:
        - protocol: TCP
          port: 80
```

Apply:

```bash
kubectl apply -f 17-network-policy.yaml
```

---

## Step 9: Test Egress Rules

```bash
# Check DNS works from client
kubectl exec client -n training -- nslookup kubernetes.default

# Try external access (should be blocked by default egress policy)
kubectl exec client -n training -- wget -qO- http://8.8.8.8
```

---

## Step 10: Namespace-based Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: training
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## Step 11: IP-based Policies

```yaml
ingress:
  - from:
      - ipBlock:
          cidr: 192.168.1.0/24
          except:
            - 192.168.1.5/32
    ports:
      - protocol: TCP
        port: 80
```

---

## Step 12: Cleanup

```bash
kubectl delete -f 17-network-policy.yaml
kubectl delete pod nginx client other-nginx -n training
kubectl delete svc nginx other-nginx -n training
```

---

## Verification Checklist

- [ ] Default: all traffic allowed (no policy)
- [ ] Deny-all ingress blocks all incoming traffic
- [ ] Allow-specific ingress permits only selected traffic
- [ ] Egress policies control outbound traffic
- [ ] Namespace selectors work correctly
- [ ] Cleanup removes policies and test resources

---

## Key Commands Summary

```bash
# List/Describe
kubectl get networkpolicies -n <namespace>
kubectl get netpol -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>

# Delete
kubectl delete networkpolicy <name> -n <namespace>
kubectl delete -f <file.yaml>

# Test connectivity
kubectl exec <pod> -n <namespace> -- curl/wget http://<target>
```

---

## Best Practices

1. **Default deny** - Start with deny-all, add specific allows
2. **Least privilege** - Only allow necessary traffic
3. **Namespace isolation** - Use namespace policies for environment separation
4. **DNS** - Always allow egress to kube-dns for service discovery
5. **Test thoroughly** - Network policies can be tricky

---

## Next Lab

Proceed to [Lab 18: Affinity & Taints](../k8s-training/18-affinity-taint.yaml) - Controlling pod placement.
