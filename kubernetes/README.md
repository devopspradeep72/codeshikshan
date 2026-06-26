# Kubernetes (K8s) - Container Orchestration

## What is Kubernetes?
Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications across clusters of machines.

---

## Real-Time Scenario 1: E-Commerce Platform Auto-Scaling

### Scenario Description
An e-commerce company experiences traffic spikes during sales events (Black Friday, Prime Day, etc.). They need their web application to automatically scale up during peak hours and scale down during off-peak hours to save costs.

### Problem
- Manual scaling is slow and error-prone
- Unexpected traffic can cause server crashes
- Over-provisioning wastes resources and money
- Need 99.9% uptime SLA

### Kubernetes Solution
```yaml
# Horizontal Pod Autoscaler (HPA)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-app
  minReplicas: 3          # Minimum pods during low traffic
  maxReplicas: 50         # Maximum pods during peak traffic
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### How It Works
1. **Monitoring**: Kubernetes monitors CPU and memory usage
2. **Decision**: When CPU usage exceeds 70%, it scales up pods
3. **Action**: New pods are created automatically
4. **Scale Down**: When usage drops, pods are gracefully terminated

### Real-World Impact
- **Before**: 10 servers always running = ₹50,000/month
- **After**: Average 5-8 servers = ₹25,000/month (50% cost savings)
- **Uptime**: 99.99% maintained during traffic spikes

---

## Real-Time Scenario 2: Rolling Update with Zero Downtime

### Scenario Description
A video streaming service needs to deploy a critical bug fix to their mobile app backend without interrupting active user streams.

### Problem
- Traditional deployment: Stop all servers → Deploy → Start (causes downtime)
- Users lose their streaming sessions
- Customer complaints and lost revenue

### Kubernetes Solution
```yaml
# Deployment with Rolling Update Strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: streaming-backend
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update
      maxUnavailable: 0   # Keep all pods available
  selector:
    matchLabels:
      app: streaming
  template:
    metadata:
      labels:
        app: streaming
    spec:
      containers:
      - name: backend
        image: streaming:v2.0   # New version
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Update Process
1. **Pod 1 Updated**: Old pod replaced with new version (5 active pods total)
2. **Health Check**: New pod runs readiness check before receiving traffic
3. **Gradual Rollout**: Pod 2, Pod 3, Pod 4 updated one by one
4. **Result**: Zero downtime, active streams continue

### Timeline
- Old Version Pods: 4 → 3 → 2 → 1 → 0
- New Version Pods: 0 → 1 → 2 → 3 → 4
- At any point: At least 4 pods serving traffic

---

## Real-Time Scenario 3: Self-Healing in Production

### Scenario Description
A banking microservice crashes unexpectedly due to a memory leak in one pod. Kubernetes automatically detects and restarts it without manual intervention.

### Problem
- One crashed pod = lost transactions
- Manual monitoring and restart takes 15-30 minutes
- Customers see errors

### Kubernetes Solution
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: banking
  template:
    metadata:
      labels:
        app: banking
    spec:
      containers:
      - name: bank-app
        image: banking:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"    # Kill pod if exceeds this
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

### Self-Healing Process
1. **Pod Crashes**: Memory leak causes OOM (Out of Memory)
2. **Detection**: Liveness probe fails 3 times in 30 seconds
3. **Action**: Kubernetes kills the crashed pod
4. **Recovery**: New pod is automatically started
5. **Time**: 10-15 seconds, fully automatic

### Impact
- **MTTR (Mean Time To Recovery)**: 30 minutes → 15 seconds
- **Manual Intervention**: Zero
- **Customer Impact**: Minimal (requests redirect to other pods)

---

## Key Kubernetes Concepts

### Pod
- Smallest deployable unit (can contain 1+ containers)
- Containers share network namespace (same IP)

### Service
- Exposes pods to network traffic
- Load balances across multiple pods
- Provides stable DNS name

### Deployment
- Declarative way to manage pods
- Enables rolling updates and rollbacks
- Maintains desired state

### StatefulSet
- For applications needing stable identity (databases)
- Maintains pod order and uniqueness

### ConfigMap & Secret
- ConfigMap: Store configuration data
- Secret: Store sensitive data (passwords, tokens)

---

## When to Use Kubernetes

✅ **Good Use Cases:**
- Microservices architecture
- Need for auto-scaling
- Multi-node deployments
- High availability requirements
- DevOps-mature teams

❌ **Avoid If:**
- Single simple application
- Small deployment
- Team inexperienced with containers
- Need minimal operational overhead

---

## Learning Path
1. Learn Docker first (containerization)
2. Deploy single pod locally
3. Create services and expose apps
4. Practice rolling updates
5. Set up monitoring and logging
6. Implement auto-scaling
