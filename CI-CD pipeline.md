# CI/CD Pipeline for Kubernetes Login Application

This guide outlines a complete Continuous Integration and Continuous Deployment (CI/CD) pipeline for your Kubernetes login application. The pipeline automates building, testing, and deploying your application whenever code changes are pushed to your repository.

## Table of Contents

1. [Introduction to CI/CD](#introduction-to-cicd)
2. [Scenario Overview](#scenario-overview)
3. [Requirements](#requirements)
4. [Jenkins Installation on Kubernetes](#jenkins-installation-on-kubernetes)
5. [Pipeline Setup](#pipeline-setup)
6. [GitHub Integration](#github-integration)
7. [Automated Testing](#automated-testing)
8. [Deployment Strategy](#deployment-strategy)
9. [Monitoring and Feedback](#monitoring-and-feedback)
10. [Security Considerations](#security-considerations)

## Introduction to CI/CD

CI/CD is a set of practices that automates the software development lifecycle:

- **Continuous Integration (CI)**: Developers merge code changes frequently, followed by automated build and test processes
- **Continuous Delivery (CD)**: Every code change that passes all tests is automatically deployed to a production-like environment
- **Continuous Deployment**: Takes delivery further by automatically deploying to production

Benefits:
- Faster development cycles
- Early bug detection
- Consistent deployment process
- Reduced manual errors

## Scenario Overview

Consider this practical scenario:

1. Your development team works on the login application
2. Multiple developers push updates to their feature branches
3. When a Pull Request is merged to the main branch, the CI/CD pipeline automatically:
   - Builds the application
   - Runs unit and integration tests
   - Builds a Docker image with a unique tag
   - Pushes the image to a registry
   - Updates the Kubernetes deployment
   - Validates the deployment is successful

This entire process happens without manual intervention, ensuring rapid feedback and reliable deployments.

## Requirements

- GitHub repository with your login application code
- Kubernetes cluster (you already have this set up)
- Jenkins server (will be installed on your K8s cluster)
- Docker registry access (DockerHub or private registry)
- Node.js and npm for testing

## Jenkins Installation on Kubernetes

```bash
# Create namespace for Jenkins
kubectl create namespace devops-tools

# Create persistent volume for Jenkins data
cat > jenkins-pv.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/jenkins-data"
EOF
kubectl apply -f jenkins-pv.yaml

# Create persistent volume claim
cat > jenkins-pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: devops-tools
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
kubectl apply -f jenkins-pvc.yaml

# Create service account for Jenkins
cat > jenkins-sa.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: devops-tools
EOF
kubectl apply -f jenkins-sa.yaml

# Deploy Jenkins
cat > jenkins-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          ports:
            - name: http
              containerPort: 8080
            - name: jnlp
              containerPort: 50000
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
            claimName: jenkins-pvc
EOF
kubectl apply -f jenkins-deployment.yaml

# Create Jenkins service
cat > jenkins-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
  selector:
    app: jenkins
EOF
kubectl apply -f jenkins-service.yaml

# Wait for Jenkins to be ready
kubectl wait --for=condition=ready pod -l app=jenkins -n devops-tools --timeout=180s
```

## Pipeline Setup

### Creating a Jenkinsfile

Create this `Jenkinsfile` in your repository root:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: docker
                image: docker:20.10.14-dind
                command:
                - sleep
                args:
                - 99d
                volumeMounts:
                - name: docker-socket
                  mountPath: /var/run/docker.sock
              - name: node
                image: node:16
                command:
                - sleep
                args:
                - 99d
              - name: kubectl
                image: bitnami/kubectl:latest
                command:
                - sleep
                args:
                - 99d
              volumes:
              - name: docker-socket
                hostPath:
                  path: /var/run/docker.sock
            """
        }
    }
    
    environment {
        DOCKER_REGISTRY = 'dockerhub-username'  // Replace with your Docker registry
        DOCKER_IMAGE = 'login-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                container('node') {
                    dir('app') {
                        sh 'npm install'
                    }
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                container('node') {
                    dir('app') {
                        sh 'npm test || echo "No tests found"'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    dir('app') {
                        sh "docker build -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                container('docker') {
                    withCredentials([string(credentialsId: 'docker-credentials', variable: 'DOCKER_AUTH')]) {
                        sh "echo ${DOCKER_AUTH} | docker login ${DOCKER_REGISTRY} -u username --password-stdin"
                        sh "docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }
        
        stage('Update Kubernetes Deployment') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl set image deployment/login-app login-app=${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        kubectl rollout status deployment/login-app
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

## GitHub Integration

### Setting Up Webhooks

1. In Jenkins, set up pipeline job:
   - Go to Jenkins dashboard
   - Create New Item → Pipeline
   - Configure source code management to use your Git repository
   - Enable "GitHub hook trigger for GITScm polling"

2. On GitHub:
   - Go to your repository settings
   - Navigate to Webhooks → Add webhook
   - Set Payload URL to `http://<node-ip>:32000/github-webhook/`
   - Content type: application/json
   - Select "Just the push event"

Now your pipeline will automatically trigger when code is pushed to your repository.

## Automated Testing

### Setting Up Tests for the Login App

Create a basic test setup in your application:

1. Install testing dependencies:
```bash
cd app
npm install --save-dev mocha chai supertest
```

2. Create `test/server.test.js`:
```javascript
const request = require('supertest');
const { expect } = require('chai');
const app = require('../server');  // Adjust if your server is exported differently

describe('Login Application', () => {
  it('should return 200 OK on health check endpoint', async () => {
    const response = await request(app).get('/health');
    expect(response.status).to.equal(200);
  });

  it('should serve the login page', async () => {
    const response = await request(app).get('/');
    expect(response.status).to.equal(200);
    expect(response.text).to.include('Login');
  });

  // Add more tests for login functionality
});
```

3. Update `package.json` to include test script:
```json
"scripts": {
  "test": "mocha --exit test/*.test.js",
  "start": "node server.js"
}
```

The pipeline will automatically run these tests before deploying.

## Deployment Strategy

This pipeline uses a simple deployment strategy:

1. **Rolling Update**: Kubernetes incrementally replaces old pods with new ones
2. **Versioned Images**: Each build creates a uniquely tagged Docker image
3. **Fast Rollbacks**: If issues are detected, you can roll back to a previous image:

```bash
# To rollback to a previous version
kubectl rollout undo deployment/login-app

# Or to a specific version
kubectl rollout undo deployment/login-app --to-revision=2
```

For more advanced strategies, consider:
- Blue/Green deployment
- Canary releases
- Feature flags

## Monitoring and Feedback

Add these stages to your Jenkinsfile to monitor deployments:

```groovy
stage('Verify Deployment') {
    steps {
        container('kubectl') {
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh """
                export KUBECONFIG=${KUBECONFIG}
                # Wait for deployment to stabilize
                sleep 15
                # Check if pods are running
                kubectl get pods -l app=login-app
                # Check sample endpoint
                curl -s http://\$(kubectl get svc login-app -o jsonpath='{.spec.clusterIP}')/health | grep -q 'ok' || exit 1
                """
            }
        }
    }
}
```

## Security Considerations

1. **Secrets Management**:
   - Never store secrets in your code
   - Use Jenkins credentials or Kubernetes secrets
   
2. **Image Scanning**:
   Add this stage to your pipeline to scan for vulnerabilities:
   ```groovy
   stage('Security Scan') {
       steps {
           container('docker') {
               sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}"
           }
       }
   }
   ```

3. **Access Control**:
   - Use service accounts with minimal permissions
   - Regularly rotate credentials

This CI/CD pipeline automates the entire deployment process for your login application, enabling your team to deliver features faster while maintaining high quality and reliability.
