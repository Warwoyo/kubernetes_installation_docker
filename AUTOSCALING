# Kubernetes Login Application with Autoscaling

This README provides a step-by-step guide for deploying the Login Web Application with autoscaling capabilities in Kubernetes. The autoscaling will automatically adjust the number of replicas based on CPU load monitored by Prometheus.

## Table of Contents
- Prerequisites
- 1. Deploy Metrics Server
- 2. Deploy Prometheus and Grafana
- 3. Deploy MySQL Database
- 4. Deploy Web Application
- 5. Configure Autoscaling
- 6. Test Autoscaling
- 7. Monitor via Grafana
- 8. Troubleshooting

## Prerequisites

- Kubernetes cluster up and running
- `kubectl` configured to access your cluster
- Helm v3 installed
- Docker installed (for building images)

## 1. Deploy Metrics Server

Metrics Server is required for the Horizontal Pod Autoscaler to function:

```bash
# Apply metrics-server if not already deployed
kubectl apply -f metrics-server.yaml

# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Verify metrics API is working
kubectl top nodes
```

## 2. Deploy Prometheus and Grafana

Set up monitoring to track CPU metrics for autoscaling decisions:

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus stack with Grafana
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin \
  --set grafana.service.type=NodePort

# Verify installation
kubectl get pods -n monitoring

# Get Grafana service NodePort
export GRAFANA_PORT=$(kubectl get svc -n monitoring prometheus-grafana -o jsonpath='{.spec.ports[0].nodePort}')
echo "Access Grafana at http://<your-node-ip>:$GRAFANA_PORT"
# Login: admin/admin
```

## 3. Deploy MySQL Database

```bash
# Create a directory on your worker node for MySQL data
ssh user@worker-node "sudo mkdir -p /mnt/data && sudo chmod 777 /mnt/data"

# Deploy MySQL components
kubectl apply -f k8s/mysql-secret.yaml
kubectl apply -f k8s/mysql-pv.yaml
kubectl apply -f k8s/mysql-pvc.yaml
kubectl apply -f k8s/mysql-service.yaml
kubectl apply -f k8s/mysql-deployment.yaml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=180s
```

## 4. Deploy Web Application

```bash
# Build the Docker image (if not already built)
cd k8s-login-app/app
sudo docker build -t login-app:latest .
docker save login-app:latest > login-app.tar

# Transfer to worker nodes if needed
# scp login-app.tar user@worker-node:/tmp/
# ssh user@worker-node "docker load < /tmp/login-app.tar"

# Deploy web application
kubectl apply -f k8s/web-deployment.yaml
kubectl apply -f k8s/web-service.yaml

# Verify deployment
kubectl get pods -l app=login-app
```

## 5. Configure Autoscaling

Create an HPA configuration to automatically scale based on CPU metrics:

```bash
# Create HPA configuration file
cat > login-app-hpa.yaml << EOF
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
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 1
        periodSeconds: 60
      selectPolicy: Min
EOF

# Apply HPA configuration
kubectl apply -f login-app-hpa.yaml

# Verify HPA is created
kubectl get hpa
```


## 6. Update the Deployment with CPU Resource Requests

```bash
# Create a patch file for your deployment
cat > cpu-patch.yaml << EOF
spec:
  template:
    spec:
      containers:
      - name: login-app
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF

# Apply the patch to your deployment
kubectl patch deployment login-app --patch "$(cat cpu-patch.yaml)" --type=strategic
```

## 7. Verify the Resource Requests Were Applied

```bash
# Check the updated deployment specification
kubectl describe deployment login-app | grep -A 10 Resources:

# You should see output like:
# Resources:
#   Requests:
#     cpu: 100m
#     memory: 128Mi
#   Limits:
#     cpu: 500m
#     memory: 512Mi
```

## 8. Check HPA Status Again

```bash
# Give it a minute to gather metrics
sleep 60

# Check HPA status
kubectl get hpa login-app-hpa
kubectl describe hpa login-app-hpa
```


## 9. Test Autoscaling

Use a stress testing tool to generate artificial load and observe autoscaling behavior:

```bash
# Install Apache Bench (ab) for stress testing
# On Ubuntu: sudo apt install apache2-utils
# On macOS: brew install ab

# First, check current deployment status
kubectl get deployment login-app
kubectl get hpa login-app-hpa

# Run a stress test (replace with your node IP)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
ab -n 10000 -c 100 http://$NODE_IP:30080/login

# Watch HPA and pods during the stress test
# In a new terminal:
watch -n 1 "kubectl get hpa login-app-hpa && echo '' && kubectl get pods -l app=login-app"

# Check CPU metrics
kubectl top pods -l app=login-app

# For more intensive CPU load, run a test inside the pods:
kubectl exec -it $(kubectl get pod -l app=login-app -o jsonpath='{.items[0].metadata.name}') -- /bin/sh -c 'while true; do echo "Generating CPU load"; done'

# Monitor autoscaling (in another terminal)
kubectl get hpa login-app-hpa -w
```

## 10. Monitor via Grafana

Set up a dashboard to visualize the autoscaling metrics:

1. Access Grafana at `http://<node-ip>:<grafana-port>`
2. Log in with `admin` / `admin`
3. Create a new dashboard with the following panels:

   a. **CPU Utilization**
   ```
   sum(rate(container_cpu_usage_seconds_total{namespace="default",pod=~"login-app-.*"}[5m])) by (pod)
   ```

   b. **Memory Usage**
   ```
   sum(container_memory_working_set_bytes{namespace="default",pod=~"login-app-.*"}) by (pod)
   ```

   c. **Number of Replicas**
   ```
   count(kube_pod_info{namespace="default",pod=~"login-app-.*"})
   ```

   d. **HPA Status**
   ```
   kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="login-app-hpa"}
   ```

## 11. Troubleshooting

If autoscaling doesn't work as expected, check these common issues:

```bash
# Verify metrics are available
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" | jq .

# Check HPA description
kubectl describe hpa login-app-hpa

# Check Prometheus targets
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Then open http://localhost:9090/targets in your browser

# Verify the application has resource requests set
kubectl describe deployment login-app | grep -A 5 Requests

# Restart metrics-server if needed
kubectl rollout restart deployment metrics-server -n kube-system
```

Autoscaling should now be functioning, with pods scaling up when CPU utilization exceeds 80% and scaling down when it drops below 50%, maintaining between 2 and 4 replicas based on the current load.


# Accessing the Login Application from Client Machines

Yes, your application is already set up with load balancing capabilities, but let me explain all the available access methods:

## Current Access Methods

### 1. NodePort Service (Basic Access)
The login application is exposed via a NodePort service on port 30080:

```bash
# Check the NodePort service details
kubectl get svc login-app

# Example output:
# login-app   NodePort   10.x.x.x   <none>   80:30080/TCP   24h
```

**Access URL:** `http://<any-node-ip>:30080`

### 2. Ingress Controller (Load Balanced Access)
From your LB_DEPLOYMENT.md file, you have an Nginx Ingress controller set up that provides more advanced load balancing features:

```bash
# Check the Ingress details
kubectl get ing login-app-ingress

# Check the Ingress controller service
kubectl get svc -n ingress-nginx
```

**Access URL:** `http://<any-node-ip>:30081`

## Connecting from a Client Machine

To access from any client machine:

1. **Direct NodePort Access:**
   ```
   http://<kubernetes-node-ip>:30080
   ```

2. **Through Load Balancer (Ingress):**
   ```
   http://<kubernetes-node-ip>:30081
   ```

3. **If using DNS:**
   If you have configured DNS for your cluster, you can access using:
   ```
   http://your-configured-domain/
   ```

## Additional Configuration Options

### 1. External Load Balancer (Cloud Environments)

If running in a cloud provider (AWS, GCP, Azure), you can create a true external load balancer:

```bash
# Update the service type to LoadBalancer
kubectl patch svc login-app -p '{"spec":{"type":"LoadBalancer"}}'
```

### 2. MetalLB (For On-premises Clusters)

For on-premises clusters, you can install MetalLB to provide external load balancing:

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# Configure IP address pool (customize the addresses)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
EOF

# Configure L2 announcement
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

### 3. Verify Current Load Balancer Status

```bash
# Check if your ingress controller is running
kubectl get pods -n ingress-nginx

# Check if your services are properly mapped
kubectl get ing
```

To test if load balancing is working correctly between your pods:

```bash
# Run multiple requests to see different pods handling the requests
for i in {1..10}; do 
  curl -s http://<node-ip>:30081/server-info
  echo ""
  sleep 1
done
```

This should show requests being distributed across different pods, confirming that load balancing is working correctly.