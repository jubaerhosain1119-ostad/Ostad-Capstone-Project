# Monitoring Setup Documentation

## Overview

This document describes the comprehensive monitoring solution for the finch application stack, including Prometheus for metrics collection, Loki for log aggregation, and Grafana for visualization and alerting.

## Architecture

The monitoring stack consists of:

1. **Prometheus**: Metrics collection and storage
2. **Loki**: Log aggregation
3. **Promtail**: Log collection agent
4. **Grafana**: Visualization and dashboards

## Prometheus Setup

### Deployment

**Location**: `kubernetes/monitoring/prometheus-deployment.yaml`

### Configuration

**Location**: `kubernetes/monitoring/prometheus-configmap.yaml`

Prometheus is configured to scrape metrics from:
- Prometheus itself
- Frontend pods (if instrumented)
- Backend pods (if instrumented)
- Redis (via redis_exporter)
- PostgreSQL (via postgres_exporter)
- Kubernetes nodes
- Kubernetes pods

### Scrape Configuration

```yaml
scrape_configs:
  - job_name: 'frontend'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: finch-frontend
        action: keep
```

### Storage

- **Persistent Volume**: 10Gi for metrics retention
- **Retention Period**: 15 days
- **Storage Path**: `/prometheus`

### Resource Allocation

- **CPU**: 250m request, 500m limit
- **Memory**: 512Mi request, 1Gi limit

### Accessing Prometheus

```bash
# Port forward to access Prometheus UI
kubectl port-forward svc/prometheus-service 9090:9090

# Access at http://localhost:9090
```

### Deploying Prometheus

```bash
# Create ConfigMap
kubectl apply -f kubernetes/monitoring/prometheus-configmap.yaml

# Deploy Prometheus
kubectl apply -f kubernetes/monitoring/prometheus-deployment.yaml

# Verify deployment
kubectl get pods -l app=prometheus
kubectl get svc prometheus-service
```

## Loki Setup

### Deployment

**Location**: `kubernetes/monitoring/loki-deployment.yaml`

### Configuration

**Location**: `kubernetes/monitoring/loki-configmap.yaml`

Loki is configured for:
- **Storage**: Filesystem with 168h (7 days) retention
- **Replication**: Single instance (for local development)
- **Schema**: v11 with 24h index period

### Storage

- **Persistent Volume**: 10Gi for log storage
- **Retention**: 168 hours (7 days)
- **Storage Path**: `/loki`

### Resource Allocation

- **CPU**: 250m request, 500m limit
- **Memory**: 512Mi request, 1Gi limit

### Accessing Loki

```bash
# Port forward to access Loki API
kubectl port-forward svc/loki-service 3100:3100

# Query logs via API
curl http://localhost:3100/ready
```

### Deploying Loki

```bash
# Create ConfigMap
kubectl apply -f kubernetes/monitoring/loki-configmap.yaml

# Deploy Loki
kubectl apply -f kubernetes/monitoring/loki-deployment.yaml

# Verify deployment
kubectl get pods -l app=loki
kubectl get svc loki-service
```

## Promtail Setup

### Deployment

**Location**: `kubernetes/monitoring/loki-deployment.yaml` (DaemonSet)

Promtail runs as a DaemonSet to collect logs from all nodes.

### Configuration

**Location**: `kubernetes/monitoring/promtail-configmap.yaml`

Promtail is configured to:
- Scrape logs from Kubernetes pods
- Scrape system logs
- Send logs to Loki service

### Log Collection

Promtail automatically discovers and collects logs from:
- All Kubernetes pods
- System logs from `/var/log`

### Resource Allocation

- **CPU**: 100m request, 200m limit
- **Memory**: 128Mi request, 256Mi limit

### Deploying Promtail

```bash
# Create ConfigMap
kubectl apply -f kubernetes/monitoring/promtail-configmap.yaml

# Promtail is deployed as part of Loki deployment
kubectl apply -f kubernetes/monitoring/loki-deployment.yaml

# Verify DaemonSet
kubectl get daemonset promtail
```

## Grafana Setup

### Deployment

**Location**: `kubernetes/monitoring/grafana-deployment.yaml`

### Configuration

Grafana is pre-configured with:
- **Prometheus** as default data source
- **Loki** as log data source
- **Default credentials**: admin/admin (change in production!)

### Data Sources

Data sources are automatically provisioned via ConfigMap:
- Prometheus: `http://prometheus-service:9090`
- Loki: `http://loki-service:3100`

### Storage

- **Persistent Volume**: 5Gi for dashboard and user data
- **Storage Path**: `/var/lib/grafana`

### Resource Allocation

- **CPU**: 250m request, 500m limit
- **Memory**: 512Mi request, 1Gi limit

### Accessing Grafana

```bash
# Port forward to access Grafana UI
kubectl port-forward svc/grafana-service 3000:3000

# Access at http://localhost:3000
# Default credentials: admin/admin
```

### Deploying Grafana

```bash
# Deploy Grafana
kubectl apply -f kubernetes/monitoring/grafana-deployment.yaml

# Verify deployment
kubectl get pods -l app=grafana
kubectl get svc grafana-service
```

## Grafana Dashboards

### Pre-configured Dashboards

**Location**: `grafana/dashboards/`

1. **Application Dashboard** (`application-dashboard.json`)
   - HTTP request rate
   - Error rate
   - Response time (p95)
   - Active connections

2. **Kubernetes Dashboard** (`kubernetes-dashboard.json`)
   - Pod CPU usage
   - Pod memory usage
   - Pod restarts
   - Node CPU usage
   - Node memory usage
   - Pod status

3. **Database Dashboard** (`database-dashboard.json`)
   - Redis memory usage
   - Redis commands per second
   - Redis connected clients
   - PostgreSQL connections
   - PostgreSQL transactions
   - PostgreSQL database size

4. **Log Dashboard** (`log-dashboard.json`)
   - Application logs
   - Error logs filtered view

### Importing Dashboards

#### Method 1: Via Grafana UI

1. Access Grafana UI
2. Go to Dashboards → Import
3. Upload JSON file or paste JSON content
4. Select Prometheus and Loki data sources
5. Click Import

#### Method 2: Via API

```bash
# Import dashboard via API
curl -X POST http://admin:admin@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @grafana/dashboards/application-dashboard.json
```

#### Method 3: Automatic Provisioning

For automatic dashboard provisioning, create a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
data:
  application-dashboard.json: |
    <dashboard JSON content>
```

## Metrics Collection

### Application Metrics

To expose metrics from applications:

#### Frontend (if using metrics library)

```javascript
// Example: Using prom-client for Node.js
const client = require('prom-client');
const register = new client.Registry();

// Add default metrics
client.collectDefaultMetrics({ register });

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

#### Backend

```javascript
// Example: Using prom-client
const client = require('prom-client');
const register = new client.Registry();

// Create custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

### Database Metrics

#### Redis Exporter

Deploy Redis exporter as a sidecar:

```yaml
containers:
- name: redis-exporter
  image: oliver006/redis_exporter:latest
  ports:
  - containerPort: 9121
```

#### PostgreSQL Exporter

Deploy PostgreSQL exporter as a sidecar:

```yaml
containers:
- name: postgres-exporter
  image: quay.io/prometheuscommunity/postgres-exporter:latest
  env:
  - name: DATA_SOURCE_NAME
    value: "postgresql://user:pass@localhost:5432/db?sslmode=disable"
  ports:
  - containerPort: 9187
```

## Alerting (Optional)

### Prometheus Alertmanager

#### Deploy Alertmanager

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.25.0
        args:
          - '--config.file=/etc/alertmanager/alertmanager.yml'
```

#### Alert Rules

Create alert rules in Prometheus:

```yaml
groups:
- name: finch-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} errors per second"

  - alert: PodRestarting
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod is restarting frequently"
      description: "Pod {{ $labels.pod }} is restarting"

  - alert: HighMemoryUsage
    expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage"
      description: "Container {{ $labels.container }} is using {{ $value }}% memory"
```

## Monitoring Best Practices

### 1. Resource Monitoring

- Monitor CPU and memory usage
- Set up alerts for resource exhaustion
- Track resource trends over time

### 2. Application Health

- Monitor request rates
- Track error rates
- Measure response times
- Monitor active connections

### 3. Database Monitoring

- Track connection counts
- Monitor query performance
- Watch for slow queries
- Monitor database size

### 4. Log Management

- Centralize all logs
- Use structured logging
- Set appropriate log levels
- Implement log rotation

### 5. Alerting

- Set up alerts for critical issues
- Use appropriate alert thresholds
- Configure alert notification channels
- Review and tune alerts regularly

## Troubleshooting

### Prometheus Not Scraping

1. Check Prometheus targets:
   ```bash
   # Access Prometheus UI
   # Go to Status → Targets
   ```

2. Verify service discovery:
   ```bash
   kubectl get pods --show-labels
   ```

3. Check Prometheus logs:
   ```bash
   kubectl logs -l app=prometheus
   ```

### Loki Not Receiving Logs

1. Check Promtail pods:
   ```bash
   kubectl get pods -l app=promtail
   kubectl logs -l app=promtail
   ```

2. Verify Loki is running:
   ```bash
   kubectl get pods -l app=loki
   curl http://localhost:3100/ready
   ```

3. Check Loki configuration:
   ```bash
   kubectl get configmap loki-config -o yaml
   ```

### Grafana Cannot Access Data Sources

1. Verify data source URLs:
   - Prometheus: `http://prometheus-service:9090`
   - Loki: `http://loki-service:3100`

2. Check service names:
   ```bash
   kubectl get svc | grep -E 'prometheus|loki'
   ```

3. Test connectivity from Grafana pod:
   ```bash
   kubectl exec -it <grafana-pod> -- wget -O- http://prometheus-service:9090/api/v1/status/config
   ```

### Dashboards Not Loading

1. Verify dashboard JSON is valid
2. Check data source names match
3. Verify metrics/logs are available
4. Check Grafana logs:
   ```bash
   kubectl logs -l app=grafana
   ```

## Production Considerations

### Scaling

- **Prometheus**: Consider federation for large clusters
- **Loki**: Use multiple instances with object storage
- **Grafana**: Can scale horizontally (read-only mode)

### High Availability

- Run multiple Prometheus instances
- Use remote write for long-term storage
- Implement Loki clustering
- Use Grafana HA setup

### Storage

- Use object storage (S3, GCS) for long-term retention
- Implement retention policies
- Monitor storage usage
- Set up automated cleanup

### Security

- Change default Grafana credentials
- Enable authentication
- Use TLS for all connections
- Implement RBAC
- Restrict network access

## Quick Start

### Deploy All Monitoring Components

```bash
# Deploy Prometheus
kubectl apply -f kubernetes/monitoring/prometheus-configmap.yaml
kubectl apply -f kubernetes/monitoring/prometheus-deployment.yaml

# Deploy Loki and Promtail
kubectl apply -f kubernetes/monitoring/loki-configmap.yaml
kubectl apply -f kubernetes/monitoring/promtail-configmap.yaml
kubectl apply -f kubernetes/monitoring/loki-deployment.yaml

# Deploy Grafana
kubectl apply -f kubernetes/monitoring/grafana-deployment.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=prometheus --timeout=300s
kubectl wait --for=condition=ready pod -l app=loki --timeout=300s
kubectl wait --for=condition=ready pod -l app=grafana --timeout=300s

# Access Grafana
kubectl port-forward svc/grafana-service 3000:3000
# Open http://localhost:3000 (admin/admin)
```

### Import Dashboards

1. Access Grafana UI
2. Navigate to Dashboards → Import
3. Import each dashboard from `grafana/dashboards/`
4. Configure data sources if needed

## Next Steps

1. Set up application metrics instrumentation
2. Configure alerting rules
3. Set up notification channels (email, Slack, PagerDuty)
4. Implement log aggregation best practices
5. Configure long-term storage for metrics
6. Set up dashboard sharing and permissions
7. Implement monitoring for monitoring stack itself
