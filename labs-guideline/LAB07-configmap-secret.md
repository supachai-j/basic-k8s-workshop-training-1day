# Lab 07: ConfigMap & Secret - Externalized Configuration

## Objective
Learn how to manage application configuration separately from container images using ConfigMaps and Secrets.

---

## What are ConfigMap and Secret?

**ConfigMap** - Stores non-sensitive configuration data (key-value pairs, files)
**Secret** - Stores sensitive data (passwords, API keys, certificates) in base64 encoding

Both decouple configuration from container images, enabling:
- Environment-specific configs (dev/staging/prod)
- Secrets management without hardcoding
- Configuration changes without rebuilding images

---

## Step 1: Create a ConfigMap

Create file `07-configmap-secret.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: training
  labels:
    lesson: "07"
data:
  # Simple key-value pairs
  APP_ENV: "training"
  APP_LOG_LEVEL: "debug"
  APP_PORT: "8080"

  # Multi-line configuration file
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
      location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
      }
    }
```

Apply ConfigMap:

```bash
kubectl apply -f 07-configmap-secret.yaml
```

---

## Step 2: Verify ConfigMap

```bash
# List ConfigMaps
kubectl get configmaps -n training
kubectl get cm -n training

# Describe ConfigMap
kubectl describe configmap app-config -n training

# View full ConfigMap
kubectl get configmap app-config -n training -o yaml
```

---

## Step 3: Create a Secret

Add to `07-configmap-secret.yaml`:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: training
  labels:
    lesson: "07"
type: Opaque
stringData:
  db-username: "training-user"
  db-password: "Sup3rS3cret!"
  api-key: "abc123-xyz789-training"
```

Apply Secret:

```bash
kubectl apply -f 07-configmap-secret.yaml
```

---

## Step 4: Verify Secret

```bash
# List Secrets
kubectl get secrets -n training

# Describe Secret (values hidden)
kubectl describe secret app-secret -n training

# Get Secret values (base64 encoded)
kubectl get secret app-secret -n training -o jsonpath='{.data}'

# Decode a specific key
kubectl get secret app-secret -n training -o jsonpath='{.data.db-password}' | base64 -d
```

---

## Step 5: Create Pod Using ConfigMap and Secret

Add to `07-configmap-secret.yaml`:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
  namespace: training
  labels:
    app: config-demo
    lesson: "07"
spec:
  containers:
    - name: app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80

      # Single environment variable from ConfigMap
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db-password

      # All keys from ConfigMap as environment variables
      envFrom:
        - configMapRef:
            name: app-config
          prefix: CFG_

      # Mount ConfigMap as files
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: secret-volume
          mountPath: /etc/app/secrets
          readOnly: true

      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"

  volumes:
    # ConfigMap mounted as files
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: nginx.conf
            path: default.conf

    # Secret mounted as files
    - name: secret-volume
      secret:
        secretName: app-secret
        defaultMode: 0400
```

Apply:

```bash
kubectl apply -f 07-configmap-secret.yaml
```

---

## Step 6: Verify Pod

```bash
# Check pod status
kubectl get pod config-demo -n training

# Wait for Running
kubectl get pod config-demo -n training -w
```

---

## Step 7: Verify Environment Variables

```bash
# Check environment variables inside container
kubectl exec config-demo -n training -- env | grep -E "(APP_|CFG_|DB_)"
```

Expected outputs:
- `APP_ENV=training`
- `CFG_APP_LOG_LEVEL=debug`
- `CFG_APP_PORT=8080`
- `DB_PASSWORD=Sup3rS3cret!`

---

## Step 8: Verify Mounted Files

```bash
# Check ConfigMap mounted file
kubectl exec config-demo -n training -- cat /etc/nginx/conf.d/default.conf

# Check Secret mounted files
kubectl exec config-demo -n training -- ls -la /etc/app/secrets/
kubectl exec config-demo -n training -- cat /etc/app/secrets/db-password
```

---

## Step 9: Test Application

```bash
# Port forward to test
kubectl port-forward config-demo -n training 8080:80 &

# Test health endpoint
curl http://localhost:8080/health

# Test main page
curl http://localhost:8080/
```

---

## Step 10: Update ConfigMap (Without Restarting Pod)

ConfigMaps mounted as files update automatically (may take up to a minute):

```bash
# Update ConfigMap
kubectl patch configmap app-config -n training -p \
  '{"data":{"APP_LOG_LEVEL":"info"}}'

# Verify file updated inside pod
kubectl exec config-demo -n training -- cat /etc/nginx/conf.d/default.conf
```

**Note:** Environment variables from `env` or `envFrom` require pod restart to take effect.

---

## Step 11: Secret Creation Methods

### Method 1: stringData (plain text)
```yaml
stringData:
  api-key: "plain-text-key"
```

### Method 2: base64 encoded data
```bash
# Encode manually
echo -n "my-password" | base64
# Output: bXktcGFzc3dvcmQ=

# Use in Secret
data:
  password: bXktcGFzc3dvcmQ=
```

### Method 3: From file
```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  -n training
```

---

## Step 12: Cleanup

```bash
kubectl delete -f 07-configmap-secret.yaml
```

---

## ConfigMap vs Secret

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| **Purpose** | Non-sensitive config | Sensitive data |
| **Encoding** | Plain text | Base64 (not encrypted!) |
| **Use Case** | App settings, config files | Passwords, API keys, certs |
| **Storage** | etcd (unencrypted) | etcd (unencrypted by default) |
| **Encryption** | At rest (if configured) | At rest (if encrypted) |

**Important:** Kubernetes Secrets are NOT encrypted by default! Use:
- Encryption at rest for etcd
- External secrets manager (Vault, AWS Secrets Manager)
- Kubernetes External Secrets Operator

---

## Verification Checklist

- [ ] ConfigMap created with key-value pairs
- [ ] Secret created (values base64 encoded)
- [ ] Pod has environment variables from ConfigMap
- [ ] Pod has environment variables from Secret
- [ ] ConfigMap mounted as file in pod
- [ ] Secret mounted as files in pod
- [ ] File updates reflect in pod without restart

---

## Key Commands Summary

```bash
# Create
kubectl apply -f <file.yaml>

# List
kubectl get configmaps -n training
kubectl get secrets -n training

# Describe
kubectl describe configmap <name> -n training
kubectl describe secret <name> -n training

# Get values
kubectl get configmap <name> -n training -o yaml
kubectl get secret <name> -n training -o jsonpath='{.data.<key>}' | base64 -d

# Update
kubectl patch configmap <name> -n training -p '{"data":{"key":"value"}}'

# Create from literals
kubectl create secret generic <name> --from-literal=key=value -n training

# Delete
kubectl delete -f <file.yaml>
```

---

## Next Lab

Proceed to [Lab 08: PersistentVolume](../k8s-training/08-persistent-volume.yaml) - Persistent storage.
