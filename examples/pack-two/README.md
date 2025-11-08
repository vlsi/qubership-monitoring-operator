# Pack 2 - DEV Team Monitoring Package

This directory contains the configuration and examples for **Pack 2**, a secondary monitoring stack designed specifically for development teams. Pack 2 operates alongside Pack 1 (the primary monitoring infrastructure) and can collect metrics from both packs while maintaining its own independent storage and visualization.

## Overview

Pack 2 provides development teams with:
- **Dedicated VictoriaMetrics instance** for storing team-specific metrics
- **Independent Grafana instance** for custom dashboards and visualizations
- **Multi-pack metrics collection** - can scrape metrics from both Pack 1 and Pack 2 labeled resources
- **Coexistence support** - runs alongside Pack 1 without conflicts

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐           ┌──────────────┐                   │
│  │   Pack 1     │           │   Pack 2     │                   │
│  │ (Platform)   │           │ (DEV Team)   │                   │
│  ├──────────────┤           ├──────────────┤                   │
│  │              │           │              │                   │
│  │ VMAgent      │           │ VMAgent ─────┼─┐                 │
│  │ VMSingle     │           │ VMSingle     │ │                 │
│  │ Grafana      │           │ Grafana      │ │                 │
│  │              │           │              │ │                 │
│  └──────┬───────┘           └──────┬───────┘ │                 │
│         │                          │         │                 │
│         │    ┌─────────────────────┘         │                 │
│         │    │                               │                 │
│  ┌──────▼────▼───────────────────────────────▼─────┐           │
│  │        Shared Infrastructure                     │           │
│  │  • Node Exporter (pack=one label)               │           │
│  │  • Kube-State-Metrics (pack=one label)          │◄──────────┤
│  │  • Kubernetes API metrics                       │  Pack 2   │
│  └──────────────────────────────────────────────────┘  scrapes  │
│                                                        both      │
│  ┌──────────────────────────────────────────────────┐  pack=one │
│  │        Pack 2 Specific Applications              │  and      │
│  │  • Custom apps with pack=two ServiceMonitors    │◄──pack=two│
│  └──────────────────────────────────────────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Key Features

### 1. Label-Based Pack Separation

Pack 2 uses Kubernetes label selectors to identify which metrics to collect:

- **pack=one**: Infrastructure metrics from Pack 1 (shared resources)
- **pack=two**: Application-specific metrics for dev teams

Pack 2's VMAgent is configured to scrape resources with **either** label, allowing it to collect both infrastructure and application metrics.

### 2. Independent Storage

Pack 2 has its own VictoriaMetrics Single instance (`pack-two-vmsingle`) that stores metrics independently from Pack 1. This ensures:
- No impact on Pack 1's performance
- Customizable retention periods
- Isolated resource allocation

### 3. Dedicated Grafana

Pack 2 includes a separate Grafana instance that:
- Queries Pack 2's VictoriaMetrics instance
- Discovers dashboards with `pack: two` label
- Provides dev teams with customized views

## Directory Structure

```
examples/pack-two/
├── README.md                           # This file
├── values.yaml                         # Helm chart values for Pack 2
├── manifests/                          # Kubernetes manifests
│   ├── pack-two-vmagent.yaml         # VMAgent configuration
│   ├── pack-two-vmsingle.yaml        # VMSingle storage configuration
│   ├── pack-two-grafana.yaml         # Grafana instance configuration
│   └── pack-two-grafana-datasource.yaml  # Grafana datasource config
├── service-monitors/                   # ServiceMonitor examples
│   ├── example-app-service-monitor.yaml
│   ├── node-exporter-service-monitor.yaml
│   └── example-pod-monitor.yaml
└── dashboards/                         # Grafana dashboard examples
    └── dev-team-overview-dashboard.yaml
```

## Deployment

### Prerequisites

1. **Pack 1 must be deployed** - Pack 2 depends on shared infrastructure (Node Exporter, Kube-State-Metrics)
2. **VictoriaMetrics Operator** must be installed
3. **Grafana Operator** must be installed
4. Sufficient cluster resources for additional monitoring stack

### Option 1: Deploy via Helm (Recommended)

Deploy Pack 2 using the qubership-monitoring-operator Helm chart:

```bash
# Install Pack 2 alongside Pack 1
helm upgrade --install pack-two ./charts/qubership-monitoring-operator \
  --namespace monitoring \
  --create-namespace \
  --values examples/pack-two/values.yaml
```

### Option 2: Deploy via kubectl

Apply the manifests directly:

```bash
# Create namespace if it doesn't exist
kubectl create namespace monitoring

# Deploy VictoriaMetrics components
kubectl apply -f examples/pack-two/manifests/pack-two-vmsingle.yaml
kubectl apply -f examples/pack-two/manifests/pack-two-vmagent.yaml

# Deploy Grafana
kubectl apply -f examples/pack-two/manifests/pack-two-grafana.yaml
kubectl apply -f examples/pack-two/manifests/pack-two-grafana-datasource.yaml

# Deploy example ServiceMonitors (optional)
kubectl apply -f examples/pack-two/service-monitors/

# Deploy example dashboards (optional)
kubectl apply -f examples/pack-two/dashboards/
```

### Verification

Check that all Pack 2 components are running:

```bash
# Check VMAgent
kubectl get vmagent -n monitoring | grep pack-two

# Check VMSingle
kubectl get vmsingle -n monitoring | grep pack-two

# Check Grafana
kubectl get grafana -n monitoring | grep pack-two

# Check pods
kubectl get pods -n monitoring -l pack=two
```

## Configuration

### Configuring Pack 1 to Use Labels

For Pack 2 to work optimally, Pack 1's ServiceMonitors and PodMonitors should be labeled with `pack: one`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: monitoring
    pack: one  # Add this label
  name: node-exporter
```

### Creating Pack 2 ServiceMonitors

To monitor your applications with Pack 2, create ServiceMonitors with the `pack: two` label:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: monitoring
    pack: two  # Critical for Pack 2 discovery
  name: my-app
spec:
  endpoints:
    - interval: 30s
      port: metrics
  selector:
    matchLabels:
      app: my-app
```

See `service-monitors/` directory for complete examples.

### Creating Pack 2 Dashboards

Create Grafana dashboards that will be discovered by Pack 2's Grafana:

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  labels:
    app.kubernetes.io/component: monitoring
    pack: two  # Critical for Pack 2 Grafana discovery
  name: my-dashboard
spec:
  instanceSelector:
    matchLabels:
      pack: two
  json: |
    { ... dashboard JSON ... }
```

See `dashboards/` directory for complete examples.

## Multi-Pack Metrics Collection

Pack 2's VMAgent is configured to collect metrics from both packs using label selectors:

```yaml
serviceScrapeSelector:
  matchExpressions:
    - key: pack
      operator: In
      values: ["one", "two"]  # Scrapes both packs
```

This allows dev teams to:
- Monitor shared infrastructure (Node Exporter, Kube-State-Metrics from Pack 1)
- Monitor team-specific applications (labeled with pack=two)
- Create dashboards combining both infrastructure and application metrics

## Coexistence Validation

Pack 2 is designed to coexist with Pack 1 through:

### 1. Separate Resource Names
- VMAgent: `pack-two-vmagent` vs `k8s`
- VMSingle: `pack-two-vmsingle` vs `k8s`
- Grafana: `pack-two-grafana` vs `grafana`

### 2. Label-Based Selectors
- Pack 1 VMAgent: Can be configured to select only `pack=one` resources
- Pack 2 VMAgent: Selects both `pack=one` and `pack=two` resources

### 3. Independent Services
Each pack has its own:
- Service endpoints
- Storage volumes
- Resource quotas
- Ingress hosts (if configured)

### 4. Testing Coexistence

Verify both packs are operating independently:

```bash
# Check Pack 1 VMAgent targets
kubectl port-forward -n monitoring svc/vmagent-k8s 8429:8429
# Visit http://localhost:8429/targets

# Check Pack 2 VMAgent targets
kubectl port-forward -n monitoring svc/vmagent-pack-two-vmagent 8429:8429
# Visit http://localhost:8429/targets

# Verify different metrics are stored
# Pack 1 Grafana should query vmsingle-k8s
# Pack 2 Grafana should query vmsingle-pack-two-vmsingle
```

## Resource Requirements

### Recommended Resources for Pack 2

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|-----------|-------------|-----------|----------------|--------------|---------|
| VMAgent | 100m | 1 core | 500Mi | 2Gi | - |
| VMSingle | 200m | 2 cores | 500Mi | 4Gi | 50Gi |
| Grafana | 100m | 500m | 256Mi | 1Gi | - |
| VMAlert | 50m | 200m | 200Mi | 500Mi | - |
| VMAlertManager | 30m | 100m | 56Mi | 256Mi | - |

**Total minimum resources**: ~500m CPU, ~1.5Gi memory, 50Gi storage

## Accessing Pack 2 Services

### Via Port-Forward

```bash
# Access Pack 2 Grafana
kubectl port-forward -n monitoring svc/grafana-service-pack-two-grafana 3000:3000
# Open http://localhost:3000
# Default credentials: admin/admin

# Access Pack 2 VMSingle (API)
kubectl port-forward -n monitoring svc/vmsingle-pack-two-vmsingle 8429:8429
# Query API: http://localhost:8429/api/v1/query?query=up

# Access Pack 2 VMAgent (metrics and targets)
kubectl port-forward -n monitoring svc/vmagent-pack-two-vmagent 8429:8429
# View targets: http://localhost:8429/targets
```

### Via Ingress (Optional)

Configure ingress in `values.yaml` or manifests:

```yaml
grafana:
  ingress:
    install: true
    host: grafana-pack-two.example.com
```

## Customization

### Adjusting Retention Period

Modify `retentionPeriod` in `pack-two-vmsingle.yaml`:

```yaml
spec:
  retentionPeriod: "30d"  # Change from 14d to 30d
```

### Adjusting Scrape Intervals

Modify scrape intervals in `pack-two-vmagent.yaml`:

```yaml
spec:
  scrapeInterval: 60s  # Change from 30s to 60s
```

### Adding Custom Scrape Configs

Add additional scrape configurations:

```yaml
spec:
  additionalScrapeConfigs:
    key: prometheus-additional.yaml
    name: vm-additional-scrape-configs
```

## Troubleshooting

### VMAgent Not Discovering ServiceMonitors

**Check:**
1. ServiceMonitor has `pack: two` label
2. ServiceMonitor namespace is not excluded by namespace selectors
3. VMAgent logs: `kubectl logs -n monitoring -l app.kubernetes.io/name=vmagent -c vmagent`

### Grafana Not Showing Dashboards

**Check:**
1. GrafanaDashboard has `pack: two` label
2. GrafanaDashboard `instanceSelector` matches Grafana instance labels
3. Grafana logs: `kubectl logs -n monitoring -l app.kubernetes.io/name=grafana`

### No Metrics in Pack 2

**Check:**
1. VMAgent is running and scraping targets
2. VMSingle is receiving data
3. Verify datasource configuration in Grafana points to correct VMSingle URL
4. Check VMAgent targets: Port-forward and visit `/targets` endpoint

### Resource Exhaustion

If Pack 2 is consuming too many resources:
1. Reduce scrape frequency
2. Add metric relabeling to drop unnecessary metrics
3. Reduce VMSingle retention period
4. Lower resource limits (but monitor for OOMKills)

## Best Practices

1. **Label Everything**: Consistently use `pack: one` or `pack: two` labels on all monitoring resources
2. **Use Metric Relabeling**: Drop unnecessary metrics to reduce storage and query load
3. **Monitor the Monitors**: Set up alerts for VMAgent, VMSingle, and Grafana health
4. **Regular Cleanup**: Remove unused ServiceMonitors and dashboards
5. **Resource Limits**: Set appropriate resource requests and limits based on your workload
6. **Backup Configuration**: Backup Grafana dashboards and alert rules regularly

## Migration from Single Pack

If you're migrating from a single monitoring pack:

1. Deploy Pack 2 alongside existing monitoring
2. Gradually add `pack: two` labels to dev team ServiceMonitors
3. Update Pack 1 ServiceMonitors with `pack: one` labels
4. Configure Pack 1's VMAgent to use pack selectors (optional - for strict separation)
5. Migrate dashboards to Pack 2's Grafana

## Support and Contributions

For issues, questions, or contributions related to Pack 2:
- Create an issue in the main repository
- Refer to the main [qubership-monitoring-operator documentation](../../README.md)
- Check the [custom resources examples](../../docs/examples/custom-resources/)

## References

- [VictoriaMetrics Operator Documentation](https://docs.victoriametrics.com/operator/)
- [Grafana Operator Documentation](https://grafana.github.io/grafana-operator/)
- [Prometheus Operator API](https://prometheus-operator.dev/docs/operator/api/)
- [qubership-monitoring-operator Main Documentation](../../README.md)
