# CI Pipeline `[Mid]`

## Pipeline Stages

A CI pipeline runs on every push. It must be fast (under 10 minutes) and reliable (no flaky tests).

```mermaid
graph LR
    Checkout --> Lint --> Build --> Test --> Security --> Artifact
```

## GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - name: Run unit tests
        run: npm run test:unit -- --coverage

      - name: Run integration tests
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/app_test
        run: npm run test:integration

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info

  security:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - name: Audit dependencies
        run: npm audit --audit-level=high

      - name: Run CodeQL
        uses: github/codeql-action/analyze@v3
        with:
          languages: javascript

  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Line by Line

| Section | Purpose |
|---------|---------|
| `on: push/pull_request` | Trigger on every push to main and every PR |
| `services: postgres` | Spin up a test database container |
| `needs: lint` | Run test only after lint passes |
| `npm ci` | Clean install from lockfile (reproducible) |
| `--coverage` | Generate coverage report |
| `npm audit --audit-level=high` | Fail on high-severity vulnerabilities |
| `docker/metadata-action` | Tag images with git SHA and branch name |
| `cache-from: type=gha` | Use GitHub Actions cache for Docker layers |

## Pipeline Speed

```yaml
# Strategies to keep pipelines under 10 minutes

parallel_jobs:          # Run lint + security in parallel
  lint: ~2min
  security: ~3min

caching:
  - npm cache (actions/setup-node)
  - docker layer cache (type=gha)

test_splitting:
  - unit tests: fast, no external deps
  - integration tests: slower, use service containers

skip_strategies:
  - path filtering (only run if relevant files changed)
  - draft PRs (skip CI on drafts)
```

```yaml
# Skip CI for docs-only changes
paths-filter:
  filters: |
    src:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
```

## Pipeline Failure Handling

```yaml
# Notify on failure
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "CI failed on ${{ github.ref }}: ${{ github.event.head_commit.message }}",
        "blocks": [{"type": "section", "text": {"type": "mrkdwn", "text": ":x: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|CI failed>"}}]
      }
```
