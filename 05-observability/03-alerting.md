# Alerting

## Alert on Symptoms, Not Causes

```yaml
# BAD — alert on internal state
alert: HighCPUUsage
expr: process_cpu_seconds_total > 0.9
# Why is this bad? High CPU might be fine if the service handles it.
# You don't know if users are impacted.

# GOOD — alert on user-facing symptoms
alert: HighErrorRate
expr: |
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
  > 0.01
for: 5m
# Users are seeing errors. Act now.
```

**Principle:** If users cannot tell something is wrong, do not page a human at 3 AM.

## Alert Fatigue

When on-call gets paged 20 times per shift for non-urgent alerts, they start ignoring pages. Including the real emergency.

```yaml
# How to prevent alert fatigue
rules:
  - alert: ServiceDown
    expr: up{job="app"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Service {{ $labels.job }} is down"
      runbook: "https://wiki/runbooks/service-down"

  - alert: HighMemoryUsage
    expr: container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} using >90% memory"

  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
```

## Severity Levels

| Severity | Meaning | Response Time | Notification |
|----------|---------|---------------|-------------|
| Critical | User-facing outage | Immediate (5 min) | Page on-call |
| Warning | Degradation, will become critical | Within 1 hour | Slack, email |
| Info | Notable event, no action needed | Next business day | Dashboard only |

Every alert must have a severity. If it doesn't warrant a severity, it's probably not an alert.

## Alerting Rules Structure

```yaml
# Prometheus alerting rules
groups:
  - name: api_alerts
    interval: 30s
    rules:
      - alert: APIHighErrorRate
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="api"}[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "API error rate is {{ $value | humanizePercentage }}"
          description: |
            The API service is returning >1% 5xx errors.
            Current rate: {{ $value | humanizePercentage }}
          runbook: "https://wiki/runbooks/api-high-error-rate"

      - alert: APIHighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{job="api"}[5m])) by (le)
          ) > 2.0
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "API p95 latency is {{ $value }}s"
```

The `for` clause prevents flapping alerts. The condition must be true for the entire duration before firing.

## Alertmanager Configuration

```yaml
# alertmanager.yml
route:
  receiver: default
  group_by: [alertname, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: pager
      repeat_interval: 30m

    - match:
        severity: warning
      receiver: slack
      repeat_interval: 2h

receivers:
  - name: default
    slack_configs:
      - channel: "#alerts"
        send_resolved: true

  - name: pager
    pagerduty_configs:
      - service_key: <pagerduty-key>

  - name: slack
    slack_configs:
      - channel: "#alerts-warning"
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: [alertname, service]
    # If critical alert fires, suppress the matching warning
```

## Runbooks

Every alert must link to a runbook. A runbook answers:

1. What does this alert mean?
2. How do I verify there's a real problem?
3. What are the steps to fix it?
4. How do I escalate if I cannot fix it?

```markdown
# Runbook: APIHighErrorRate

## What
API 5xx error rate exceeds 1% for 5 minutes.

## Verify
1. Check Grafana dashboard: is the error rate elevated?
2. Check logs: `sum by (error) (rate(http_requests_total{status=~"5.."}[5m]))`
3. Is a specific endpoint failing, or all endpoints?

## Common Causes
1. Database connection pool exhaustion
2. Upstream service (payments, auth) returning errors
3. Bad deployment (check recent deploys)

## Fix
1. If bad deploy: `kubectl rollout undo deployment/api -n production`
2. If DB pool exhaustion: restart a pod to reset connections
3. If upstream: check the upstream service's status page

## Escalate
If unresolved in 30 minutes, escalate to backend team lead.
```

Without runbooks, on-call engineers guess. Guessing at 3 AM makes things worse.
