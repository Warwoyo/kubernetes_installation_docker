# Load Balancing for Login Web Application

This guide demonstrates how to set up load balancing for the login web application to distribute traffic across multiple frontend containers.

## Introduction

Your current deployment already has multiple replicas (`replicas: 2`) defined in the web deployment, which means Kubernetes is running two pods. The default Kubernetes service provides basic load balancing between these pods.

In this guide, we'll enhance this with:
1. A dedicated load balancer configuration
2. Testing to verify the load balancing is working

## 1. Modify Web Deployment for Better Load Balancing

First, let's update the web deployment to ensure we have proper identification for each pod:

```bash
cat > k8s/web-deployment-lb.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: login-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: login-app
  template:
    metadata:
      labels:
        app: login-app
    spec:
      containers:
      - name: login-app
        image: login-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_DATABASE
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```
# Load Balancing for Login Web Application

This guide demonstrates how to set up load balancing for the login web application to distribute traffic across multiple frontend containers.

## Introduction

Your current deployment already has multiple replicas (`replicas: 2`) defined in the web deployment, which means Kubernetes is running two pods. The default Kubernetes service provides basic load balancing between these pods.

In this guide, we'll enhance this with:
1. A dedicated load balancer configuration
2. Testing to verify the load balancing is working

## 1. Modify Web Deployment for Better Load Balancing

First, let's update the web deployment to ensure we have proper identification for each pod:

```bash
cat > k8s/web-deployment-lb.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: login-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: login-app
  template:
    metadata:
      labels:
        app: login-app
    spec:
      containers:
      - name: login-app
        image: login-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_DATABASE
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```

## 2. Add Server Identification to the Web Application

To better visualize load balancing, let's modify the web application to display which server is handling the request. Create a patch file:

```bash
cat > app/server-patch.js << 'EOF'
const os = require('os');
const serverInfo = {
  hostname: os.hostname(),
  podName: process.env.POD_NAME || 'unknown',
  nodeName: process.env.NODE_NAME || 'unknown'
};

// Add this after the health route
app.get('/server-info', (req, res) => {
  res.json(serverInfo);
});

// Insert this line before app.listen
app.use((req, res, next) => {
  res.setHeader('X-Served-By', serverInfo.podName);
  next();
});
EOF
```

Apply this patch to your server.js file, then rebuild the Docker image:

```bash
# Rebuild the Docker image with server identification
cd app
# Add the server info code to your server.js
cat server-patch.js >> server.js

docker build -t login-app:latest .
docker save login-app:latest > login-app.tar

# If needed, transfer to worker nodes and load
# scp login-app.tar user@worker-node:/home/user/
# On worker nodes: docker load < login-app.tar
```

## 3. Apply the Updated Web Deployment

```bash
kubectl apply -f k8s/web-deployment-lb.yaml
```

## 4. Create an Nginx Ingress Controller for Advanced Load Balancing

```bash
# Create namespace for ingress-nginx
kubectl create namespace ingress-nginx

# Add the ingress-nginx repository to Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress-nginx using Helm
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.nodeSelector."kubernetes\\.io/hostname"=<your-worker-node-name> \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30081
```

Replace `<your-worker-node-name>` with the name of your worker node (run `kubectl get nodes` to find it).

## 5. Create Ingress Resource for the Application

```bash
cat > k8s/login-app-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: login-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: login-app
            port:
              number: 80
EOF
```

Apply the ingress resource:

```bash
kubectl apply -f k8s/login-app-ingress.yaml
```

## 6. Update Service to Work with Ingress

Make sure your service is correctly defined to work with the ingress:

```bash
cat > k8s/web-service-lb.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: login-app
spec:
  selector:
    app: login-app
  ports:
  - port: 80
    targetPort: 3000
EOF
```

Apply the updated service:

```bash
kubectl apply -f k8s/web-service-lb.yaml
```

## 7. Testing the Load Balancer

To verify the load balancing is working:

```bash
# Make multiple requests to see different pod responses
for i in {1..10}; do 
  curl -s http://10.34.7.115:30081/server-info | jq .
  sleep 1
done
```

Replace `10.34.7.115` with your node IP address. You should see responses from different pods.

## 8. Access the Application

You can now access your application through the Ingress controller:

```
http://10.34.7.115:30081
```

## 9. Visualizing Load Balancing

To better visualize the load balancing, add a small JavaScript code to your application's frontend to show which server processed the request:

```javascript
// Add this to your dashboard.js or other frontend scripts
fetch('/server-info')
  .then(response => response.json())
  .then(data => {
    const serverInfoDiv = document.createElement('div');
    serverInfoDiv.style.position = 'fixed';
    serverInfoDiv.style.bottom = '10px';
    serverInfoDiv.style.right = '10px';
    serverInfoDiv.style.padding = '5px';
    serverInfoDiv.style.background = 'rgba(0,0,0,0.6)';
    serverInfoDiv.style.color = 'white';
    serverInfoDiv.style.fontSize = '12px';
    serverInfoDiv.style.borderRadius = '3px';
    serverInfoDiv.textContent = `Server: ${data.podName}`;
    document.body.appendChild(serverInfoDiv);
  });
```

## 10. Advanced Load Balancing Configuration

For more advanced load balancing features:

### Session Affinity (Sticky Sessions)

If you need user sessions to stick to the same pod:

```bash
kubectl annotate ingress login-app-ingress nginx.ingress.kubernetes.io/affinity="cookie"
kubectl annotate ingress login-app-ingress nginx.ingress.kubernetes.io/session-cookie-name="SERVERID"
```

### Traffic Weighting

To direct different percentages of traffic to different versions of your application:

```bash
# Create another deployment with a newer version
kubectl apply -f k8s/web-deployment-v2.yaml

# Create a service for v2
kubectl apply -f k8s/web-service-v2.yaml

# Update ingress to split traffic
kubectl apply -f k8s/weighted-ingress.yaml
```

## 11. Troubleshooting

If you encounter issues:

```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check if ingress is properly configured
kubectl describe ingress login-app-ingress

# Check endpoints for the service
kubectl get endpoints login-app
```

This completes the setup of load balancing for the login web application.
```
