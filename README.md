# Production Kubernetes Configurations

[![Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes)](https://kubernetes.io/)
[![AWS EKS](https://img.shields.io/badge/Cloud-AWS%20EKS-FF9900?style=for-the-badge&logo=amazon-aws)](https://aws.amazon.com/eks/)
[![GCP GKE](https://img.shields.io/badge/Cloud-GCP%20GKE-4285F4?style=for-the-badge&logo=google-cloud)](https://cloud.google.com/kubernetes-engine)
[![99.9% Uptime](https://img.shields.io/badge/Target%20SLA-99.9%25%20Availability-brightgreen?style=for-the-badge)]()

Production-ready Kubernetes manifest patterns for operating enterprise workloads across **AWS EKS** and **GCP GKE** — reflecting the same configuration patterns used to sustain **99.9% cluster availability** at PT Link Net Tbk.

---

## 📁 Repository Structure

```
k8s-production-configs/
├── deployments/
│   └── api-gateway.yaml          # Full production Deployment with security hardening
├── hpa/
│   └── api-gateway-hpa.yaml      # HorizontalPodAutoscaler (CPU + Memory based)
├── policies/
│   └── pdb-and-netpol.yaml       # PodDisruptionBudget + NetworkPolicy + Namespaces
├── monitoring/
│   └── service-monitor.yaml      # Prometheus ServiceMonitor for metric scraping
└── README.md
```

---

## 🏗️ Production Architecture Principles

These configs demonstrate the **4 pillars of production Kubernetes reliability**:

### 1. Zero-Downtime Deployments
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0    # Never terminate a pod before new one is Ready
```
Combined with `readinessProbe` to ensure new pods are only added to the load balancer after passing health checks.

### 2. Auto-Scaling (HPA)
Horizontal Pod Autoscaler scales pods dynamically based on CPU and Memory:
```
Min: 2 pods  →  Scales up when CPU > 70%  →  Max: 10 pods
```
Scale-down includes a **5-minute stabilization window** to prevent pod flapping.

### 3. High Availability Policies
- **PodDisruptionBudget:** Guarantees minimum 2 pods always available during node drain/upgrades.
- **Pod Anti-Affinity:** Scheduler spreads pods across different nodes automatically.
- **NetworkPolicy:** Restricts ingress/egress to only required ports and namespaces (zero-trust network).

### 4. Security Hardening
Every container is configured with:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

---

## 🔍 Health Probe Strategy

Three-probe pattern for production reliability:

| Probe | Purpose | Action on Failure |
|---|---|---|
| `startupProbe` | Give slow-starting apps time to boot | Kill and restart |
| `livenessProbe` | Detect hung/deadlocked processes | Kill and restart |
| `readinessProbe` | Gate traffic until app is truly ready | Remove from Service endpoints |

---

## 📊 Observability Integration

The `ServiceMonitor` manifest (requires Prometheus Operator) enables automatic metrics scraping:
- Prometheus discovers the `api-gateway` pods via label selectors.
- Metrics are scraped every **15 seconds** from `/metrics` endpoint.
- Compatible with Grafana for dashboard visualization.

---

## 🚀 Applying to a Cluster

```bash
# Apply namespace first
kubectl apply -f policies/pdb-and-netpol.yaml

# Deploy the application
kubectl apply -f deployments/api-gateway.yaml

# Configure autoscaling
kubectl apply -f hpa/api-gateway-hpa.yaml

# Enable monitoring (requires Prometheus Operator)
kubectl apply -f monitoring/service-monitor.yaml

# Verify all pods are running
kubectl get pods -n production -w

# Check HPA status
kubectl get hpa -n production

# Check PDB status
kubectl get pdb -n production
```

---

## ☁️ Cloud-Specific Notes

### AWS EKS
- Use **Cluster Autoscaler** or **Karpenter** alongside HPA for node-level scaling.
- Mount IRSA (IAM Roles for Service Accounts) for S3/RDS/CloudWatch access.
- CloudWatch Container Insights for cluster-level monitoring.

### GCP GKE
- Enable **GKE Autopilot** for fully-managed node pools.
- Use **Workload Identity** instead of service account keys.
- Cloud Monitoring for GKE integrates natively with Prometheus format.
