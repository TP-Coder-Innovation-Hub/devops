# Docker Compose `[Mid]`

## Multi-Container Applications

Real applications have multiple services: a web server, a database, a cache. Docker Compose defines and runs them together with a single config file.

```mermaid
graph TD
    App[app:8080] --> DB[postgres:5432]
    App --> Redis[redis:6379]
```

## Compose File

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://appuser:secret@postgres:5432/appdb
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

## Service Dependencies

`depends_on` controls startup order:

```yaml
depends_on:
  postgres:
    condition: service_healthy   # wait until healthcheck passes
  redis:
    condition: service_started   # wait until container starts
```

Without `condition`, services start in order but don't wait to be ready. Use `service_healthy` when your app requires the dependency to be fully available.

## Commands

```bash
# Start all services (detached)
docker compose up -d

# Follow logs
docker compose logs -f app

# Logs for one service
docker compose logs -f postgres

# Rebuild after code change
docker compose up -d --build app

# Stop all services
docker compose down

# Stop and remove volumes (clean slate)
docker compose down -v

# Run one-off command
docker compose exec app npm run migrate
docker compose run --rm app npm run seed

# Scale a service (stateless only)
docker compose up -d --scale app=3

# Check service status
docker compose ps
```

## Environment Configuration

```yaml
# Use .env file for local development
# .env
POSTGRES_PASSWORD=localdev
NODE_ENV=development
```

```yaml
# Override for local development
# docker-compose.override.yml (auto-merged)
services:
  app:
    volumes:
      - ./src:/app/src     # live reload
    environment:
      NODE_ENV: development
    command: npm run dev    # override production CMD
```

```bash
# Production compose file
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Networking

Compose creates a default network. Services communicate using service names as hostnames:

```yaml
# Inside the app container:
# postgres:5432 resolves to the postgres service
# redis:6379 resolves to the redis service
DATABASE_URL: postgres://appuser:secret@postgres:5432/appdb
```

No need for IP addresses or port mapping between services. Only expose ports to the host that you need to access externally.

## Volumes

```yaml
volumes:
  # Named volume — managed by Docker, persists across restarts
  postgres_data:

  # Bind mount — maps host directory to container
  - ./src:/app/src:ro    # :ro = read-only

  # tmpfs — in-memory, removed on stop
  - type: tmpfs
    target: /tmp
    tmpfs:
      size: 100M
```

Named volumes for databases. Bind mounts for local development. Never store persistent data in the container's writable layer.
