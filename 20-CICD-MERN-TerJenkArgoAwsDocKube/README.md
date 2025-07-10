# Advanced End-to-End DevSecOps Kubernetes Three-Tier Project

A comprehensive DevSecOps project demonstrating a complete CI/CD pipeline using AWS EKS, Jenkins, ArgoCD, Prometheus, Grafana, and Terraform. This project implements a three-tier architecture (Frontend, Backend, Database) with security scanning, monitoring, and GitOps practices.

**Learn from AmanPathak-DevOps channel**

Reference: <https://blog.stackademic.com/advanced-end-to-end-devsecops-kubernetes-three-tier-project-using-aws-eks-argocd-prometheus-fbbfdb956d1a>

## Project Architecture

- **Frontend**: React.js application
- **Backend**: Node.js API server  
- **Database**: MongoDB
- **Container Orchestration**: AWS EKS (Kubernetes)
- **CI/CD**: Jenkins with automated pipelines
- **GitOps**: ArgoCD for deployment automation
- **Monitoring**: Prometheus & Grafana
- **Security**: SonarQube, Trivy for vulnerability scanning
- **Infrastructure**: Terraform for IaC

## Prerequisites

- AWS Account with appropriate permissions
- Basic knowledge of Kubernetes, Docker, and CI/CD
- Git and GitHub account

## Step-by-Step Implementation

### 1. AWS Infrastructure Setup

#### 1.1 Create IAM User
```bash
# Create IAM user with AdministratorAccess policy for this demo
# Save Access Key ID and Secret Access Key
```

#### 1.2 EC2 Instance for Jenkins Server
1. Launch EC2 instance (t2.xlarge recommended)
2. Create IAM role with EC2 admin permissions
3. Configure Security Group with ports: 8080 (Jenkins), 9000 (SonarQube), 9090 (Prometheus)
4. Use Session Manager to connect to EC2

#### 1.3 Install Required Tools on Jenkins Server
```bash
# Connect to EC2 and switch to ubuntu user
sudo su ubuntu

# Run the installation script
bash tools-install.sh
```

**Tools installed by the script:**
- Java 17 (for Jenkins)
- Jenkins
- Docker & Docker Compose
- SonarQube (containerized)
- AWS CLI
- kubectl
- eksctl
- Terraform
- Trivy (security scanner)
- Helm

### 2. Jenkins Configuration

#### 2.1 Access Jenkins
```bash
# Get Jenkins initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Access Jenkins at: http://YOUR_EC2_PUBLIC_IP:8080
```

#### 2.2 Install Required Plugins
- AWS Credentials
- Pipeline: AWS Steps
- Terraform
- Stage View
- SonarQube Scanner
- Docker Pipeline

#### 2.3 Configure Credentials
1. **AWS Credentials**: Add AWS Access Key and Secret Key
2. **SonarQube Token**: Generate token from SonarQube and add to Jenkins

#### 2.4 Configure Tools
```bash
# Find Terraform installation path
whereis terraform

# Add in Jenkins > Manage Jenkins > Global Tool Configuration
# Terraform: /usr/bin/terraform
```

### 3. EKS Cluster Deployment

#### 3.1 Create EKS Cluster using Jenkins Pipeline
1. Create new Jenkins Pipeline job
2. Use the Jenkinsfile from `EKS-Terraform-GitHub-Actions/Jenkinsfile`
3. Build with parameters:
   - Environment: `dev`
   - Terraform_Action: `apply`

#### 3.2 Manual EKS Creation (Alternative)
```bash
# Create EKS cluster using eksctl
eksctl create cluster --name devsecops-eks \
  --region us-east-1 \
  --nodegroup-name linux-nodes \
  --node-type t3.medium \
  --nodes 3

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name devsecops-eks
```

### 4. Application Deployment

#### 4.1 Deploy Database (MongoDB)
```bash
kubectl apply -f Kubernetes-Manifests-file/Database/
```

#### 4.2 Deploy Backend
```bash
kubectl apply -f Kubernetes-Manifests-file/Backend/
```

#### 4.3 Deploy Frontend
```bash
kubectl apply -f Kubernetes-Manifests-file/Frontend/
```

#### 4.4 Configure Ingress
```bash
kubectl apply -f Kubernetes-Manifests-file/ingress.yaml
```

### 5. ArgoCD Setup for GitOps

#### 5.1 Install ArgoCD
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get ArgoCD admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

#### 5.2 Configure ArgoCD Application
- Access ArgoCD UI at https://localhost:8080
- Create applications for Frontend, Backend, and Database
- Connect to your Git repository containing Kubernetes manifests

### 6. Monitoring Setup

#### 6.1 Install Prometheus
```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus using Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus -n monitoring
```

#### 6.2 Install Grafana
```bash
# Install Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana -n monitoring

# Get Grafana admin password
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

### 7. CI/CD Pipeline Configuration

#### 7.1 Create Application Pipeline
Create Jenkins pipeline with stages:
1. **Code Checkout**: Pull code from Git
2. **Security Scan**: SonarQube code analysis
3. **Build**: Docker image build
4. **Security Scan**: Trivy image scanning
5. **Push**: Push to container registry
6. **Deploy**: Update Kubernetes manifests
7. **Notify**: ArgoCD auto-sync

#### 7.2 Sample Pipeline Script
```groovy
pipeline {
    agent any
    
    stages {
        stage('Git Checkout') {
            steps {
                git 'https://github.com/your-repo/three-tier-app.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh 'sonar-scanner'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t your-app:${BUILD_NUMBER} .'
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh 'trivy image your-app:${BUILD_NUMBER}'
            }
        }
        
        stage('Push to Registry') {
            steps {
                sh 'docker push your-app:${BUILD_NUMBER}'
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                sh 'kubectl set image deployment/your-app your-app=your-app:${BUILD_NUMBER}'
            }
        }
    }
}
```

### 8. Access Applications

#### 8.1 Get Service URLs
```bash
# Get LoadBalancer URLs
kubectl get svc -A

# Access applications through LoadBalancer endpoints
```

### 9. Security Best Practices

1. **Secret Management**: Use Kubernetes secrets and AWS Secrets Manager
2. **RBAC**: Implement proper Role-Based Access Control
3. **Network Policies**: Restrict pod-to-pod communication
4. **Image Scanning**: Regular vulnerability scans with Trivy
5. **Code Quality**: Continuous monitoring with SonarQube

### 10. Monitoring and Alerting

1. **Prometheus Metrics**: Monitor cluster and application metrics
2. **Grafana Dashboards**: Visualize system performance
3. **Alerting**: Configure alerts for critical issues
4. **Log Aggregation**: Centralized logging with ELK stack (optional)

## Cleanup

### Destroy EKS Cluster
```bash
# Using Terraform
terraform destroy -var-file=dev.tfvars -auto-approve

# Using eksctl
eksctl delete cluster --name devsecops-eks --region us-east-1
```

### Terminate EC2 Instance
```bash
# Terminate Jenkins server instance from AWS Console
```

## Project Structure

```
20-CICD-MERN-TerJenkArgoAwsDocKube/
├── EKS-Terraform-GitHub-Actions/     # Terraform code for EKS
├── Kubernetes-Jenkens-DevSecOps-Project/
│   ├── Application-Code/             # Frontend & Backend code
│   ├── Jenkins-Pipeline-Code/        # Pipeline scripts
│   ├── Jenkins-Server-TF/           # Terraform for Jenkins server
│   └── Kubernetes-Manifests-file/   # K8s deployment files
└── README.md
```

## Troubleshooting

### Common Issues
1. **EKS Access**: Ensure IAM user has proper EKS permissions
2. **kubectl**: Update kubeconfig after cluster creation
3. **Jenkins Plugins**: Install all required plugins before running pipelines
4. **Security Groups**: Open necessary ports for services

### Useful Commands
```bash
# Check EKS cluster status
eksctl get cluster

# View pod logs
kubectl logs -f pod-name

# Check service endpoints
kubectl get endpoints

# Describe problematic resources
kubectl describe pod/service/deployment resource-name
```

This project demonstrates a complete DevSecOps workflow with automated security scanning, continuous deployment, and comprehensive monitoring suitable for production environments.
