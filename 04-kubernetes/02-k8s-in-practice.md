# Kubernetes in Practice

## ConfigMaps

Inject configuration without rebuilding images.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  CACHE_HOST: "redis.production.svc.cluster.local"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
```

```yaml
# Use in a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: ghcr.io/org/myapp:sha-abc1234
          envFrom:
            - configMapRef:
                name: app-config
```

Update a ConfigMap, restart pods to pick up changes. ConfigMaps are not hot-reloaded by default.

## Secrets

Store sensitive data. Base64-encoded (not encrypted — enable etcd encryption at rest).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YXBwdXNlcg==          # base64("appuser")
  password: c3VwZXJzZWNyZXQ=      # base64("supersecret")
```

```bash
# Create from literal
kubectl create secret generic db-credentials \
    --from-literal=username=appuser \
    --from-literal=password=supersecret

# Create from file
kubectl create secret generic tls-cert \
    --from-file=tls.crt=./certs/tls.crt \
    --from-file=tls.key=./certs/tls.key
```

```yaml
# Mount as environment variables
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

## Volumes

Persistent storage for Pods.

```yaml
# PersistentVolumeClaim — request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```yaml
# Mount in Pod
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data
```

| Volume Type | Use Case |
|-------------|----------|
| `persistentVolumeClaim` | Databases, file uploads (persists across pod restarts) |
| `emptyDir` | Temporary scratch space (deleted when pod stops) |
| `configMap` | Mount config files into container |
| `secret` | Mount sensitive files (TLS certs, keys) |
| `hostPath` | Mount from node filesystem (development only) |

## Health Checks

Kubernetes uses probes to decide when a pod is ready and when it needs to be restarted.

```yaml
# Readiness — can this pod serve traffic?
# Failed = remove from Service load balancer
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

# Liveness — is this pod healthy?
# Failed = restart the pod
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

# Startup — has the pod finished starting?
# Use for slow-starting apps
startupProbe:
  httpGet:
    path: /health/ready
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**Common mistake:** Using liveness probe for readiness. Liveness restarts the pod on failure. Readiness just removes it from the load balancer. Use readiness for transient issues (database temporarily unavailable).

## Resource Limits

```yaml
resources:
  requests:          # guaranteed minimum
    memory: "128Mi"
    cpu: "100m"      # 100 millicores = 0.1 CPU
  limits:            # maximum allowed
    memory: "256Mi"
    cpu: "500m"      # 500 millicores = 0.5 CPU
```

- **Requests** — scheduler uses this to place pods on nodes with enough capacity
- **Limits** — pod is throttled (CPU) or killed (memory) if exceeded

Always set both. Pods without requests can starve other workloads. Pods without limits can consume an entire node.

## Common Commands

```bash
# Get resources
kubectl get pods -n production
kubectl get pods -l app=myapp          # filter by label
kubectl get all -n production          # all resources in namespace

# Describe (events, config, status)
kubectl describe pod app-abc123 -n production

# Logs
kubectl logs -f deployment/app -n production
kubectl logs --previous app-abc123      # previous crashed container

# Exec into container
kubectl exec -it app-abc123 -- sh

# Port forward (local access)
kubectl port-forward svc/app-service 8080:80 -n production

# Apply changes
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/           # apply all files in directory

# Scale
kubectl scale deployment/app --replicas=5 -n production
```
