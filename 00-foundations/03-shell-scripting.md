# Shell Scripting `[Entry]`

## Why Shell Scripts

Shell scripts automate repetitive tasks: deployments, backups, environment setup, health checks. If you type the same commands twice, script it.

## Variables and Substitution

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_NAME="api-server"
VERSION="1.4.2"
DEPLOY_DIR="/opt/app"

# Command substitution
COMMIT_HASH=$(git rev-parse --short HEAD)
BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "Building ${APP_NAME} v${VERSION} at ${COMMIT_HASH}"
```

- `set -euo pipefail` — exit on error, undefined variable, pipe failure
- Use `${VAR}` with braces to avoid ambiguity
- Use `$()` for command substitution, not backticks

## Conditionals

```bash
if [ -f "${DEPLOY_DIR}/app.pid" ]; then
    echo "App appears to be running"
    exit 1
fi

if [ "${ENVIRONMENT}" = "production" ]; then
    REPLICAS=3
elif [ "${ENVIRONMENT}" = "staging" ]; then
    REPLICAS=1
else
    REPLICAS=0
fi
```

## Loops

```bash
# Deploy to multiple servers
SERVERS=("10.0.0.5" "10.0.0.6" "10.0.0.7")

for SERVER in "${SERVERS[@]}"; do
    echo "Deploying to ${SERVER}..."
    ssh "deploy@${SERVER}" "systemctl restart ${APP_NAME}"
done

# Wait for service health
MAX_RETRIES=30
for i in $(seq 1 "${MAX_RETRIES}"); do
    if curl -sf "http://localhost:8080/health" > /dev/null; then
        echo "Service is healthy"
        exit 0
    fi
    sleep 2
done

echo "Service failed to start"
exit 1
```

## Functions

```bash
log() {
    local level="$1"
    shift
    local message="$*"
    local timestamp
    timestamp=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    echo "[${timestamp}] [${level}] ${message}"
}

deploy() {
    local env="$1"
    local version="$2"

    log INFO "Starting deployment: env=${env} version=${version}"

    if ! pull_image "${version}"; then
        log ERROR "Failed to pull image ${version}"
        return 1
    fi

    if ! run_migrations "${env}"; then
        log ERROR "Migrations failed, rolling back"
        rollback "${env}"
        return 1
    fi

    restart_service "${env}"
    wait_for_healthy "${env}"
    log INFO "Deployment complete"
}
```

## Real-World Deploy Script

```bash
#!/usr/bin/env bash
set -euo pipefail

ENV="${1:?Usage: $0 <environment> <version>}"
VERSION="${2:?Usage: $0 <environment> <version>}"

REGISTRY="registry.example.com"
IMAGE="${REGISTRY}/app:${VERSION}"

log() { echo "[$(date -u +%FT%T)] $*"; }

# Pull latest image
log "Pulling ${IMAGE}"
docker pull "${IMAGE}"

# Run database migrations
log "Running migrations"
docker run --rm --env-file ".env.${ENV}" "${IMAGE}" migrate

# Rolling restart
log "Restarting service"
kubectl set image "deployment/app" "app=${IMAGE}" --namespace "${ENV}"
kubectl rollout status "deployment/app" --namespace "${ENV}" --timeout=300s

# Verify
log "Verifying deployment"
kubectl get pods --namespace "${ENV}" -l app=app

log "Deploy complete: ${ENV}@${VERSION}"
```

## Best Practices

- Always use `set -euo pipefail`
- Quote all variables (`"${VAR}"`)
- Use `shellcheck` to lint scripts
- Keep scripts short — if it exceeds 200 lines, use a real programming language
- Store scripts under version control
