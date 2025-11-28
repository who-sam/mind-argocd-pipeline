# Notes App - Kubernetes Deployment Guide

## Repository Structure

```
argocd-pipeline/
├── namespace.yaml
├── postgres-secrets.yaml
├── postgres-pvc.yaml
├── postgres-deployment.yaml
├── postgres-service.yaml
├── backend-deployment.yaml
├── backend-service.yaml
├── frontend-deployment.yaml
├── frontend-service.yaml
├── ingress.yaml
├── backend-hpa.yaml
├── frontend-hpa.yaml
├── kustomization.yaml
├── argocd-application.yaml
└── README-K8S-DEPLOYMENT.md
```

## Prerequisites

1. **EKS Cluster** with the following:
   - Kubernetes version 1.27+
   - AWS Load Balancer Controller installed
   - EBS CSI Driver installed
   - Metrics Server installed (for HPA)

2. **ArgoCD** installed in the cluster

3. **Jenkins** with:
   - Docker installed
   - Git installed
   - Credentials configured:
     - `dockerhub-credentials` (username/password)
     - `github-token` (secret text)

## Setup Instructions

### 1. Install Required EKS Add-ons

#### Install AWS Load Balancer Controller
```bash
# Add the EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### Install EBS CSI Driver
```bash
# Install EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.25"

# Or use eksctl
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <your-cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

eksctl create addon --name aws-ebs-csi-driver --cluster <your-cluster-name> \
  --service-account-role-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```

#### Install Metrics Server (for HPA)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 2. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD Server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get ArgoCD initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Get ArgoCD URL
kubectl get svc argocd-server -n argocd
```

### 3. Configure Secrets (IMPORTANT)

**Before deploying, update the secrets in `postgres-secrets.yaml`:**

```bash
# Generate a strong JWT secret
openssl rand -base64 32

# Edit postgres-secrets.yaml and update:
# - POSTGRES_PASSWORD
# - DB_PASSWORD
# - JWT_SECRET
```

### 4. Deploy Application with ArgoCD

#### Option A: Using ArgoCD UI
1. Login to ArgoCD UI
2. Click "New App"
3. Fill in the details:
   - Application Name: `notes-app`
   - Project: `default`
   - Sync Policy: `Automatic`
   - Repository URL: `https://github.com/who-sam/argocd-pipeline.git`
   - Path: `.`
   - Cluster: `https://kubernetes.default.svc`
   - Namespace: `notes-app`
4. Enable "Auto-Create Namespace"
5. Click "Create"

#### Option B: Using kubectl
```bash
kubectl apply -f argocd-application.yaml
```

### 5. Verify Deployment

```bash
# Check namespace
kubectl get namespace notes-app

# Check all resources
kubectl get all -n notes-app

# Check pods status
kubectl get pods -n notes-app

# Check ingress
kubectl get ingress -n notes-app

# Get application URL
kubectl get ingress notes-app-ingress -n notes-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 6. Monitor ArgoCD Sync

```bash
# Watch ArgoCD application status
kubectl get application notes-app -n argocd -w

# Get sync status
argocd app get notes-app

# Manually sync if needed
argocd app sync notes-app
```

## Jenkins CI/CD Integration

### Jenkins Pipeline Flow

1. **Checkout** - Clones MIND and argocd-pipeline repos
2. **Build** - Builds Docker images for frontend and backend
3. **Security Scan** - Scans images for vulnerabilities (optional)
4. **Push** - Pushes images to Docker Hub
5. **Update Manifests** - Updates deployment YAMLs with new image tags
6. **Commit & Push** - Commits changes to argocd-pipeline repo
7. **ArgoCD Sync** - ArgoCD automatically detects changes and deploys

### Configure Jenkins Credentials

1. **DockerHub Credentials** (`dockerhub-credentials`):
   - Type: Username with password
   - Username: your-dockerhub-username
   - Password: your-dockerhub-token

2. **GitHub Token** (`github-token`):
   - Type: Secret text
   - Secret: your-github-personal-access-token
   - Permissions needed: `repo` (full control)

## Architecture Overview

```
                                    ┌─────────────────┐
                                    │   AWS ALB       │
                                    │   (Ingress)     │
                                    └────────┬────────┘
                                             │
                        ┌────────────────────┼────────────────────┐
                        │                    │                    │
                  ┌─────▼──────┐      ┌─────▼──────┐      ┌──────▼─────┐
                  │  Frontend  │      │  Backend   │      │  Backend   │
                  │   Pod 1    │      │   Pod 1    │      │   Pod 2    │
                  └────────────┘      └─────┬──────┘      └──────┬─────┘
                                             │                    │
                                             └──────────┬─────────┘
                                                        │
                                                 ┌──────▼───────┐
                                                 │  PostgreSQL  │
                                                 │     Pod      │
                                                 └──────────────┘
                                                        │
                                                 ┌──────▼───────┐
                                                 │  EBS Volume  │
                                                 │   (10GB)     │
                                                 └──────────────┘
```

## Resource Specifications

### PostgreSQL
- **Replicas**: 1 (Stateful)
- **Storage**: 10Gi EBS gp3
- **Memory**: 256Mi (request) / 512Mi (limit)
- **CPU**: 250m (request) / 500m (limit)

### Backend
- **Replicas**: 2-5 (Auto-scaling)
- **Memory**: 128Mi (request) / 256Mi (limit)
- **CPU**: 100m (request) / 200m (limit)
- **HPA**: CPU 70%, Memory 80%

### Frontend
- **Replicas**: 2-5 (Auto-scaling)
- **Memory**: 64Mi (request) / 128Mi (limit)
- **CPU**: 50m (request) / 100m (limit)
- **HPA**: CPU 70%, Memory 80%

## Troubleshooting

### Pods not starting
```bash
# Check pod status
kubectl describe pod <pod-name> -n notes-app

# Check logs
kubectl logs <pod-name> -n notes-app

# Check events
kubectl get events -n notes-app --sort-by='.lastTimestamp'
```

### Database connection issues
```bash
# Test database connectivity
kubectl run -it --rm debug --image=postgres:15-alpine --restart=Never -n notes-app -- psql -h postgres-service -U postgres -d notes_app

# Check database pod logs
kubectl logs -f deployment/postgres -n notes-app
```

### Ingress not working
```bash
# Check ingress status
kubectl describe ingress notes-app-ingress -n notes-app

# Check AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify security groups and target groups in AWS Console
```

### ArgoCD sync issues
```bash
# Check application status
argocd app get notes-app

# Force sync
argocd app sync notes-app --force

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller
```

### Image pull errors
```bash
# Verify image exists in Docker Hub
docker pull whosam1/notes-app-frontend:latest
docker pull whosam1/notes-app-backend:latest

# Check image pull secrets (if private)
kubectl get secrets -n notes-app
```

## Scaling

### Manual Scaling
```bash
# Scale frontend
kubectl scale deployment frontend -n notes-app --replicas=3

# Scale backend
kubectl scale deployment backend -n notes-app --replicas=3
```

### Check HPA Status
```bash
# View HPA status
kubectl get hpa -n notes-app

# Describe HPA
kubectl describe hpa backend-hpa -n notes-app
kubectl describe hpa frontend-hpa -n notes-app
```

## Monitoring

### View Resource Usage
```bash
# Pod resource usage
kubectl top pods -n notes-app

# Node resource usage
kubectl top nodes
```

### Application Logs
```bash
# Frontend logs
kubectl logs -f deployment/frontend -n notes-app

# Backend logs
kubectl logs -f deployment/backend -n notes-app

# PostgreSQL logs
kubectl logs -f deployment/postgres -n notes-app
```

## Backup and Restore

### Backup PostgreSQL Data
```bash
# Create backup
kubectl exec -it deployment/postgres -n notes-app -- pg_dump -U postgres notes_app > backup.sql

# Or use PVC snapshot (recommended)
kubectl get pvc postgres-pvc -n notes-app
```

### Restore PostgreSQL Data
```bash
# Restore from backup
kubectl exec -i deployment/postgres -n notes-app -- psql -U postgres notes_app < backup.sql
```

## Clean Up

```bash
# Delete application via ArgoCD
argocd app delete notes-app

# Or delete manually
kubectl delete namespace notes-app

# Clean up ArgoCD application definition
kubectl delete -f argocd-application.yaml
```

## Security Best Practices

1. **Update Secrets**: Change default passwords in `postgres-secrets.yaml`
2. **Enable HTTPS**: Add SSL certificate ARN to ingress annotations
3. **Network Policies**: Implement network policies to restrict pod communication
4. **RBAC**: Apply least privilege RBAC policies
5. **Image Scanning**: Enable Trivy or similar scanning in CI pipeline
6. **Secrets Management**: Consider using AWS Secrets Manager or External Secrets Operator
7. **Pod Security Standards**: Apply pod security policies/standards

## Performance Tuning

1. **Database Connection Pooling**: Configure in backend application
2. **Resource Limits**: Adjust based on actual usage patterns
3. **HPA Thresholds**: Fine-tune based on load testing
4. **Ingress Caching**: Add caching headers for static assets
5. **Database Optimization**: Add indexes, tune PostgreSQL settings

## Cost Optimization

1. **Right-size Resources**: Monitor and adjust requests/limits
2. **Use Spot Instances**: For non-critical workloads
3. **Enable Cluster Autoscaler**: Scale nodes based on demand
4. **EBS Volume Optimization**: Use gp3 instead of gp2
5. **Review ALB Usage**: Consider using NLB if appropriate

## Support

For issues, contact the DevOps team or create an issue in the repository.
