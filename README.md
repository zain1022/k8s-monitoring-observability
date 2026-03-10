# Kubernetes Monitoring & Observability Stack

A production-style monitoring and observability stack deployed on Kubernetes using Prometheus and Grafana. This project instruments a live FastAPI service with application-level metrics, visualizes them using a custom RED dashboard, and configures automated alert rules to simulate real-world incident response workflows.

This project extends the [Kubernetes Mini-Platform](https://github.com/zain1022/k8s-mini-platform) by adding a full observability layer on top of the running mini-api service.

---

## Architecture

```
mini-api (FastAPI)
    │
    │  /metrics endpoint (prometheus-fastapi-instrumentator)
    ▼
ServiceMonitor (monitoring namespace)
    │
    │  scrapes every 15s
    ▼
Prometheus (kube-prometheus-stack via Helm)
    │
    ├──▶ Grafana (RED Dashboard — Rate, Errors, Duration)
    │
    └──▶ AlertManager (HighErrorRate, HighLatency, PodDown)
```

---

## Stack

| Component | Purpose |
|---|---|
| Prometheus | Metrics collection and alert evaluation |
| Grafana | Dashboard visualization |
| AlertManager | Alert routing |
| Kube State Metrics | Kubernetes object-level metrics |
| Node Exporter | Host-level hardware and OS metrics |
| prometheus-fastapi-instrumentator | Application-level HTTP metrics |
| Helm | Stack deployment and management |

---

## Prerequisites

- Windows + PowerShell
- Docker Desktop
- Minikube
- kubectl
- Helm v3+
- [k8s-mini-platform](https://github.com/zain1022/k8s-mini-platform) deployed and running

---

## Project Structure

```
k8s-monitoring-observability/
├── manifests/
│   ├── servicemonitor.yaml       # Wires Prometheus to mini-api
│   └── alert-rules.yaml          # HighErrorRate, HighLatency, PodDown
├── dashboards/
│   └── mini-api-observability.json  # Exportable Grafana dashboard
├── docs/
│   ├── runbook.md                # Incident response workflows
│   └── screenshots/              # Dashboard and target screenshots
└── README.md
```

---

## How to Run

### 1. Start Minikube and deploy mini-api

Make sure your mini-platform is running first:

```powershell
minikube start --driver=docker
minikube addons enable metrics-server
kubectl get all -n mini-platform
```

### 2. Add the Prometheus Helm repo

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 3. Deploy the monitoring stack

```powershell
kubectl create namespace monitoring
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
```

Verify all pods are running:

```powershell
kubectl get pods -n monitoring
```

### 4. Apply the ServiceMonitor and alert rules

```powershell
kubectl apply -f manifests/servicemonitor.yaml
kubectl apply -f manifests/alert-rules.yaml
```

### 5. Access Grafana

```powershell
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open http://localhost:3000 in your browser.

Retrieve the admin password:

```powershell
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

Login with username `admin` and the retrieved password.

### 6. Access Prometheus

```powershell
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open http://localhost:9090 in your browser.

---

## Prometheus Targets

After applying the ServiceMonitor, Prometheus automatically discovers and scrapes both mini-api pods every 15 seconds.

Navigate to **Status → Targets** and filter by `mini-api` to verify both pods show **UP**.

![Prometheus Targets](docs/screenshots/prometheus_targets.png)

---

## Grafana Dashboard

The custom RED dashboard visualizes three core service health metrics:

| Panel | Query | Purpose |
|---|---|---|
| Request Rate (req/sec) | `rate(http_requests_total{job="mini-api-svc"}[1m])` | Traffic volume per endpoint |
| Request Latency p99 (seconds) | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="mini-api-svc"}[1m]))` | Worst-case response time |
| Error Rate (4xx) | `rate(http_requests_total{job="mini-api-svc", status="4xx"}[1m])` | Failed request rate |

![RED Dashboard](docs/screenshots/red-dashboard.png)

To import the dashboard: **Dashboards → Import → Upload JSON** → select `dashboards/mini-api-observability.json`

---

## Alert Rules

Three production-style alert rules are configured via PrometheusRule:

### HighErrorRate
- **Severity:** Warning
- **Condition:** 4xx error rate exceeds 0.01 req/sec over 1 minute
- **Response:** See [runbook](docs/runbook.md#alert-higherrorrate)

### HighLatency
- **Severity:** Warning
- **Condition:** p99 latency exceeds 500ms over 1 minute
- **Response:** See [runbook](docs/runbook.md#alert-highlatency)

### PodDown
- **Severity:** Critical
- **Condition:** Any mini-api pod unreachable for more than 1 minute
- **Response:** See [runbook](docs/runbook.md#alert-poddown)

![Alert Rules](docs/screenshots/alert-rules.png)

Verify alerts are loaded:

```powershell
kubectl get prometheusrule -n monitoring | Select-String mini-api
```

---

## Incident Response

A structured incident response runbook is documented in [docs/runbook.md](docs/runbook.md).

It covers step-by-step triage and resolution workflows for each alert, common error patterns (CrashLoopBackOff, OOMKilled, ImagePullBackOff), and general diagnostic commands.

---

## Key Takeaways

- Deployed a full observability stack using **Helm** — the industry standard for Kubernetes package management
- Instrumented a **live FastAPI service** with application-level Prometheus metrics using `prometheus-fastapi-instrumentator`
- Configured a **ServiceMonitor** to automatically wire the app into the Prometheus scrape pipeline
- Built a custom **RED dashboard** (Rate, Errors, Duration) in Grafana — the standard framework for service-level monitoring
- Wrote **PrometheusRule** alert definitions for error rate, latency, and pod availability
- Documented structured **incident response runbooks** simulating production on-call workflows
- Differentiated between **infrastructure-level observability** (node/pod metrics) and **application-level observability** (HTTP request metrics)
