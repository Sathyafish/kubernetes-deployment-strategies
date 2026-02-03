# Kubernetes Deployment Strategies

This repository contains sample deployment YAML files demonstrating various Kubernetes deployment strategies. Each strategy is designed to minimize downtime, enhance reliability, and improve the deployment process based on different use cases.

## 📁 Repository Structure

```
kubernetes-deployment-strategies/
├── 1-blue-green/
│   ├── blue-deployment.yaml
│   └── green-deployment.yaml
├── 2-canary/
│   └── canary-deployment.yaml
├── 3-ab-testing/
│   └── ab-testing-deployment.yaml
├── 4-ramped-slow-rollout/
│   └── ramped-deployment.yaml
├── 5-shadow-deployment/
│   └── shadow-deployment.yaml
├── 6-best-effort-controlled-rollout/
│   ├── argo-rollout.yaml
│   └── flagger-deployment.yaml
└── README.md
```

## 🚀 Deployment Strategies Overview

### 1. Blue/Green Deployment
**Deploys a new version (green) alongside the current (blue), then switches traffic once stable**

- **Downtime:** Zero (if switch passes readiness checks)
- **Rollback:** Instant (switch service selector back)
- **Use Cases:** Major version jumps, database schema changes, compliance requirements
- **Cost:** Doubles infrastructure during deployment

**Files:** `1-blue-green/`
- `blue-deployment.yaml` - Current stable version
- `green-deployment.yaml` - New version running alongside

**How to Switch Traffic:**
```bash
# Switch to green
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

### 2. Canary Deployment
**Sends a small percentage of traffic to the new version, gradually increasing after validation**

- **Downtime:** Very low
- **Rollback:** Very easy (scale down canary)
- **Use Cases:** Feature launches, config changes, error-budget-aware organizations
- **Requirements:** Metrics & alerting for automated promotion/abort

**Files:** `2-canary/`
- `canary-deployment.yaml` - Both stable (90%) and canary (10%) versions

**Progressive Rollout:**
```bash
# Start with 10% traffic (1 replica out of 10)
kubectl apply -f canary-deployment.yaml

# Monitor metrics, then increase canary
kubectl scale deployment myapp-canary --replicas=3  # 30%
kubectl scale deployment myapp-canary --replicas=5  # 50%

# Once validated, promote canary to stable
kubectl set image deployment/myapp-stable myapp=myapp:v2.0.0
kubectl scale deployment myapp-canary --replicas=0
```

---

### 3. A/B Testing
**Routes traffic to different versions based on specific user segments or rules**

- **Downtime:** Very low
- **Rollback:** Moderate (need to manage user cohorts)
- **Use Cases:** UX experiments, pricing tests, ML model comparison
- **Requirements:** Ingress/Service Mesh, analytics pipeline

**Files:** `3-ab-testing/`
- `ab-testing-deployment.yaml` - Version A and B deployments

**Implementation:**
- Requires Ingress controller (NGINX) or Service Mesh (Istio)
- Routes traffic based on HTTP headers, cookies, or user segments
- See inline comments for Ingress annotations

---

### 4. Ramped Slow Rollout
**Slowly increases the number of Pods with the new version over time**

- **Downtime:** Very low
- **Rollback:** Easy
- **Use Cases:** Mission-critical apps requiring metric validation between waves
- **Requirements:** Manual intervention or GitOps gates

**Files:** `4-ramped-slow-rollout/`
- `ramped-deployment.yaml` - Conservative RollingUpdate with manual pauses

**Manual Control:**
```bash
# Apply deployment
kubectl apply -f ramped-deployment.yaml

# Pause after first wave
kubectl rollout pause deployment/myapp-ramped

# Monitor metrics for 15 minutes, then resume
kubectl rollout resume deployment/myapp-ramped

# Repeat pause/resume cycle for each wave
```

---

### 5. Shadow Deployment
**Clones live traffic to a new version for testing without affecting users**

- **Downtime:** Zero (users never see shadow version)
- **Rollback:** None needed (nothing is live)
- **Use Cases:** Performance testing, regression testing, ML model validation
- **Requirements:** Service Mesh (Istio) for traffic mirroring

**Files:** `5-shadow-deployment/`
- `shadow-deployment.yaml` - Production and shadow versions

**Key Features:**
- Shadow version receives copy of production traffic
- Shadow responses are discarded (never returned to users)
- Requires Istio VirtualService for traffic mirroring (see inline example)

---

### 6. Best-Effort Controlled Rollout
**Attempts gradual rollout with intelligent backoff or analysis**

- **Downtime:** Very low
- **Rollback:** Automatic based on metrics
- **Use Cases:** Automated progressive delivery with metric-based decisions
- **Requirements:** Argo Rollouts or Flagger

**Files:** `6-best-effort-controlled-rollout/`
- `argo-rollout.yaml` - Argo Rollouts with automated canary analysis
- `flagger-deployment.yaml` - Flagger with Prometheus metrics

#### Option A: Argo Rollouts
```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Apply rollout
kubectl apply -f argo-rollout.yaml

# Monitor
kubectl argo rollouts get rollout myapp-controlled --watch

# Manual promotion (if needed)
kubectl argo rollouts promote myapp-controlled
```

#### Option B: Flagger
```bash
# Install Flagger (example for Linkerd)
kubectl apply -k github.com/fluxcd/flagger//kustomize/linkerd

# Apply canary
kubectl apply -f flagger-deployment.yaml

# Trigger rollout by updating image
kubectl set image deployment/myapp myapp=myapp:v2.0.0

# Monitor
kubectl describe canary myapp
```

---

## 🎯 Strategy Comparison

| Strategy | Traffic Shift | Downtime Risk | Rollback | Tools Required | Best For |
|----------|---------------|---------------|----------|----------------|----------|
| **Blue/Green** | All-at-once | Zero | Instant | Native K8s | Major releases |
| **Canary** | Progressive % | Very Low | Easy | Native K8s or Tools | Feature launches |
| **A/B Testing** | User segment | Very Low | Moderate | Ingress/Service Mesh | Experiments |
| **Ramped** | Throttled waves | Very Low | Easy | Native K8s | Mission-critical |
| **Shadow** | Mirrored (no user impact) | Zero | N/A | Service Mesh (Istio) | Testing |
| **Controlled** | Automated progressive | Very Low | Automatic | Argo/Flagger | Metric-driven |

---

## 🛠️ Prerequisites

### Basic Requirements
- Kubernetes cluster (v1.19+)
- kubectl CLI configured
- Basic understanding of Kubernetes concepts

### Additional Tools (Strategy-Specific)
- **A/B Testing:** NGINX Ingress Controller or Istio
- **Shadow Deployment:** Istio Service Mesh
- **Controlled Rollout:** 
  - Argo Rollouts + kubectl plugin
  - OR Flagger + Prometheus

---

## 📖 Usage

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd kubernetes-deployment-strategies
   ```

2. **Choose your strategy:**
   Navigate to the appropriate directory and review the YAML files.

3. **Customize the deployments:**
   - Update image names and versions
   - Adjust replica counts
   - Configure resource limits
   - Update service selectors and ports

4. **Apply to your cluster:**
   ```bash
   kubectl apply -f <strategy-folder>/<deployment-file>.yaml
   ```

5. **Monitor the deployment:**
   ```bash
   kubectl rollout status deployment/<deployment-name>
   kubectl get pods -w
   ```

---

## 🔧 Common Commands

### Deployment Management
```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Watch rollout status
kubectl rollout status deployment/myapp

# View rollout history
kubectl rollout history deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2
```

### Scaling
```bash
# Scale deployment
kubectl scale deployment myapp --replicas=5

# Autoscale
kubectl autoscale deployment myapp --min=3 --max=10 --cpu-percent=80
```

### Debugging
```bash
# Get pods
kubectl get pods -l app=myapp

# Describe deployment
kubectl describe deployment myapp

# View logs
kubectl logs -l app=myapp --tail=100 -f

# Get events
kubectl get events --sort-by='.lastTimestamp'
```

---

## ⚠️ Important Notes

1. **Health Checks:** Always configure liveness and readiness probes
2. **Resource Limits:** Set appropriate CPU and memory limits
3. **Monitoring:** Implement metrics collection (Prometheus recommended)
4. **Testing:** Test deployments in non-production environments first
5. **Rollback Plan:** Always have a rollback strategy ready
6. **Cost:** Be aware of infrastructure costs (especially Blue/Green and Shadow)

---

## 📚 Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [Flagger Documentation](https://docs.flagger.app/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)

---

## 🤝 Contributing

Feel free to submit issues or pull requests to improve these examples.

---

## 📄 License

These examples are provided as-is for educational purposes.
