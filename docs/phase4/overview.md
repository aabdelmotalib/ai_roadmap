# Phase 4: Kubernetes & Advanced Ops

You've deployed AI systems to AWS. Now scale them to serve production traffic.

**Docker Compose** runs 5-10 services. **Kubernetes** runs thousands.

This phase teaches you industry-standard ops: Kubernetes (K8s).

---

## Why Kubernetes

By now you've deployed on AWS:
- ECS Fargate runs containers
- CloudWatch monitors them
- GitHub Actions auto-deploys

But as your system grows:
- One API server dies → manual restart? That's not infrast...

Actually, K8s restarts it automatically.

- Need 100 API servers for Black Friday? Scale with one command
- Need to update the image? K8s rolls out gradually, no downtime
- Need to run on AWS, GCP, Azure? Same K8s manifests everywhere

**K8s is the industry standard** because it solves operational complexity.

Every major company runs K8s in production: Google, Netflix, Stripe, OpenAI, Anthropic.

If you're serious about AI operations, you need this.

---

## What You'll Learn

This phase teaches you to:

**Week 1: Kubernetes Fundamentals**
- Pods, Deployments, Services (the core concepts)
- kubectl commands to inspect and debug
- Deploy your first AI service on Minikube
- Understand resource requests and limits
- ConfigMaps and Secrets for configuration

**Week 2: AI on Kubernetes**
- Deploy FastAPI + Qdrant + PostgreSQL on K8s
- StatefulSets for databases (why they're different)
- Persistent storage (volumes)
- Helm charts (package manager for K8s)
- Ingress controller for routing

**Week 3: Scaling and Observability**
- Horizontal Pod Autoscaler: auto-scale based on CPU
- KEDA: auto-scale based on queue depth
- Prometheus metrics and Grafana dashboards
- Monitoring LLM inference latency
- AWS EKS: production-grade managed K8s

**Week 4: Integration and Capstone**
- Full stack on Minikube (development)
- Deploy to AWS EKS (production)
- Auto-scaling under load
- Incident response
- Capstone: production K8s AI platform

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Control Plane (Master)                                  │
│  ├─ API Server (manage everything)                       │
│  ├─ Scheduler (assign pods to nodes)                     │
│  ├─ Controller Manager (desired state)                   │
│  └─ etcd (database of everything)                        │
│                                                           │
│  Worker Nodes (actual servers)                           │
│  ├─ Node 1: [ Pod (FastAPI) ] [ Pod (Monitoring) ]      │
│  ├─ Node 2: [ Pod (FastAPI) ] [ Pod (Metrics) ]         │
│  └─ Node 3: [ Pod (Qdrant) ] [ Pod (Cache) ]            │
│                                                           │
│  Services & Ingress                                      │
│  ├─ ClusterIP Service: internal routing                  │
│  ├─ LoadBalancer Service: external traffic              │
│  └─ Ingress: URL path routing                           │
│                                                           │
└─────────────────────────────────────────────────────────┘
          ↓ Traffic ↓
    Internet clients
```

---

## Key Concepts Preview

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **Cluster** | Collection of nodes managed as one | Fleet of servers managed together |
| **Node** | A physical/virtual server | Individual ship in the fleet |
| **Pod** | Running container(s) | Shipping container on the ship |
| **Deployment** | "Keep 3 replicas of this pod running" | Chef manager: "I want 3 cooks always working" |
| **Service** | Stable address to reach pods (they come/go) | Restaurant phone number (chefs change, number stays) |
| **Ingress** | Route external traffic by URL path | Hotel front desk routing guests |
| **ConfigMap** | Non-secret config (env vars, files) | Config file in the cluster |
| **Secret** | Secret config (API keys, passwords) | Encrypted config in the cluster |
| **PersistentVolume** | Storage that outlives pods | Storage that survives restarts |
| **StatefulSet** | Deployment for stateful services (DBs) | Deployment that preserves identity |

---

## Phase 4 Topics

1. **[Kubernetes →](./kubernetes.md)** — Understand K8s concepts, kubectl commands, resource requests/limits
2. **[AI on Kubernetes →](./ai-on-k8s.md)** — Deploy your entire AI stack (API + Qdrant + PostgreSQL + Redis)
3. **[Scaling & Observability →](./scaling.md)** — Autoscaling, Prometheus, Grafana, EKS, incident response
4. **[Milestone →](./milestone.md)** — Capstone project: production K8s platform

---

## Prerequisites

You need:
- Completed Phase 3 (MLOps, experimentation)
- Understanding of Docker (images, containers, Compose)
- AWS familiarity (EC2, ECS, RDS basics)

If you haven't done Phase 3, do it first. You can't manage something you don't understand.

---

## Time Estimate

**3-4 weeks** of focused work:
- Week 1: Fundamentals (minikube setup, kubectl, basic deployments)
- Week 2: Full stack on Minikube (all your services together)
- Week 3: Production patterns (scaling, monitoring, EKS)
- Week 4: Capstone (build, deploy, load test, incident response)

If you already know K8s, this moves faster. If not, don't rush — K8s has a steep learning curve.

---

## Honest Assessment

### K8s is Worth Learning

✅ You will use K8s in production AI roles
✅ Required knowledge at every major tech company
✅ Once you understand it, complexity becomes manageable
✅ Enables you to operate at massive scale

### But K8s has a steep vocabulary curve

⚠️ Lots of new concepts (pods, services, ingress, statefulsets, CRDs, operators...)
⚠️ Errors are sometimes cryptic (ImagePullBackOff, CrashLoopBackOff)
⚠️ "It works on my machine" (minikube) vs production (EKS) surprises
⚠️ 80% of what you learn applies to 20% of problems; the other 20% is very specific

**Strategy:** Learn core concepts (pods, deployments, services) deeply. Don't memorize every YAML field. Use the reference docs.

---

## Next Steps

Start with [Kubernetes Fundamentals →](./kubernetes.md)

You'll understand K8s inside 3 weeks. Then you'll wonder how you ever lived without it.

Let's go.
