# 🚀 n8n Helm Chart for Kubernetes

<div align="center">

![n8n](https://raw.githubusercontent.com/n8n-io/n8n/master/assets/n8n-logo.png)

**Workflow Automation Tool - Self-Hosted Helm Chart**

[![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=Helm&labelColor=0F1689)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/kubernetes-326ce5.svg?&style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io)

</div>

---

## 📋 Table of Contents

- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Quick Start](#-quick-start)
- [Installation Methods](#-installation-methods)
  - [ClusterIP (Internal)](#1-clusterip-internal-access)
  - [NodePort (External)](#2-nodeport-external-access)
  - [Ingress (Production)](#3-ingress-production-ready)
- [Configuration](#-configuration)
- [Management](#-management)
- [Advanced Setup](#-advanced-setup)
- [Troubleshooting](#-troubleshooting)

---

## ✨ Features

- ✅ **Multiple Access Methods**: ClusterIP, NodePort, Ingress
- ✅ **Persistent Storage**: PVC support for data persistence
- ✅ **Health Checks**: Liveness and Readiness probes
- ✅ **Security**: Basic Auth, Security Context, RBAC
- ✅ **Scalable**: Resource limits and requests
- ✅ **Database Support**: SQLite (default) or PostgreSQL
- ✅ **Production Ready**: Ingress with TLS/SSL support

---

## 📦 Prerequisites

- Kubernetes cluster (v1.19+)
- Helm 3.x
- kubectl configured
- Storage class for PVC (optional)

```bash
# Verify installations
kubectl version --client
helm version
```

---

## 🚀 Quick Start

### Clone or Download Chart Files

```bash
# Create directory structure
mkdir -p n8n/templates
cd n8n

# Copy all chart files to respective directories
# Chart.yaml, values.yaml in root
# All templates in templates/ folder
```

### Verify Chart

```bash
# Lint the chart
helm lint .

# Dry run to check
helm install n8n . -n n8n --dry-run --debug
```

---

## 📥 Installation Methods

### 1. ClusterIP (Internal Access)

**Default installation for internal cluster access only**

#### Basic Installation

```bash
helm install n8n . -n n8n --create-namespace
```

#### Custom Installation

```bash
helm install n8n . -n n8n --create-namespace \
  --set service.type=ClusterIP \
  --set env[0].value="true" \
  --set env[1].value="admin" \
  --set env[2].value="YourSecurePassword123!"
```

#### Access via Port Forward

```bash
# Forward port to local machine
kubectl port-forward -n n8n svc/n8n 5678:5678

# Open browser
# http://localhost:5678
```

**Use Case**: Development, testing, internal workflows

---

### 2. NodePort (External Access)

**Access n8n from outside the cluster via Node IP**

#### Create Configuration File

```bash
cat <<EOF > values-nodeport.yaml
service:
  type: NodePort
  port: 5678
  nodePort: 30678

env:
  - name: N8N_BASIC_AUTH_ACTIVE
    value: "true"
  - name: N8N_BASIC_AUTH_USER
    value: "admin"
  - name: N8N_BASIC_AUTH_PASSWORD
    value: "YourSecurePassword123!"
  - name: N8N_HOST
    value: "0.0.0.0"
  - name: N8N_PORT
    value: "5678"
  - name: N8N_PROTOCOL
    value: "http"
  - name: WEBHOOK_URL
    value: "http://YOUR_NODE_IP:30678/"
  - name: GENERIC_TIMEZONE
    value: "Asia/Bangkok"
EOF
```

#### Install

```bash
helm install n8n . -n n8n --create-namespace -f values-nodeport.yaml
```

#### Get Node IP

```bash
# Get node IP address
kubectl get nodes -o wide

# Access n8n at: http://NODE_IP:30678
```

**Use Case**: Small deployments, home labs, testing external webhooks

---

### 3. Ingress (Production Ready)

**Production setup with domain name and SSL/TLS**

#### Install NGINX Ingress Controller

```bash
# Add Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

#### Install cert-manager (for SSL)

```bash
# Add cert-manager repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --create-namespace \
  --set installCRDs=true
```

#### Create ClusterIssuer for Let's Encrypt

```bash
cat <<EOF | kubectl apply -f -
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
EOF
```

#### Create n8n Configuration

```bash
cat <<EOF > values-ingress.yaml
service:
  type: ClusterIP

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: n8n.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: n8n-tls
      hosts:
        - n8n.yourdomain.com

env:
  - name: N8N_BASIC_AUTH_ACTIVE
    value: "true"
  - name: N8N_BASIC_AUTH_USER
    value: "admin"
  - name: N8N_BASIC_AUTH_PASSWORD
    value: "YourSecurePassword123!"
  - name: N8N_HOST
    value: "0.0.0.0"
  - name: N8N_PORT
    value: "5678"
  - name: N8N_PROTOCOL
    value: "https"
  - name: WEBHOOK_URL
    value: "https://n8n.yourdomain.com/"
  - name: GENERIC_TIMEZONE
    value: "Asia/Bangkok"

persistence:
  enabled: true
  size: 20Gi

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 512Mi
EOF
```

#### Install n8n

```bash
helm install n8n . -n n8n --create-namespace -f values-ingress.yaml

# Wait for certificate to be ready (2-5 minutes)
kubectl get certificate -n n8n -w
```

**Access**: https://n8n.yourdomain.com

**Use Case**: Production deployments, team collaboration, public webhooks

---

## ⚙️ Configuration

### Available Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | n8n image repository | `n8nio/n8n` |
| `image.tag` | n8n image tag | `latest` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `5678` |
| `service.nodePort` | NodePort (if type=NodePort) | `30678` |
| `persistence.enabled` | Enable persistence | `true` |
| `persistence.size` | PVC size | `10Gi` |
| `resources.limits.cpu` | CPU limit | `1000m` |
| `resources.limits.memory` | Memory limit | `1Gi` |

### Environment Variables

Common environment variables you can set:

```yaml
env:
  - name: N8N_BASIC_AUTH_ACTIVE
    value: "true"
  - name: N8N_BASIC_AUTH_USER
    value: "admin"
  - name: N8N_BASIC_AUTH_PASSWORD
    value: "your-password"
  - name: WEBHOOK_URL
    value: "http://your-domain.com/"
  - name: GENERIC_TIMEZONE
    value: "Asia/Bangkok"
  - name: N8N_METRICS
    value: "true"
```

---

## 🔧 Management

### Check Status

```bash
# View all resources
kubectl get all -n n8n

# View pods
kubectl get pods -n n8n

# View services
kubectl get svc -n n8n

# View PVC
kubectl get pvc -n n8n

# Helm status
helm status n8n -n n8n

# Helm values
helm get values n8n -n n8n
```

### View Logs

```bash
# Follow logs
kubectl logs -n n8n -l app.kubernetes.io/name=n8n -f

# View specific pod logs
kubectl logs -n n8n <pod-name>

# Last 100 lines
kubectl logs -n n8n -l app.kubernetes.io/name=n8n --tail=100
```

### Upgrade

```bash
# Upgrade with new values file
helm upgrade n8n . -n n8n -f values-new.yaml

# Upgrade specific value
helm upgrade n8n . -n n8n --set service.type=NodePort

# Upgrade to new chart version
helm upgrade n8n . -n n8n --reuse-values
```

### Rollback

```bash
# View history
helm history n8n -n n8n

# Rollback to previous version
helm rollback n8n -n n8n

# Rollback to specific revision
helm rollback n8n 2 -n n8n
```

### Uninstall

```bash
# Uninstall release
helm uninstall n8n -n n8n

# Delete namespace (removes all resources)
kubectl delete namespace n8n
```

---

## 🔐 Advanced Setup

### Use PostgreSQL Database

#### Install PostgreSQL

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres bitnami/postgresql \
  -n n8n \
  --set auth.username=n8n \
  --set auth.password=n8npassword \
  --set auth.database=n8n
```

#### Create Secret

```bash
kubectl create secret generic n8n-secrets \
  --from-literal=db-password=n8npassword \
  -n n8n
```

#### Update values.yaml

```yaml
env:
  - name: DB_TYPE
    value: "postgresdb"
  - name: DB_POSTGRESDB_HOST
    value: "postgres-postgresql"
  - name: DB_POSTGRESDB_PORT
    value: "5432"
  - name: DB_POSTGRESDB_DATABASE
    value: "n8n"
  - name: DB_POSTGRESDB_USER
    value: "n8n"
  - name: DB_POSTGRESDB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: n8n-secrets
        key: db-password
```

### Enable Metrics

```yaml
env:
  - name: N8N_METRICS
    value: "true"
```

### Custom Storage Class

```yaml
persistence:
  enabled: true
  storageClass: "fast-ssd"
  size: 50Gi
```

### High Availability (Multiple Replicas)

```yaml
replicaCount: 3

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - n8n
        topologyKey: kubernetes.io/hostname
```

---

## 🧪 Testing

### Health Check

```bash
# Test with curl pod
kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- \
  curl http://n8n.n8n.svc.cluster.local:5678/healthz

# Direct test (if using NodePort)
curl http://NODE_IP:30678/healthz
```

### Access Shell

```bash
# Get shell in pod
kubectl exec -it -n n8n <pod-name> -- /bin/sh

# Check files
ls -la /home/node/.n8n
```

### Test Webhook

```bash
# Create test workflow with webhook
# Get webhook URL from n8n UI
# Test with curl
curl -X POST http://your-n8n-url/webhook-test/test \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

---

## 🐛 Troubleshooting

### Pod Not Starting

```bash
# Describe pod
kubectl describe pod -n n8n -l app.kubernetes.io/name=n8n

# Check events
kubectl get events -n n8n --sort-by='.lastTimestamp'

# Check logs
kubectl logs -n n8n -l app.kubernetes.io/name=n8n --previous
```

### PVC Issues

```bash
# Check PVC status
kubectl get pvc -n n8n
kubectl describe pvc -n n8n

# Check storage class
kubectl get storageclass
```

### Service Not Accessible

```bash
# Check service
kubectl get svc -n n8n
kubectl describe svc n8n -n n8n

# Check endpoints
kubectl get endpoints -n n8n

# Test from another pod
kubectl run test --image=busybox -i --rm --restart=Never -- \
  wget -O- http://n8n.n8n.svc.cluster.local:5678/healthz
```

### Ingress Not Working

```bash
# Check ingress
kubectl get ingress -n n8n
kubectl describe ingress -n n8n

# Check ingress controller
kubectl get pods -n ingress-nginx

# Check certificate
kubectl get certificate -n n8n
kubectl describe certificate n8n-tls -n n8n
```

### Common Issues

| Issue | Solution |
|-------|----------|
| Pod CrashLoopBackOff | Check logs: `kubectl logs -n n8n <pod>` |
| Out of Memory | Increase `resources.limits.memory` |
| Permission Denied | Check `securityContext` and PVC permissions |
| Can't access NodePort | Check firewall rules and security groups |
| SSL Certificate Pending | Wait 2-5 minutes, check cert-manager logs |

---

## 📊 Monitoring

### Basic Monitoring

```bash
# Watch pod status
kubectl get pods -n n8n -w

# Monitor resources
kubectl top pods -n n8n
kubectl top nodes
```

### Prometheus Integration

Add to `values.yaml`:

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5678"
  prometheus.io/path: "/metrics"

env:
  - name: N8N_METRICS
    value: "true"
```

---

## 🔄 Backup & Restore

### Backup

```bash
# Backup PVC data
kubectl cp n8n/<pod-name>:/home/node/.n8n ./n8n-backup

# Backup with velero (if installed)
velero backup create n8n-backup --include-namespaces n8n
```

### Restore

```bash
# Restore from backup
kubectl cp ./n8n-backup n8n/<pod-name>:/home/node/.n8n

# Restart pod
kubectl rollout restart deployment/n8n -n n8n
```

---

## 📚 Additional Resources

- [n8n Documentation](https://docs.n8n.io/)
- [n8n GitHub](https://github.com/n8n-io/n8n)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## 📝 License

This Helm Chart is provided as-is. n8n is licensed under [Fair-code license](https://docs.n8n.io/reference/license/).

---

## 💡 Tips

1. **Always change default passwords** in production
2. **Use HTTPS** with Ingress for production
3. **Enable persistence** to keep your workflows
4. **Monitor resources** and adjust limits accordingly
5. **Backup regularly** - your workflows are important
6. **Use PostgreSQL** for production workloads
7. **Test webhooks** from external networks
8. **Update regularly** to get latest features and security patches

---

<div align="center">

**Made with ❤️ for the Kubernetes community**

⭐ Star this repo if you find it helpful!
