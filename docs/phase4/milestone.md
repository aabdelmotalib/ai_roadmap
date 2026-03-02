# Phase 4 Milestone: Production Kubernetes Platform

**You can now operate AI systems at massive scale on Kubernetes.**

This milestone validates you can run, scale, monitor, and operate production-grade AI infrastructure.

---

## Exit Criteria Checklist (20+ Items)

### Kubernetes Fundamentals

- [ ] Understand Pod, Deployment, Service, ConfigMap, Secret concepts
- [ ] Know difference between Deployment (stateless) and StatefulSet (stateful)
- [ ] Have created Deployments, Services, ConfigMaps, Secrets from scratch
- [ ] Understand readinessProbe vs livenessProbe (and when you need each)
- [ ] Know kubectl get, describe, logs, exec, scale commands
- [ ] Understand Namespaces and created multiple namespaces
- [ ] Successfully deployed to Minikube and EKS

### Stateful Services

- [ ] Deployed Qdrant with StatefulSet (persistent storage)
- [ ] Deployed PostgreSQL with StatefulSet (persistent storage)
- [ ] Understand ordered scaling (StatefulSet-0 before StatefulSet-1)
- [ ] Created PersistentVolumeClaims and verified data persists after pod deletion

### Networking & Service Discovery

- [ ] Pods can reach each other by service hostname (api ↔ qdrant ↔ postgres)
- [ ] Created Ingress and routed external traffic
- [ ] Understand pod anti-affinity (spread across nodes)
- [ ] Know difference between ClusterIP, NodePort, LoadBalancer services

### Configuration Management

- [ ] Stored non-secret config in ConfigMaps
- [ ] Stored API keys in Secrets (not in image, not in ConfigMap)
- [ ] Know how to inject both into containers

### Autoscaling

- [ ] Created HPA (Horizontal Pod Autoscaler)
- [ ] Successfully scaled pods under load
- [ ] Understand min/max replicas and CPU target

### Observability

- [ ] Deployed Prometheus and collected metrics
- [ ] Created Grafana dashboard with API latency, error rate, pod count
- [ ] Set up Grafana alerts (latency > 10s, error rate > 1%)
- [ ] Read Prometheus queries (PromQL)

### Production Operations

- [ ] Deployed to AWS EKS
- [ ] Understand node groups (on-demand vs spot)
- [ ] Know how to debug pods (logs, describe, exec, rollout)
- [ ] Performed rolling update and rollback

---

## Capstone Project: Production K8s AI Platform

**What you'll build:** Complete AI stack on Minikube (dev) and EKS (production) with:
- API auto-scaling
- Stateful services (Qdrant, PostgreSQL)
- Monitoring and alerts
- Graceful degradation (health checks, timeouts)

### Architecture

```
Internet
  ↓
AWS Load Balancer
  ↓
Ingress (Nginx)
  ↓
API Service
  ↓
API Deployment (3-20 replicas, HPA-scaled on CPU)
  ↓
Qdrant StatefulSet (3 replicas, persistent storage)
PostgreSQL StatefulSet (1 replica, backup job)
Redis Deployment (caching)
  ↓
Prometheus (metrics)
Grafana (dashboards, alerts)
```

### Requirements

✅ **Minikube Setup:**
- [ ] All services running (API, Qdrant, PostgreSQL, Redis)
- [ ] Health checks passing
- [ ] API can reach backend services
- [ ] Persistent storage verified

✅ **Scaling:**
- [ ] HPA created with CPU target 70%
- [ ] Load test: generate 1000 req/sec
- [ ] API replicas scale from 3 to 15+
- [ ] Latency stays < 5s during scale

✅ **Monitoring:**
- [ ] Prometheus scrapes metrics from all services
- [ ] Grafana dashboard shows:
  - Request rate (requests/sec)
  - P99 latency (should be < 10s)
  - Error rate (should be < 0.5%)
  - Pod count (should match HPA state)
  - Token usage (should trend to estimate cost)
- [ ] Alerts working (email or Slack)

✅ **Production Deployment (EKS):**
- [ ] Cluster created with 3 on-demand + 5 spot nodes
- [ ] All manifests deployed to EKS
- [ ] Services accessible via Ingress DNS
- [ ] TLS certificate (self-signed for demo)
- [ ] All monitoring running

✅ **Resilience:**
- [ ] Delete a pod, watch it restart
- [ ] Delete a node, pods reschedule to other nodes
- [ ] Update image, rolling update completes with zero downtime
- [ ] Database survives pod restart (persistent volume)

### Deliverables

```
├── k8s/
│   ├── dev/                        (Minikube manifests)
│   │   ├── namespace.yaml
│   │   ├── configmaps.yaml
│   │   ├── secrets.sh
│   │   ├── deployments/
│   │   │   ├── api.yaml
│   │   │   ├── redis.yaml
│   │   │   └── ingress.yaml
│   │   ├── statefulsets/
│   │   │   ├── qdrant.yaml
│   │   │   └── postgres.yaml
│   │   ├── services.yaml
│   │   └── hpa.yaml
│   │
│   └── prod/                       (EKS manifests)
│       ├── namespace.yaml
│       ├── kustomization.yaml      (differentiate from dev)
│       ├── all YAML files (same structure as dev, different resources)
│       ├── monitoring/
│       │   ├── prometheus-values.yaml
│       │   └── grafana-values.yaml
│       └── ingress-prod.yaml
│
├── helm/                           (Optional: Helm chart for deployment)
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── templates/
│   │   ├── api.yaml
│   │   ├── qdrant.yaml
│   │   ├── postgres.yaml
│   │   └── ...
│
├── terraform/                      (Optional: AWS infrastructure as code)
│   ├── main.tf
│   ├── vpc.tf
│   ├── eks.tf
│   └── variables.tf
│
├── scripts/
│   ├── deploy-minikube.sh
│   ├── deploy-eks.sh
│   ├── load-test.sh               (Apache Bench or similar)
│   ├── cleanup.sh
│   └── verify-health.sh
│
├── Dockerfile
├── main.py                         (FastAPI with all endpoints)
├── requirements.txt
│
└── README.md                       (Complete documentation)
    ├── Architecture diagram
    ├── How to deploy (step-by-step)
    ├ How to run load test
    ├─ How to monitor
    └─ Troubleshooting guide
```

### Success Criteria (All Must Pass)

✅ **Minikube:**
- [ ] Deploy with one command: `./deploy-minikube.sh`
- [ ] All pods running and healthy: `kubectl get pods`
- [ ] Health check passes: `curl localhost:8000/health`
- [ ] Can scale to 10 replicas: `kubectl scale deployment api --replicas=10`
- [ ] Latency < 500ms under normal load

✅ **EKS:**
- [ ] Deploy with: `./deploy-eks.sh`
- [ ] Cluster auto-scales nodes (starts with 3, scales to max 8)
- [ ] Ingress accessible via public DNS
- [ ] TLS working (openssl s_client -connect domain:443)
- [ ] Graceful shutdown (drain, terminate, reschedule on other node)

✅ **Monitoring:**
- [ ] Prometheus scrapes metrics (http://prometheus:9090/targets)
- [ ] Grafana dashboard loads (http://grafana:3000)
- [ ] Alert fired when latency spikes (test by overloading)

✅ **Documentation:**
- [ ] README explains every component
- [ ] Scripts have clear comments
- [ ] Troubleshooting guide covers 10+ common issues
- [ ] Deployment diagram in Mermaid format

---

## Self-Assessment Rubric

### Beginner (Not Ready for Production)

**Indicators:**
- Can deploy Minikube, but not EKS
- Don't fully understand why StatefulSet for databases
- Monitoring setup but don't read the metrics
- Health checks not configured properly
- Scaling doesn't work or is manual

**Example:** "I deployed API on K8s but Qdrant lost data when the pod crashed because I didn't use persistent storage."

### Competent (Ready for Team)

**Indicators:**
- Deployed full stack to both Minikube and EKS
- Understand Deployment vs StatefulSet and apply correctly
- Health checks configured (readiness + liveness)
- Prometheus + Grafana working, reading metrics
- HPA auto-scales correctly
- Can debug pod issues (logs, describe, exec)
- Documented deployment process

**Example:** "I deployed API + Qdrant + PostgreSQL + monitoring to EKS. It auto-scales from 3 to 20 pods under load. If a pod crashes, K8s restarts it. I can see metrics in Grafana and respond to alerts."

### Hire-Ready (Expert Level)

**Indicators:**
- Everything above, plus:
- Optimizations: cost-aware (spot instances for batch), performance tuning
- Advanced: custom metrics (scaling on token usage, not just CPU)
- KEDA for event-driven scaling
- IRSA for no-credential pod access
- GitOps setup (ArgoCD) for infra-as-code
- Disaster recovery tested (backup/restore)
- Security hardened (RBAC, network policies, secrets encryption)

**Example:** "I run EKS cluster with 60% spot instances (save money), scale API on both CPU and token usage (prediction), monitor cost in real-time, have automated backups, and can deploy new version with single git commit (ArgoCD)."

---

## What a Failed Milestone Looks Like

❌ **You're not ready if:**

1. "Qdrant pods start but lose vectors when they restart" — no persistent volume
2. "Services can't find each other" — DNS or service creation wrong
3. "HPA creates pods but API still slow" — other bottleneck (network, DB connection pool)
4. "Deployed to EKS but no monitoring" — flying blind, can't respond to incidents
5. "Health checks never pass" — misunderstanding of readiness probe delay
6. "Can scale to 100 replicas but node runs out of memory" — no resource limits
7. "Rolling update causes 30% error spike" — pod shutdown not graceful
8. "Can't debug pod crashes" — don't know kubectl describe/logs workflow

---

## Common Reasons People Rush Phase 4

### "K8s is too complex, I'll stick with Docker Compose"
**Consequence:** As your system grows to 10+ services with traffic spikes, manual scaling/updates become exhausting. Team paged at 3am for human-triggered scale-up.

**Reality:** K8s has steep curve but payoff is huge. Invest the 3-4 weeks now.

### "We can skip monitoring, I'll just check CPU usage manually"
**Consequence:** System degrades slowly. You don't notice latency creep from 200ms to 5s. Customers complain. You lose trust.

**Reality:** Monitoring is non-negotiable for production. Set it up from day 1.

### "Minikube is good enough, no need for EKS"
**Consequence:** Deploys work locally, fail on EKS (storage classes, networking, security contexts different). Spend days debugging prod-only issues.

**Reality:** Deploy to both. Find issues in EKS early, not after customers hit them.

### "We'll optimize cost later, just spin up large nodes"
**Consequence:** Monthly bill: $3K. Should be $800 with spot instances + node consolidation.

**Reality:** Design for cost from the beginning. Spot instances, scale-to-zero for batch, autoscaling.

---

## Bridge to Phase 5

You've now built:
**Phases 1-3:** AI systems (LLMs, RAG, agents, evaluation, MLOps)
**Phase 4:** Infrastructure (Kubernetes, autoscaling, monitoring)

Next: **Phase 5 (Portfolio & Jobs)** teaches you to package all this into a comprehensive portfolio and land the job.

Your portfolio will showcase:
- Phase 1: AI system evaluation (RAGAS, metrics, analysis)
- Phase 2: Deployment on AWS (Docker, ECS, Lambda)
- Phase 3: Experiment tracking (MLflow, GitHub Actions)
- Phase 4: K8s infrastructure (EKS, monitoring, autoscaling)

→ **[Phase 5: Portfolio & Jobs →](../phase5/overview.md)**

---

## 1-Week Phase 4 Review Checklist

- [x] Day 1: K8s fundamentals (pods, deployments, services, minikube)
- [x] Day 2: Deploy FastAPI and expose service
- [x] Day 3: Deploy Qdrant and PostgreSQL with persistent storage
- [x] Day 4: Configure ingress and external routing
- [x] Day 5: Set up HPA and Prometheus metrics
- [x] Day 6: Create Grafana dashboard and alerts
- [x] Day 7: Deploy to EKS and verify in production

---

## Final Checkpoint

Before moving to Phase 5, verify:

- [ ] Can deploy any Docker image to K8s
- [ ] Can scale API from 3 to 100 replicas
- [ ] Can route traffic with Ingress
- [ ] Can debug any pod issue (logs, networking, storage)
- [ ] Understand HPA metrics and scaling decisions
- [ ] Can read Prometheus queries and Grafana dashboards
- [ ] EKS deployment works without local minikube
- [ ] Documentation is complete and someone else can follow it

**If all pass, you're ready for Phase 5.**

**If not, pick the problem area and spend extra week on it. K8s is hard; better to master it now.**

---

## 21. Resources

- **Kubernetes Official:** https://kubernetes.io/docs/
- **EKS Documentation:** https://docs.aws.amazon.com/eks/
- **Helm Charts:** https://artifacthub.io/
- **Prometheus Queries:** https://prometheus.io/docs/prometheus/latest/querying/basics/
- **Grafana Dashboards:** https://grafana.com/grafana/dashboards/
- **Real-world K8s:** https://www.cncf.io/ (Cloud Native Computing Foundation)

---

**Congratulations on Phase 4.** You're now a Kubernetes-capable AI engineer. You can operate at scale.

→ **[Phase 5: Portfolio & Jobs →](../phase5/overview.md)**
