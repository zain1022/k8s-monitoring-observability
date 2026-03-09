# Incident Response Runbook — mini-api Observability Stack

## Overview

This runbook documents incident response workflows for the mini-api service running on Kubernetes, monitored via Prometheus and Grafana. It is intended to simulate production-grade reliability practices including alert triage, diagnosis, and resolution.

---

## Monitoring Stack

| Component | Purpose |
|---|---|
| Prometheus | Metrics collection and alert evaluation |
| Grafana | Dashboard visualization (RED metrics) |
| AlertManager | Alert routing and notification |
| Kube State Metrics | Kubernetes object-level metrics |
| Node Exporter | Host-level hardware and OS metrics |
| prometheus-fastapi-instrumentator | Application-level metrics from mini-api |

---

## Alert Definitions

### 1. HighErrorRate
- **Severity:** Warning
- **Condition:** `rate(http_requests_total{job="mini-api-svc", status="4xx"}[1m]) > 0.01`
- **Meaning:** The mini-api service is returning more than 0.01 4xx errors per second over a 1-minute window
- **Typical Causes:** Bad requests from clients, misconfigured routes, missing resources

### 2. HighLatency
- **Severity:** Warning
- **Condition:** `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="mini-api-svc"}[1m])) > 0.5`
- **Meaning:** The 99th percentile response time exceeds 500ms
- **Typical Causes:** Resource exhaustion, pod throttling, slow downstream dependencies

### 3. PodDown
- **Severity:** Critical
- **Condition:** `up{job="mini-api-svc"} == 0`
- **Meaning:** One or more mini-api pods are unreachable by Prometheus for more than 1 minute
- **Typical Causes:** CrashLoopBackOff, OOMKilled, failed liveness probe, node failure

---

## Incident Response Workflows

### Alert: HighErrorRate

**Step 1 — Confirm the alert**
```bash
# Check request counts by status code
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Query in Prometheus UI:
rate(http_requests_total{job="mini-api-svc", status="4xx"}[1m])
```

**Step 2 — Check pod logs for error details**
```bash
kubectl logs -n mini-platform deploy/mini-api --tail=100
kubectl logs -n mini-platform <pod-name> --previous  # if pod restarted
```

**Step 3 — Check recent events**
```bash
kubectl get events -n mini-platform --sort-by=.metadata.creationTimestamp | Select-Object -Last 20
```

**Step 4 — Inspect pod health**
```bash
kubectl describe pod -n mini-platform <pod-name>
kubectl get pods -n mini-platform
```

**Step 5 — Resolution**
- If bad route: fix application code, rebuild image, trigger rolling update
- If misconfigured ConfigMap: update and reapply `kubectl apply -f manifests/configmap.yaml`
- If client-side issue: no action needed, monitor for sustained error rate

---

### Alert: HighLatency

**Step 1 — Confirm the alert**
```bash
# Query in Prometheus UI:
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="mini-api-svc"}[1m]))
```

**Step 2 — Check resource utilization**
```bash
kubectl top pods -n mini-platform
kubectl top nodes
```

**Step 3 — Check HPA status**
```bash
kubectl get hpa -n mini-platform
kubectl describe hpa -n mini-platform mini-api-hpa
```

**Step 4 — Check if CPU throttling is occurring**
```bash
kubectl describe pod -n mini-platform <pod-name>
# Look for: cpu throttling warnings in events
```

**Step 5 — Resolution**
- If CPU throttled: increase resource limits in `manifests/deployment.yaml`, reapply
- If HPA not scaling: verify metrics-server is running with `kubectl top nodes`
- If sustained: consider scaling manually `kubectl scale deployment -n mini-platform mini-api --replicas=4`

---

### Alert: PodDown

**Step 1 — Confirm which pods are down**
```bash
kubectl get pods -n mini-platform
kubectl get pods -n mini-platform -o wide
```

**Step 2 — Check pod status and events**
```bash
kubectl describe pod -n mini-platform <pod-name>
kubectl get events -n mini-platform --sort-by=.metadata.creationTimestamp
```

**Step 3 — Check logs (including previous container if restarted)**
```bash
kubectl logs -n mini-platform <pod-name>
kubectl logs -n mini-platform <pod-name> --previous
```

**Step 4 — Common error diagnosis**

| Error | Diagnosis Command | Fix |
|---|---|---|
| CrashLoopBackOff | `kubectl logs <pod> --previous` | Fix app error, rebuild image |
| OOMKilled | `kubectl describe pod <pod>` | Increase memory limits |
| ImagePullBackOff | `kubectl describe pod <pod>` | Verify image tag exists |
| Pending | `kubectl describe pod <pod>` | Check node resources |

**Step 5 — Resolution**
```bash
# Force restart deployment
kubectl rollout restart deployment -n mini-platform mini-api

# Or rollback to last stable version
kubectl rollout undo -n mini-platform deployment/mini-api
kubectl rollout status -n mini-platform deployment/mini-api
```

---

## General Diagnostic Commands

```bash
# Full cluster overview
kubectl get all -n mini-platform

# Pod resource usage
kubectl top pods -n mini-platform

# Deployment status
kubectl rollout status -n mini-platform deployment/mini-api

# Exec into pod for internal debugging
kubectl exec -n mini-platform -it <pod-name> -- sh

# Check environment variables inside pod
echo $ENV
echo $MESSAGE

# Service and endpoint health
kubectl get svc -n mini-platform
kubectl get endpoints -n mini-platform mini-api-svc

# Check Prometheus targets
# Navigate to http://localhost:9090 -> Status -> Targets -> filter "mini-api"
```

---

## Grafana Dashboard Reference

**Dashboard:** mini-api Observability

| Panel | Query | Purpose |
|---|---|---|
| Request Rate (req/sec) | `rate(http_requests_total{job="mini-api-svc"}[1m])` | Traffic volume per endpoint |
| Request Latency p99 (seconds) | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="mini-api-svc"}[1m]))` | Worst-case response time |
| Error Rate (4xx) | `rate(http_requests_total{job="mini-api-svc", status="4xx"}[1m])` | Failed request rate |

---

## Key Takeaways

- Practiced the **RED method** (Rate, Errors, Duration) for service monitoring
- Configured **PrometheusRules** for automated alert evaluation
- Built structured runbooks to simulate **production incident response**
- Used **ServiceMonitor** to wire application metrics into the Prometheus scrape pipeline
- Differentiated between **infrastructure-level** (node/pod) and **application-level** (HTTP) observability
