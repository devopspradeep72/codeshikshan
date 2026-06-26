# Complete DevOps Learning Path

## Prerequisites
- Basic Linux command line knowledge
- Understanding of networking basics (ports, protocols)
- Basic scripting (bash)
- Version control (Git basics)

---

## Phase 1: Foundation (Week 1-2)

### Docker - Containerization
**Why First?** You need to understand containers before orchestration.

**Learning Goals:**
1. What is containerization and why it matters
2. Build Docker images from Dockerfile
3. Run containers and manage lifecycle
4. Use docker-compose for multi-container apps
5. Push/pull images from registry

**Practice:**
- Containerize 3 different applications (Node, Python, Java)
- Create docker-compose file for full stack
- Push to Docker Hub

**Scenario:** Package your web app with database and cache as containers

---

## Phase 2: Orchestration (Week 3-4)

### Kubernetes - Container Orchestration
**Why?** Manage containers at scale across multiple machines.

**Learning Goals:**
1. Kubernetes architecture (control plane, nodes)
2. Core concepts (pods, services, deployments)
3. ConfigMaps and Secrets
4. Persistent volumes
5. Ingress and networking
6. Auto-scaling (HPA, VPA)
7. Monitoring and logging

**Practice:**
- Deploy app with auto-scaling
- Perform rolling updates
- Set up monitoring
- Practice disaster recovery

**Scenario:** Run e-commerce platform with auto-scaling during peak hours

---

## Phase 3: Infrastructure (Week 5-6)

### Terraform - Infrastructure as Code
**Why?** Define infrastructure programmatically for consistency and repeatability.

**Learning Goals:**
1. Core concepts (resources, variables, outputs)
2. State management
3. Modules and reusability
4. Multiple environments (dev/staging/prod)
5. Remote state and locking
6. Cloud providers (AWS, Azure, GCP basics)

**Practice:**
- Create VPC with subnets
- Provision EC2 instances
- Set up RDS database
- Create S3 buckets
- Manage multiple environments

**Scenario:** Provision multi-environment infrastructure for SaaS application

---

## Phase 4: Configuration Management (Week 7-8)

### Ansible - Automation & Configuration
**Why?** Configure machines at scale without agents.

**Learning Goals:**
1. YAML syntax
2. Playbooks and plays
3. Inventory management
4. Modules and idempotency
5. Roles for reusability
6. Handlers and notifications
7. Advanced features (async, loops, conditionals)

**Practice:**
- Deploy applications to multiple servers
- Configuration compliance checking
- Incident response automation
- Rolling deployments

**Scenario:** Deploy app updates to 100 servers with zero downtime

---

## Phase 5: Integration (Week 9-10)

### CI/CD Pipeline
**Integrate all tools together:**

```
Git Push → Jenkins/GitLab CI
  ↓
Run Tests
  ↓
Build Docker Image
  ↓
Push to Registry
  ↓
Terraform Plan & Apply (if infra changes)
  ↓
Deploy to Kubernetes using Helm
  ↓
Run Smoke Tests
  ↓
Notify Slack
```

---

## Phase 6: Advanced Topics (Week 11-12)

### Topics to Explore
1. **Helm** - Package manager for Kubernetes
2. **GitOps** - Git as single source of truth
3. **Service Mesh** - Advanced networking (Istio, Linkerd)
4. **Monitoring** - Prometheus, Grafana, ELK
5. **Security** - Network policies, RBAC, secrets management
6. **Cost Optimization** - Right-sizing, reserved instances
7. **Disaster Recovery** - Backup, failover, multi-region

---

## Hands-On Projects

### Project 1: Complete Web Application Stack
**Duration:** 1 week

**Requirements:**
- Containerize web app (frontend + backend + database)
- Deploy to Kubernetes with auto-scaling
- Use Terraform to provision cloud infrastructure
- Use Ansible to configure servers
- Set up CI/CD pipeline

**Scenario:** E-commerce platform with 1M users

---

### Project 2: Multi-Environment Deployment
**Duration:** 1 week

**Requirements:**
- Dev environment (minimal resources)
- Staging environment (production-like)
- Production environment (highly available)
- Use Terraform workspaces
- Use Ansible for consistency
- Promote code through environments

**Scenario:** SaaS platform with dev/staging/prod

---

### Project 3: Disaster Recovery Automation
**Duration:** 2 weeks

**Requirements:**
- Provision infrastructure in 2 regions
- Automate failover process
- Regular DR drills
- Recovery time objectives (RTO) < 5 minutes
- Recovery point objectives (RPO) < 1 minute
- Document entire process

**Scenario:** Mission-critical application

---

## Real-World Example: Complete Application Deployment

### Architecture
```
┌─────────────────────────────────────────────────────────┐
│ Load Balancer (ALB)                                     │
├─────────────────────────────────────────────────────────┤
│ Kubernetes Cluster (Terraform Provisioned)              │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │ Frontend Pod     │  │ Backend Pod      │  ...       │
│  │ (React)          │  │ (Node.js)        │  x3        │
│  │ Auto-scaled      │  │ Auto-scaled      │            │
│  └──────────────────┘  └──────────────────┘            │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │ Database (RDS PostgreSQL)                │          │
│  │ Provisioned with Terraform               │          │
│  │ Multi-AZ for high availability           │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
       ↑
       │ Ansible Playbooks
       │ (Configuration & Deployment)
       │
Terraform
(Infrastructure)
```

### Deployment Workflow

**Step 1: Developer Commits Code**
```bash
git commit -m "Fix bug in payment processing"
git push origin main
```

**Step 2: CI Pipeline Triggered**
```
1. Clone code
2. Run unit tests
3. Build Docker image: myapp:commit-hash
4. Push to ECR (AWS Container Registry)
5. Run integration tests
6. Notify Slack: "Build successful"
```

**Step 3: Staging Deployment (Automated)**
```bash
# Ansible deploys new version to staging
ansible-playbook -i staging deploy.yml -e "image_tag=commit-hash"

# Kubernetes rolls out update
# Old pods → New pods (one by one)
```

**Step 4: Smoke Tests**
```bash
# Run tests against staging
- API responses correct
- Database queries work
- Payment processing works
- Performance acceptable
```

**Step 5: Manual Approval & Production Deployment**
```bash
# Team approves in GitHub
# PR merged to main

# Production deployment (Kubernetes rolling update)
kubectl set image deployment/myapp \
  myapp=myapp:commit-hash

# Verify via Ansible health checks
ansible-playbook -i prod health-check.yml
```

**Step 6: Post-Deployment**
```
- Monitor Prometheus metrics
- Check application logs (ELK)
- Verify user impact (New Relic)
- Notify team: "Deployment successful"
```

---

## Key Metrics to Track

### Deployment Metrics
- **Deployment Frequency**: How often you deploy (target: daily)
- **Lead Time**: Time from code commit to production (target: < 1 hour)
- **Change Failure Rate**: % of deployments causing incidents (target: < 15%)
- **MTTR**: Mean Time To Recovery from incidents (target: < 1 hour)

### Infrastructure Metrics
- **Uptime**: % of time system is available (target: 99.99%)
- **Cost per User**: Infrastructure cost per active user
- **Resource Utilization**: % of provisioned resources actually used

### Application Metrics
- **Response Time**: API latency (target: < 200ms)
- **Error Rate**: % of requests failing (target: < 0.1%)
- **Throughput**: Requests per second

---

## Troubleshooting Common Issues

### Docker Issues
- Image not found → Check registry, tag correctly
- Port already in use → Check docker ps, kill old container
- Out of disk → docker system prune

### Kubernetes Issues
- Pod stuck pending → Check resource requests vs available
- CrashLoopBackOff → Check logs: kubectl logs pod-name
- Service not accessible → Check service selector labels

### Terraform Issues
- State locked → Check DynamoDB, remove lock manually (careful!)
- Resource already exists → Check state file, import if needed
- Credentials not working → Check environment variables, AWS config

### Ansible Issues
- SSH connection refused → Check inventory, IP, firewall
- Privilege escalation fails → Check sudo password, become options
- Idempotency broken → Module changed when should be unchanged

---

## Resources for Continued Learning

### Official Documentation
- Docker: https://docs.docker.com
- Kubernetes: https://kubernetes.io/docs
- Terraform: https://www.terraform.io/docs
- Ansible: https://docs.ansible.com

### Practice Platforms
- Play with Docker: https://labs.play-with-docker.com
- Kubernetes Playground: https://www.killercoda.com/playgrounds/scenario/kubernetes
- Terraform Cloud: Free tier with remote state

### Certifications
- Docker Certified Associate (DCA)
- Kubernetes Application Developer (CKAD)
- Certified Kubernetes Administrator (CKA)
- HashiCorp Certified: Terraform Associate
- Red Hat Certified Specialist in Ansible Automation

---

## Next Steps

1. **Pick a Simple Project**: Start with containerizing one application
2. **Deploy Locally First**: Use docker-compose before Kubernetes
3. **Practice Regularly**: DevOps requires hands-on experience
4. **Learn by Doing**: Don't just watch videos, build things
5. **Review Failures**: Every failed deployment is a learning opportunity
6. **Stay Updated**: DevOps tools evolve constantly

Good luck on your DevOps journey! 🚀
