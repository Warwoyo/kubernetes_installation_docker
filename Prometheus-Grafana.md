# Kubernetes Monitoring with Prometheus and Grafana

This guide provides comprehensive instructions for setting up monitoring in your Kubernetes cluster to collect metrics for autoscaling decisions and operational insights.

## Table of Contents
- Installing Metrics Server
- Setting Up Prometheus and Grafana
- Accessing and Configuring Grafana
- Instrumenting Your Application
- Creating Custom Dashboards
- Setting Up Horizontal Pod Autoscaler (HPA)
- Load Testing for Autoscaling
- Troubleshooting

## Installing Metrics Server

Metrics Server collects resource metrics from Kubelets and exposes them in the Kubernetes API server for use by the Horizontal Pod Autoscaler.

```bash
git clone https://github.com/Widhi-yahya/kubernetes_installation_docker.git
cd kubernetes_installation_docker/
kubectl apply -f metrics-server.yaml

# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Test that metrics are being collected
kubectl top nodes
kubectl top pods -A
```

## Setting Up Prometheus and Grafana

Prometheus is used for collecting and storing metrics, while Grafana provides visualization.

### Installing with Helm

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create a monitoring namespace
kubectl create namespace monitoring

# Install Prometheus stack (includes Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin \
  --set grafana.service.type=NodePort

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### Verify Prometheus Components

```bash
# Check Prometheus components
kubectl get pods -n monitoring | grep prometheus

# Check Alertmanager
kubectl get pods -n monitoring | grep alertmanager

# Check Grafana
kubectl get pods -n monitoring | grep grafana

# Get Grafana service details
kubectl get svc -n monitoring prometheus-grafana
```

## Accessing and Configuring Grafana

After deploying the Prometheus stack, you can access Grafana to visualize your metrics.

```bash
# Get Grafana service NodePort
export GRAFANA_PORT=$(kubectl get svc -n monitoring prometheus-grafana -o jsonpath='{.spec.ports[0].nodePort}')
echo "Access Grafana at http://<your-node-ip>:$GRAFANA_PORT"

# Login credentials:
# Username: admin
# Password: admin (or the password you set in the Helm installation)
```

## Instrumenting Your Application

To get custom metrics from your Node.js application, add Prometheus instrumentation:

### For Node.js Applications:

```javascript
// Add these to your package.json dependencies
// "prom-client": "^14.0.1",
// "express-prom-bundle": "^6.4.1"

// In your server.js:
const promBundle = require("express-prom-bundle");

// Add prometheus middleware
const metricsMiddleware = promBundle({
  includeMethod: true,
  includePath: true,
  promClient: {
    collectDefaultMetrics: {
      // Collect every 5 seconds
      timeout: 5000
    }
  }
});

// Use the middleware early in your Express app
app.use(metricsMiddleware);

// Your metrics will be available at the /metrics endpoint
```

### ServiceMonitor for Your Applications

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: login-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: login-app
  endpoints:
  - port: http  # Make sure this matches your service port name
    interval: 15s
    path: /metrics
  namespaceSelector:
    matchNames:
    - default  # Namespace where your app is running
```

Apply this configuration:

```bash
kubectl apply -f login-app-monitor.yaml
```

## Creating Custom Dashboards

### Import Built-in Kubernetes Dashboards

1. Access Grafana UI
2. Navigate to "+" icon on the left sidebar
3. Select "Import"
4. Enter the following dashboard IDs:
   - 6417 (Kubernetes Cluster)
   - 8588 (Kubernetes Deployment)
   - 11663 (Kubernetes Pod Monitoring)

### Create Network Metrics Dashboard

In Grafana:

1. Create a new dashboard
2. Add a panel for Network Received:
   ```
   sum(rate(container_network_receive_bytes_total{namespace="default",pod=~"login-app.*"}[5m])) by (pod)
   ```

3. Add a panel for Network Transmitted:
   ```
   sum(rate(container_network_transmit_bytes_total{namespace="default",pod=~"login-app.*"}[5m])) by (pod)
   ```

4. Add a panel for CPU Usage:
   ```
   sum(rate(container_cpu_usage_seconds_total{namespace="default",pod=~"login-app.*"}[5m])) by (pod)
   ```

5. Add a panel for Memory Usage:
   ```
   sum(container_memory_working_set_bytes{namespace="default",pod=~"login-app.*"}) by (pod)
   ```

## Setting Up Horizontal Pod Autoscaler (HPA)

### CPU-based HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: login-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: login-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

Apply this configuration:

```bash
kubectl apply -f cpu-hpa.yaml
```

### Custom Metrics HPA (Network-based)

First, install Prometheus Adapter:

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --values - <<EOF
prometheus:
  url: http://prometheus-operated.monitoring.svc
  port: 9090
rules:
  default: false
  custom:
  - seriesQuery: 'container_network_receive_bytes_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_total"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)'
EOF
```

Create network-based HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: login-app-network-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: login-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: container_network_receive_bytes_per_second
      target:
        type: AverageValue
        averageValue: 1000000  # 1MB/s
```

Apply this configuration:

```bash
kubectl apply -f network-hpa.yaml
```

## Load Testing for Autoscaling

To test autoscaling, generate load on your application:

```bash
# Install hey load testing tool
go install github.com/rakyll/hey@latest

# Run a load test against your application
hey -z 5m -c 50 http://<node-ip>:30080/login

# Watch HPA in action
kubectl get hpa -w

# Monitor pods scaling up/down
kubectl get pods -w
```

## Monitoring Autoscaling

```bash
# Get HPA status
kubectl get hpa

# Get detailed HPA description
kubectl describe hpa login-app-hpa

# Check current metric values
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" | jq .
```

## Troubleshooting

### Metrics Server Issues

```bash
# Check metrics-server logs
kubectl logs -n kube-system -l k8s-app=metrics-server

# Verify the API service
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

# Restart metrics-server if needed
kubectl rollout restart deployment metrics-server -n kube-system
```

### Prometheus Issues

```bash
# Check Prometheus pods
kubectl get pods -n monitoring | grep prometheus

# Check Prometheus logs
kubectl logs -n monitoring -l app=prometheus

# Check targets in Prometheus UI
# Port-forward Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Then access http://localhost:9090/targets in your browser
```

### Grafana Issues

```bash
# Reset admin password if needed
kubectl exec -it -n monitoring $(kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}') -- grafana-cli admin reset-admin-password admin
```

This comprehensive monitoring setup will provide you with the metrics needed to make informed autoscaling decisions and gain insights into your application's performance.

Similar code found with 2 license types