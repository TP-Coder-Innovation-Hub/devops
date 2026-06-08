# Capstone Project

## Objective

Set up a complete CI/CD pipeline for a sample application: Docker build, automated testing, deploy to Kubernetes, and monitor with observability tools.

## Architecture

```mermaid
graph LR
    Push[Git Push] --> CI[GitHub Actions CI]
    CI -->|build + test| Registry[Container Registry]
    Registry --> CD[GitHub Actions CD]
    CD -->|deploy| K8s[Kubernetes Cluster]
    K8s --> Monitor[Prometheus + Grafana]
```

## Prerequisites

- GitHub account
- Docker installed locally
- `kubectl` installed
- Access to a Kubernetes cluster (kind, minikube, or cloud)
- Basic familiarity with all modules (00 through 05)

## Steps

### 1. Application Setup

Create a simple HTTP application with a health endpoint.

```yaml
# Accept any language. Requirements:
endpoints:
  - GET / — returns "Hello World"
  - GET /health — returns 200 OK
  - GET /metrics — Prometheus metrics format

tests:
  - unit tests for core logic
  - integration test for /health endpoint
```

### 2. Dockerfile

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["node", "dist/server.js"]
```

Verify locally:
```bash
docker build -t capstone-app .
docker run -d -p 8080:8080 --name capstone capstone-app
curl http://localhost:8080/health
```

### 3. CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test
      - run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.event_name == 'push' }}
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### 4. Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: capstone-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: capstone-app
  template:
    metadata:
      labels:
        app: capstone-app
    spec:
      containers:
        - name: app
          image: ghcr.io/YOUR_ORG/capstone-app:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: capstone-app
spec:
  selector:
    app: capstone-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 5. CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    needs: [ci]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - run: |
          kubectl set image deployment/capstone-app \
            app=ghcr.io/YOUR_ORG/capstone-app:${{ github.sha }}
          kubectl rollout status deployment/capstone-app --timeout=300s
```

### 6. Observability

```yaml
# Add Prometheus annotations to pods
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

Deploy Prometheus and Grafana to the cluster. Create a dashboard with RED metrics for the application.

### 7. Verify

```bash
# Application responds
kubectl port-forward svc/capstone-app 8080:80
curl http://localhost:8080/health

# Metrics exposed
curl http://localhost:8080/metrics

# Check deployment
kubectl get pods -l app=capstone-app
kubectl logs -f deployment/capstone-app
```

## Success Criteria

- [ ] Push to `main` triggers CI pipeline
- [ ] CI builds, tests, and pushes container image
- [ ] CD deploys to Kubernetes automatically
- [ ] Application serves traffic via Service
- [ ] Health checks pass (readiness + liveness)
- [ ] Metrics visible in Grafana dashboard
- [ ] Rolling update works (deploy new version, verify zero downtime)
