# Pack 2 Quick Start Guide

This guide will help you quickly deploy Pack 2 for DEV team monitoring.

## Prerequisites

- Kubernetes cluster (1.19+)
- kubectl configured
- Pack 1 already deployed (or deploy simultaneously)
- Sufficient resources (see [README.md](README.md))

## Quick Deploy

### Step 1: Deploy Pack 2 with Helm

```bash
# Deploy Pack 2 using the Helm chart
helm upgrade --install pack-two charts/qubership-monitoring-operator \
  --namespace monitoring \
  --create-namespace \
  --values examples/pack-two/values.yaml \
  --wait
```

### Step 2: Verify Deployment

```bash
# Check all Pack 2 components
kubectl get all -n monitoring -l pack=two

# Expected output should include:
# - vmagent-pack-two-vmagent
# - vmsingle-pack-two-vmsingle
# - grafana-deployment-pack-two-grafana
```

### Step 3: Access Grafana

```bash
# Port forward to Grafana
kubectl port-forward -n monitoring svc/grafana-service-pack-two-grafana 3000:3000

# Open browser to http://localhost:3000
# Login: admin / admin
```

### Step 4: Deploy ServiceMonitors for Your Apps

Create a ServiceMonitor for your application:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: monitoring
    pack: two
  name: my-app
  namespace: my-namespace
spec:
  endpoints:
    - interval: 30s
      port: metrics
  selector:
    matchLabels:
      app: my-app
EOF
```

### Step 5: Verify Metrics Collection

```bash
# Port forward to VMAgent
kubectl port-forward -n monitoring svc/vmagent-pack-two-vmagent 8429:8429

# Open http://localhost:8429/targets to see discovered targets
```

## Alternative: Deploy with kubectl

If you prefer to deploy manifests directly:

```bash
# Deploy VictoriaMetrics components
kubectl apply -f examples/pack-two/manifests/pack-two-vmsingle.yaml
kubectl apply -f examples/pack-two/manifests/pack-two-vmagent.yaml

# Deploy Grafana
kubectl apply -f examples/pack-two/manifests/pack-two-grafana.yaml
kubectl apply -f examples/pack-two/manifests/pack-two-grafana-datasource.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l pack=two -n monitoring --timeout=300s
```

## Common Issues

### Issue: VMAgent not discovering ServiceMonitors

**Solution:** Ensure ServiceMonitors have the `pack: two` label:
```yaml
metadata:
  labels:
    pack: two
    app.kubernetes.io/component: monitoring
```

### Issue: Grafana shows no data

**Solution:** Check datasource configuration:
```bash
kubectl get grafanadatasource -n monitoring pack-two-victoriametrics -o yaml
```

Ensure it points to: `http://vmsingle-pack-two-vmsingle.monitoring.svc:8429`

### Issue: Pods in CrashLoopBackOff

**Solution:** Check logs:
```bash
kubectl logs -n monitoring -l pack=two --tail=50
```

Often caused by insufficient resources or misconfiguration.

## Next Steps

1. **Add more ServiceMonitors** - Monitor your applications by creating ServiceMonitors with `pack: two` label
2. **Create Dashboards** - Build custom Grafana dashboards (see [dashboards/](dashboards/) for examples)
3. **Configure Alerts** - Set up alerts using VMAlert
4. **Scale Resources** - Adjust resource limits based on your workload (edit `values.yaml`)

## Testing Multi-Pack Setup

To verify Pack 2 is collecting metrics from both Pack 1 and Pack 2:

```bash
# Access Grafana
kubectl port-forward -n monitoring svc/grafana-service-pack-two-grafana 3000:3000

# In Grafana Explore, run these queries:
# 1. Check Pack 1 infrastructure metrics:
node_cpu_seconds_total

# 2. Check Pack 2 app metrics (if you have apps monitored):
up{pack="two"}

# 3. Check all metrics with pack label:
{pack=~"one|two"}
```

## Cleanup

To remove Pack 2:

```bash
# If deployed with Helm
helm uninstall pack-two -n monitoring

# If deployed with kubectl
kubectl delete -f examples/pack-two/manifests/
kubectl delete -f examples/pack-two/service-monitors/
kubectl delete -f examples/pack-two/dashboards/
```

## Need Help?

- See [README.md](README.md) for detailed documentation
- Check [Troubleshooting section](README.md#troubleshooting)
- Review [examples](../../docs/examples/)
