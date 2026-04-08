# Lab 20: Helm - Package Manager for Kubernetes

## Objective
Learn how to use Helm to deploy, manage, and upgrade applications on Kubernetes using charts.

---

## What is Helm?

**Helm** is the package manager for Kubernetes. It helps you:
- Deploy complex applications with a single command
- Version and manage application configurations
- Rollback to previous versions easily
- Share applications as reusable charts

**Key concepts:**
- **Chart** - A package of pre-configured Kubernetes resources
- **Release** - A deployed instance of a chart
- **Repository** - A place where charts are stored and shared

---

## Step 1: Install Helm (If Not Already Installed)

```bash
# macOS (using Homebrew)
brew install helm

# Verify installation
helm version
```

Expected output:
```
version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GoVersion:"..."}
```

---

## Step 2: Add a Helm Repository

Helm charts are stored in repositories. Let's add the official Bitnami charts repository:

```bash
# Add Bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repository index
helm repo update

# List added repositories
helm repo list
```

---

## Step 3: Search for Charts

```bash
# Search for nginx chart
helm search repo nginx

# Search for redis chart
helm search repo redis

# Search with specific version
helm search repo nginx --versions
```

---

## Step 4: View Chart Information

Before installing, let's inspect a chart:

```bash
# View chart details
helm show chart bitnami/nginx

# View all values (default configuration)
helm show values bitnami/nginx

# View README
helm show readme bitnami/nginx
```

---

## Step 5: Install a Simple Application

Let's install nginx using Helm:

```bash
# Install nginx with release name "my-nginx"
helm install my-nginx bitnami/nginx -n training --create-namespace

# Check installation status
helm list -n training
kubectl get all -n training -l app.kubernetes.io/instance=my-nginx
```

**Expected output:**
```
NAME       NAMESPACE  REVISION  UPDATED                            STATUS    CHART         APP VERSION
my-nginx   training   1         2026-04-08 12:00:00.000000 +0000   deployed  nginx-15.x.x  1.25.x
```

---

## Step 6: Access the Application

```bash
# Get service details
kubectl get svc -n training -l app.kubernetes.io/instance=my-nginx

# Port forward to access locally
kubectl port-forward svc/my-nginx -n training 8080:80

# Test (in another terminal)
curl http://localhost:8080
```

---

## Step 7: Customize Chart Values

Charts have configurable values. Let's customize during installation:

```bash
# Install with custom values
helm install myredis bitnami/redis \
  -n training \
  --set architecture=standalone \
  --set auth.enabled=false \
  --set master.count=1

# Check the deployment
kubectl get pods -n training -l app.kubernetes.io/instance=myredis
```

---

## Step 8: Create a Custom Values File

Instead of using `--set`, create a values file:

Create file `my-values.yaml`:

```yaml
# Application name override
fullnameOverride: "my-webapp"

# Replica count
replicaCount: 2

# Image settings
image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent

# Service configuration
service:
  type: ClusterIP
  port: 80

# Resource limits
resources:
  limits:
    cpu: "100m"
    memory: "128Mi"
  requests:
    cpu: "50m"
    memory: "64Mi"

# Enable liveness/readiness probes
livenessProbe:
  enabled: true
  httpGet:
    path: /
    port: 80
readinessProbe:
  enabled: true
  httpGet:
    path: /
    port: 80
```

Install using the values file:

```bash
helm install my-webapp bitnami/nginx \
  -n training \
  -f my-values.yaml

# Verify
kubectl get deployment -n training my-webapp
kubectl get pods -n training -l app.kubernetes.io/instance=my-webapp
```

---

## Step 9: Upgrade a Release

Let's upgrade nginx to a newer version:

```bash
# Check current version
helm list -n training

# Upgrade to specific version
helm upgrade my-nginx bitnami/nginx -n training --set image.tag=1.26-alpine

# Check release history
helm history my-nginx -n training
```

---

## Step 10: Rollback a Release

```bash
# Rollback to previous version
helm rollback my-nginx -n training

# Rollback to specific revision
helm rollback my-nginx 1 -n training

# Verify rollback
helm history my-nginx -n training
kubectl get deployment my-nginx -n training -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## Step 11: List and Inspect Releases

```bash
# List all releases
helm list -n training -a

# Get full details of a release
helm status my-nginx -n training

# Get release values
helm get values my-nginx -n training

# Get all values (including defaults)
helm get values my-nginx -n training --all
```

---

## Step 12: Create Your Own Simple Chart

Let's create a basic Helm chart from scratch:

```bash
# Create a new chart
helm create my-chart -n training

# Inspect the structure
ls -la my-chart/
```

Structure:
```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── charts/             # Dependency charts
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # Helper templates
│   └── NOTES.txt       # Post-install notes
└── .helmignore
```

---

## Step 13: Customize the Template

Edit `my-chart/values.yaml`:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: "100m"
    memory: "128Mi"
  requests:
    cpu: "50m"
    memory: "64Mi"
```

Edit `my-chart/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    app: {{ include "my-chart.name" . }}
    chart: {{ include "my-chart.chart" . }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "my-chart.name" . }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ include "my-chart.name" . }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 10 }}
```

---

## Step 14: Lint and Dry-Run

Before installing, validate your chart:

```bash
# Lint the chart
helm lint my-chart

# Dry-run (preview what would be installed)
helm install test-release my-chart -n training --dry-run --debug
```

---

## Step 15: Package and Share Your Chart

```bash
# Package the chart
helm package my-chart

# This creates: my-chart-0.1.0.tgz

# Install from local package
helm install my-app ./my-chart-0.1.0.tgz -n training
```

---

## Step 16: Cleanup

```bash
# Uninstall releases
helm uninstall my-nginx -n training
helm uninstall myredis -n training
helm uninstall my-webapp -n training
helm uninstall test-release -n training
helm uninstall my-app -n training

# Remove Helm repository
helm repo remove bitnami

# Verify
helm list -n training
kubectl get all -n training
```

---

## Verification Checklist

- [ ] Helm installed and version verified
- [ ] Bitnami repository added and updated
- [ ] Application installed using `helm install`
- [ ] Application accessed via port-forward
- [ ] Custom values file created and used
- [ ] Release upgraded to new version
- [ ] Release rolled back successfully
- [ ] Custom chart created and linted
- [ ] Dry-run validated the template
- [ ] All releases uninstalled

---

## Helm vs Manual YAML

| Feature | Manual YAML | Helm |
|---------|------------|------|
| Versioning | Manual | Built-in |
| Rollback | Manual | `helm rollback` |
| Templating | No | Yes |
| Sharing | Copy files | Charts |
| Dependencies | Manual | Charts/Dependencies |

---

## Key Commands Summary

```bash
# Repository management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo <chart>

# Chart inspection
helm show chart <chart>
helm show values <chart>
helm show readme <chart>

# Installation
helm install <release> <chart> -n <namespace>
helm install <release> <chart> -n <namespace> -f values.yaml
helm install <release> <chart> -n <namespace> --set key=value

# Management
helm list -n <namespace>
helm status <release> -n <namespace>
helm get values <release> -n <namespace>
helm get values <release> -n <namespace> --all

# Upgrades and Rollbacks
helm upgrade <release> <chart> -n <namespace>
helm rollback <release> -n <namespace>
helm rollback <release> <revision> -n <namespace>
helm history <release> -n <namespace>

# Uninstall
helm uninstall <release> -n <namespace>

# Chart creation
helm create <chart-name>
helm lint <chart>
helm package <chart>
helm install <release> <chart> --dry-run --debug
```

---

## Common Issues

### Release not found
```bash
# Check if release exists
helm list -n <namespace>

# Check all namespaces
helm list -A
```

### Image pull errors
```bash
# Check image tag exists
helm show values <chart> | grep tag

# Override image
helm install <release> <chart> --set image.tag=latest
```

### Permission denied
```bash
# Make sure namespace exists or use --create-namespace
helm install <release> <chart> -n <namespace> --create-namespace
```

### Dry-run shows errors
```bash
# Validate chart syntax
helm lint <chart>

# Check template rendering
helm template <release> <chart> -n <namespace>
```

---

## Next Steps

Congratulations! You've completed all 20 Kubernetes Training Labs! 🎉

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
20. ✅ Helm - Package manager

**Further learning:**
- ArgoCD/Flux (GitOps)
- Prometheus/Grafana (monitoring)
- Istio/Linkerd (service mesh)
- Vault (secrets management)
- Kustomize (template-free customization)

---

## Thank You!

**Training resources location:**
```
/Users/tumz/workspace/nttlab-trueminsert/
├── k8s-training/         # YAML lab files
├── presentation/         # Slide content
└── labs-guideline/       # Step-by-step guides
```
