# DevOps Mega Project - Complete CI/CD Pipeline Documentation

**Project Overview:** A comprehensive CI/CD solution with Infrastructure as Code, Automated CI/CD Pipeline, Kubernetes Deployment, and Monitoring.

**Architecture Components:** 4 main sections - Infrastructure, CI Pipeline, CD/Deployment, and Monitoring

---

## Table of Contents
1. [Terraform Mega Project (Infrastructure)](#terraform-mega-project---infrastructure)
2. [Mega Project CI (Continuous Integration)](#mega-project-ci---continuous-integration)
3. [Mega Project CD (Continuous Deployment)](#mega-project-cd---continuous-deployment)
4. [Monitoring & Observability](#monitoring--observability)

---

# TERRAFORM MEGA PROJECT - INFRASTRUCTURE


## URL : https://github.com/cybe44oot/Terraform-Mega-Project.git

## Repository: `terraform-mega-project`
**Purpose:** Infrastructure as Code for AWS EKS cluster and all supporting infrastructure

### Phase 1: Initial Setup & Prerequisites

#### Step 1: Create Infrastructure Server VM
- Launch an EC2 instance (Ubuntu 20.04 or later)
- Ensure it has appropriate IAM role for AWS access
- Allocate sufficient storage for tools and artifacts

#### Step 2: Install Required Tools on Infra Server

**Install AWS CLI:**
```bash
# Download and install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

**Install Terraform:**
```bash
# Download Terraform
wget https://releases.hashicorp.com/terraform/1.0.0/terraform_1.0.0_linux_amd64.zip
unzip terraform_1.0.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform --version
```

**Install kubectl:**
```bash
# Download kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

**Install EKSCTL:**
```bash
# Download and extract eksctl
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz

# Move to PATH
sudo mv eksctl /usr/local/bin/

# Verify installation
eksctl version
```

**Install Helm:**
```bash
# Option 1: Using script
sudo apt update && sudo apt upgrade -y
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Option 2: Manual installation
wget https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version
```

#### Step 3: Clone and Apply Terraform Configuration

```bash
# Clone the terraform project
git clone https://github.com/cybe44oot/Terraform-Mega-Project.git
cd Terraform-Mega-Project

# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Plan the deployment
terraform plan

# Apply the infrastructure
terraform apply

# Wait for completion (typically 15-20 minutes for EKS cluster)
```

### Phase 2: Kubernetes Cluster Configuration

#### Step 4: Configure kubectl with EKS Cluster

```bash
# Update kubeconfig to connect to EKS cluster
aws eks --region eu-north-1 update-kubeconfig --name devopsshack-cluster

# Verify cluster access
kubectl cluster-info
kubectl get nodes
```

#### Step 5: Configure OIDC Provider for Service Accounts

**Purpose:** Enable Kubernetes Service Accounts to assume IAM roles

```bash
# Associate IAM OIDC Provider with EKS cluster
eksctl utils associate-iam-oidc-provider \
  --region eu-north-1 \
  --cluster devopsshack-cluster \
  --approve
```

**Why this is needed:**
- Service Accounts in Kubernetes need to authenticate with AWS APIs
- This creates a trust relationship between the cluster and IAM
- Enables IRSA (IAM Roles for Service Accounts) feature

#### Step 6: Create EBS CSI Driver Service Account

**Purpose:** Enable dynamic persistent volume creation for Kubernetes

```bash
# Create IAM Service Account for EBS CSI Driver
eksctl create iamserviceaccount \
  --region eu-north-1 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster devopsshack-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts

# This command:
# - Creates a Kubernetes Service Account named 'ebs-csi-controller-sa'
# - Attaches AWS IAM policy for EBS operations
# - Enables the service account to create volumes
```

#### Step 7: Deploy EBS CSI Driver

```bash
# Install EBS CSI Driver pods in cluster
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"

# Verify deployment
kubectl get pods -n kube-system | grep ebs-csi

# Expected output: ebs-csi-controller and ebs-csi-node pods running
```

### Phase 3: Create RBAC and Storage Resources

#### Step 8: Set Up Jenkins Service Account and Permissions

**Purpose:** Enable Jenkins to authenticate with Kubernetes cluster and deploy applications

**Create files in rbac folder:**
```
Now create the role/rolebinding and cluster role/clusterrolebinding
and also the sa-secret u will find them in the rbac folder

now we need to get token of this sa witch we are going to use as authentication

save it in somewhere else
```

### Summary: Infrastructure Setup Checklist

- [ ] AWS account with appropriate IAM permissions
- [ ] Terraform configuration cloned and applied
- [ ] EKS cluster created and accessible
- [ ] kubectl configured with cluster credentials
- [ ] OIDC provider associated with cluster
- [ ] EBS CSI driver deployed
- [ ] Jenkins service account and RBAC created

---

# MEGA PROJECT CI - CONTINUOUS INTEGRATION

## Repository: `mega-project-ci`
**Purpose:** Implement automated CI pipeline for building, testing, and pushing container images

## URL: https://github.com/cybe44oot/mega-project-ci.git

### Architecture Overview

**CI Pipeline Flow:**
1. **Pre-Build Phase** → Git Checkout → Compile → Unit Tests → Quality Analysis
2. **Build Phase** → Docker Build → Docker Push to Registry
3. **Post-Build Phase** → Notifications → Webhook Triggers

### Phase 1: Jenkins Infrastructure Setup

#### Step 1: Create Jenkins Server Instance

**Instance Requirements:**
- Instance Type: t3.medium or larger
- OS: Ubuntu 20.04 LTS
- Storage: 30GB minimum
- Security Group: Allow ports 8080 (HTTP), 50000 (Agent communication)

#### Step 2: Install Java

```bash
# Update package lists
sudo apt update
sudo apt upgrade -y

# Install Java (Required for Jenkins)
sudo apt install openjdk-11-jdk -y

# Verify installation
java -version
```

#### Step 3: Install Jenkins

```bash
# Add Jenkins repository
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt update
sudo apt install jenkins -y

# Start Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Verify Jenkins is running
sudo systemctl status jenkins

# Access Jenkins at: http://<jenkins-ip>:8080
# Retrieve initial admin password:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

#### Step 4: Install Docker

```bash
# Install Docker prerequisites
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add Docker repository
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add Jenkins user to docker group
sudo usermod -aG docker jenkins

newgrp docker

# Verify Docker
docker --version
```

**Note:** Restart Jenkins after adding user to docker group:
```bash
sudo systemctl restart jenkins
```

#### Step 5: Install kubectl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify checksum
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

#### Step 6: Install Trivy (Security Scanning)

```bash
# Install dependencies
sudo apt-get install wget gnupg -y

# Add Trivy repository
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list

# Install Trivy
sudo apt-get update
sudo apt-get install trivy -y

# Verify
trivy version
```

### Phase 2: Jenkins Configuration

#### Step 7: Install Required Plugins

**Navigate to:** Jenkins Dashboard → Manage Jenkins → Manage Plugins → Available

**Install these plugins:**
- Docker Pipeline
- Kubernetes CLI
- SonarQube Scanner
- Maven Integration
- Pipeline Stage View
- Cobertura Plugin
- Email Extension Plugin
- Generic Webhook Trigger
- GitHub Integration

**Steps:**
1. Go to Dashboard → Manage Jenkins → Manage Plugins
2. Search for each plugin name
3. Check the checkbox
4. Click "Install without restart" or "Download now and install after restart"
5. Wait for installation to complete

#### Step 8: Configure SonarQube Credentials

**In Jenkins:**

**Step 8a: Add SonarQube Server Token**
1. Dashboard → Manage Jenkins → Manage Credentials
2. Click "System" → "Global credentials"
3. Click "Add Credentials"
4. Kind: Secret text
5. Secret: (Paste SonarQube token from SonarQube server)
6. ID: `sonarqube-token`
7. Save

**Step 8b: Configure SonarQube Server**
1. Dashboard → Manage Jenkins → Configure System
2. Find "SonarQube servers"
3. Click "Add SonarQube"
4. Name: `SonarQube`
5. Server URL: `http://<sonarqube-ip>:9000`
6. Server authentication token: Select the token created above
7. Save

#### Step 9: Configure Tools

**Navigate to:** Dashboard → Manage Jenkins → Configure System → Global Tool Configuration

**Configure Maven:**
1. Find "Maven" section
2. Click "Add Maven"
3. Name: `maven-3`
4. Version: `3.8.1` (or latest)
5. Click "Install automatically"
6. Save

**Configure SonarQube Scanner:**
1. Find "SonarQube Scanner" section
2. Click "Add SonarQube Scanner"
3. Name: `sonar-scanner`
4. Version: Select latest available
5. Click "Install automatically"
6. Save

#### Step 10: Configure Maven Settings with Nexus Credentials

**File Location:** Jenkins → Manage Jenkins → Managed Files

1. Click "Add a new Config"
2. Choose "Global Maven settings.xml"
3. ID: `global-maven-settings`
4. edit the following to the servers section to be:

```xml
<servers>
  <server>
    <id>maven-releases</id>
    <username>admin</username>
    <password>12345678</password>
  </server>
  <server>
    <id>maven-snapshots</id>
    <username>admin</username>
    <password>12345678</password>
  </server>
</servers>
```

5. Save

### Phase 3: Create CI Pipeline Job

#### Step 11: Create Pipeline Job in Jenkins

**Steps:**
1. Click "New Item" → Enter job name → Select "Pipeline" → OK
2. Under "Pipeline" section, select "Pipeline script from SCM"
3. SCM: Git
4. Repository URL: `https://github.com/cybe44oot/mega-project-ci.git`
5. Branch: `*/main`
6. Script Path: `Jenkinsfile`
7. Save

#### Step 12: Create Jenkinsfile

**File: Jenkinsfile**
```
you will find it in the Mega-Project-CI repo
```
### Phase 4: Configure Webhook Triggers

#### Step 13: Set Up Automated Trigger

**In Jenkins:**
1. Job Configuration → Build Triggers
2. Check "Generic Webhook Trigger"
3. Post content parameters:
   - Variable: `ref`
   - Expression: `$.ref` (JSONPath)
4. Token: `1234` (or any secret value)
5. Optional filter:
   - Expression: `refs/heads/main`
   - Text: `$ref`

**Webhook URL:** `http://<jenkins-ip>:8080/generic-webhook-trigger/invoke?token=1234`

**On GitHub:**
1. Repository Settings → Webhooks → Add webhook
2. Payload URL: (paste webhook URL from above)
3. Content type: application/json
4. Events: Push events
5. Save

### Phase 5: Nexus Integration

#### Step 14: Configure Nexus Repositories

**Update pom.xml with repository URLs:**

```xml
<distributionManagement>
    <repository>
        <id>maven-releases</id>
        <url>http://nexus-server:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>maven-snapshots</id>
        <url>http://nexus-server:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

### CI Pipeline Summary

**Stage Breakdown:**
- **Git Checkout:** Pulls source code from repository
- **Compile:** Builds the application code
- **Unit Tests:** Runs automated tests
- **SonarQube Analysis:** Performs code quality and security analysis
- **Build:** Creates JAR/WAR artifact
- **Docker Build:** Creates container image
- **Trivy Scan:** Scans container for vulnerabilities
- **Docker Push:** Pushes image to DockerHub/Registry

**Key Artifacts:**
- Container image pushed to registry
- SonarQube quality metrics stored
- Build logs for debugging

---

# MEGA PROJECT CD - CONTINUOUS DEPLOYMENT

## Repository: `mega-project-cd`
**Purpose:** Automated deployment to Kubernetes cluster with rolling updates and service management

## URL: https://github.com/cybe44oot/mega-project-cd.git

### Architecture Overview

**CD Pipeline Flow:**
1. Receive deployment trigger from CI
2. Pull Kubernetes manifests from repository
3. Apply manifests to EKS cluster
4. Configure Ingress and SSL
5. Expose services via LoadBalancer
6. Monitor deployment health

### Phase 1: Kubernetes Namespace and Deployment Preparation

#### Step 1: Create Application Namespace

```
all of the resources will be found in the repo manifest
```
### Phase 3: Ingress and SSL Configuration

#### Step 7: Install NGINX Ingress Controller

**Install NGINX Ingress on Infra Server:**

```bash
# Apply NGINX Ingress manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Get LoadBalancer IP
kubectl get svc -n ingress-nginx | grep ingress-nginx-controller
```

#### Step 8: Install Cert-Manager for SSL

**On Infra Server:**

```bash


# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Verify installation
kubectl get pods -n cert-manager

# Wait for all pods to be running
sleep 30
kubectl get pods -n cert-manager
```

#### Step 9: Create ClusterIssuer for SSL Certificates

**File: letsencrypt-issuer.yaml**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Apply:**
```bash
kubectl apply -f letsencrypt-issuer.yaml

# Verify issuer
kubectl get clusterissuer
```

#### Step 10: Create Ingress with SSL

**File: ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bootpage-ingress
  namespace: webapps
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - alhobab.site
    - www.alhobab.site
    secretName: bootpage-tls
  rules:
  - host: alhobab.site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bootpage-service
            port:
              number: 80
  - host: www.alhobab.site
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bootpage-service
            port:
              number: 80
```

**Apply Ingress:**
```bash
# First, update DNS records to point to NGINX LoadBalancer IP
# Get the NGINX LoadBalancer IP:
kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide

# Then apply ingress:
kubectl apply -f ingress.yaml

# Verify ingress
kubectl get ingress -n webapps
kubectl describe ingress bootpage-ingress -n webapps

# Check certificate creation (takes a few minutes)
kubectl get certificate -n webapps
kubectl describe certificate bootpage-tls -n webapps
```

### Phase 4: CI/CD Pipeline Integration

#### Step 11: Jenkins CD Pipeline

**File: Jenkinsfile-CD**

```groovy
pipeline {
    agent any

    environment {
        K8S_TOKEN = credentials('k8s-token')
        DOCKER_IMAGE = "dockerhub-username/app:${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/cybe44oot/mega-project-cd.git'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'devopsshack-cluster',
                    contextName: '',
                    credentialsId: 'k8s-token',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: 'https://A385AC9BFF829E9144AA7129151C1092.gr7.eu-north-1.eks.amazonaws.com'
                ) {
                    sh '''
                        # Update image tag in deployment
                        sed -i "s|image:.*|image: ${DOCKER_IMAGE}|g" Manifest/deployment.yaml
                        
                        # Apply manifests
                        kubectl apply -f Manifest/manifest.yaml
                        
                        # Apply HPA 
                        kubectl apply -f Manifest/hpa.yaml
                        
                        # Wait for rollout
                        sleep 30
                        
                        # Check deployment status
                        kubectl get pods -n webapps
                        kubectl get service -n webapps
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8s-token',
                    namespace: 'webapps',
                    serverUrl: 'https://A385AC9BFF829E9144AA7129151C1092.gr7.eu-north-1.eks.amazonaws.com'
                ) {
                    sh '''
                        # Wait for pods to be ready
                        kubectl rollout status deployment/bootpage-deployment -n webapps --timeout=5m
                        
                        # Check service endpoints
                        kubectl get endpoints bootpage-service -n webapps
                        
                        # Verify HPA is working
                        kubectl get hpa -n webapps
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to Kubernetes successful!"
        }
        failure {
            echo "Deployment failed. Rolling back..."
        }
    }
}
```

### CD Deployment Checklist

- [ ] EKS cluster with proper networking
- [ ] Namespaces created (webapps)
- [ ] ConfigMaps and Secrets configured
- [ ] Application deployment manifests created
- [ ] Database deployment and storage ready
- [ ] HPA configured for autoscaling
- [ ] NGINX Ingress Controller installed
- [ ] Cert-Manager installed
- [ ] SSL certificates configured
- [ ] Ingress rules created
- [ ] DNS records updated
- [ ] Jenkins CD pipeline configured
- [ ] Deployment tests passed

---

# MONITORING & OBSERVABILITY

## Purpose: Real-time monitoring of applications and infrastructure

### Architecture Overview

**Monitoring Stack:**
- **Prometheus:** Metrics collection and storage
- **Grafana:** Visualization and dashboards
- **Node Exporter:** Infrastructure metrics
- **Kube State Metrics:** Kubernetes cluster metrics

### Phase 1: Prometheus Setup via Helm

#### Step 1: Add Prometheus Helm Repository

```bash
# Add Prometheus Community Helm charts repo

1-setup Prometheus using helm :
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 

2-helm repo update

3-apply values pro/values.yaml

---
alertmanager:
  enabled: false
prometheus:
  prometheusSpec:
    service:
      type: LoadBalancer
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ebs-sc
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
grafana:
  enabled: true
  service:
    type: LoadBalancer
  adminUser: admin
  adminPassword: admin123
nodeExporter:
  service:
    type: LoadBalancer
kubeStateMetrics:
  enabled: true
  service:
    type: LoadBalancer
additionalScrapeConfigs:
  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100
  - job_name: kube-state-metrics
    static_configs:
      - targets:
          - kube-state-metrics:8080
```

#### Step 3: Install Prometheus Stack

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install the Prometheus stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f monitoring-values.yaml \
  -n monitoring

# Wait for deployment
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus -n monitoring --timeout=300s

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### Phase 2: Expose Services and Access

#### Step 4: Patch Services to LoadBalancer Type

```bash
kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc monitoring-kube-state-metrics -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc monitoring-prometheus-node-exporter -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'


k get svc -n monitoring

you should now find the Grafana and prometheus links

# Output will show:
# Prometheus:   <LoadBalancer-IP>:9090
# Grafana:      <LoadBalancer-IP>:3000
# Node Exporter: <LoadBalancer-IP>:9100
# Kube State Metrics: <LoadBalancer-IP>:8080
```

### Phase 3: Access and Configure Monitoring Tools

#### Step 5: Access Prometheus

**URL:** `http://<prometheus-loadbalancer-ip>:9090`

**Features to explore:**
- Query metrics using PromQL
- View targets and scrape configs
- Check alerts (if configured)
- View dashboards

**Sample PromQL Queries:**
```promql
# CPU usage by node
node_cpu_seconds_total

# Memory usage
node_memory_MemAvailable_bytes

# Pod count by namespace
count(kube_pod_info) by (namespace)

# Container restart count
rate(kube_pod_container_status_restarts_total[15m])
```

#### Step 6: Access Grafana

**URL:** `http://<grafana-loadbalancer-ip>:3000`

**Default Credentials:**
- Username: `admin`
- Password: `admin123`

**Initial Setup:**
1. Log in with default credentials
2. Change admin password
3. Add Prometheus data source:
   - Data Sources → Add Data Source
   - Type: Prometheus
   - URL: `http://monitoring-kube-prometheus-prometheus:9090`
   - Save & Test

