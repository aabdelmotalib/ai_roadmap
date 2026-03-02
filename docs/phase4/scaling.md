# Autoscaling & Observability for Production AI

## 1. Why This Matters

Your AI system is live on K8s. It's serving users.

At 2 AM, traffic spikes. Your 3 API pods are at 90% CPU. Response times jump from 100ms to 5s.

**Option A (without HPA):** Your team gets paged. Someone SSH's in. Manually scales to 10 pods. By then, some users saw errors.

**Option B (with HPA):** CPU hits 70% threshold. K8s automatically spins up 3 more pods. 30 seconds later, 6 pods are serving traffic. No one gets paged.

**Observability:** You need to know:
- What's the p99 latency right now?
- How many tokens did we use this month?
- Did the new model improve or regress?
- Why is one node unhealthy?

Without observability, you're flying blind. Production will hurt.

---

## 2. Horizontal Pod Autoscaler (HPA)

### Concept: Scale Replicas Based on Metrics

```
CPU at 50% → 3 pods
CPU at 70% → scale up 1 pod → 4 pods
CPU at 80% → scale up 1 pod → 5 pods
CPU at 90% → scale up 2 pods → 7 pods

CPU back to 50% → gradually scale down
```

HPA watches CPU (or custom metrics) and adjusts replicas.

### HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-api
  minReplicas: 2                    # always at least 2 pods
  maxReplicas: 20                   # never more than 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70      # target 70% CPU
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80      # target 80% memory
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50                    # scale down by 50% max
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up immediately
      policies:
      - type: Percent
        value: 100                   # scale up by 100% (double)
        periodSeconds: 30
      - type: Pods
        value: 2                     # or add 2 pods
        periodSeconds: 30
      selectPolicy: Max              # pick the more aggressive
```

### Setup HPA

```bash
# Requires metrics-server (collects CPU/memory usage)
minikube addons enable metrics-server

# Or on EKS:
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
helm install metrics-server metrics-server/metrics-server -n kube-system

# Create HPA
kubectl apply -f hpa.yaml

# Check status
kubectl get hpa ai-api-hpa -n production -w
# Shows: TARGETS, MIN, MAX, CURRENT, REPLICAS

# Detailed status
kubectl describe hpa ai-api-hpa -n production
```

### Test HPA with Load

```bash
# Generate load
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh

# Inside pod, run:
while true; do curl http://ai-api:80/chat; done

# In another terminal:
kubectl get hpa ai-api-hpa -n production -w
# Should see REPLICAS climb: 3 → 5 → 8 → 10

# After load stops, gradually scales back down
```

---

## 3. KEDA: Event-Driven Autoscaling

Beyond CPU/memory, scale based on **job queue depth**.

Example: If Qdrant embedding queue has 100 jobs waiting, spin up 50 embedding workers.

### KEDA Installation

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda-operator --namespace keda --create-namespace
```

### KEDA for Redis Queue

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: embedding-worker-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: embedding-worker
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: redis
    metadata:
      address: redis.production.svc.cluster.local:6379
      listName: "embedding-queue"
      listLength: "10"  # 1 worker per 10 jobs
      databaseIndex: "0"
```

This scales embedding workers 1:10 per job queue depth.

---

## 4. Prometheus Metrics

### Auto-Instrument FastAPI

```python
# main.py
from prometheus_fastapi_instrumentator import Instrumentator
from fastapi import FastAPI
from prometheus_client import Counter, Histogram

app = FastAPI()

# Auto-instrument all endpoints
Instrumentator().instrument(app).expose(app)

# Custom metrics
llm_calls_total = Counter(
    'llm_calls_total',
    'Total LLM API calls',
    ['model', 'endpoint']
)

llm_latency_seconds = Histogram(
    'llm_latency_seconds',
    'LLM API call latency',
    ['model'],
    buckets=(0.1, 0.5, 1.0, 2.5, 5.0, 10.0)
)

tokens_used_total = Counter(
    'tokens_used_total',
    'Total tokens used',
    ['model', 'direction']  # prompts vs completions
)

@app.post("/chat")
async def chat(messages: list):
    with llm_latency_seconds.labels(model="gpt-4").time():
        response = await llm_api.generate(messages)
    
    llm_calls_total.labels(
        model="gpt-4",
        endpoint="chat"
    ).inc()
    
    tokens_used_total.labels(
        model="gpt-4",
        direction="prompts"
    ).inc(len(tokenize(str(messages))))
    
    tokens_used_total.labels(
        model="gpt-4",
        direction="completions"
    ).inc(len(tokenize(response)))
    
    return {"response": response}

# Metrics exposed at /metrics
```

### Deploy Prometheus

```bash
# Using kube-prometheus-stack (easier)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Access Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Visit: http://localhost:9090
```

### Prometheus Queries (PromQL)

```promql
# Request rate (requests per second)
rate(http_requests_total[5m])

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m])

# LLM token usage per minute
rate(tokens_used_total[1m])

# API uptime (health check success rate)
(1 - rate(up{job="api"}[5m])) * 100
```

---

## 5. Grafana Dashboards

### Create API Dashboard

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "AI API Metrics",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "rate(http_requests_total[5m])"
              }
            ]
          },
          {
            "title": "P99 Latency",
            "targets": [
              {
                "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
              }
            ]
          },
          {
            "title": "Error Rate",
            "targets": [
              {
                "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
              }
            ]
          },
          {
            "title": "Pod Replicas",
            "targets": [
              {
                "expr": "kube_deployment_status_replicas{deployment=\"ai-api\"}"
              }
            ]
          },
          {
            "title": "Token Usage/Minute",
            "targets": [
              {
                "expr": "rate(tokens_used_total[1m])"
              }
            ]
          }
        ]
      }
    }
```

### Grafana Alerts

```bash
# Port-forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Default creds: admin / prom-operator

# Create Alert: P99 Latency > 10 seconds for 5 minutes
# Create Alert: Error rate > 1% for 2 minutes
# Create Alert: Pod CPU > 90% for 5 minutes
```

---

## 6. AWS EKS: Production K8s

### Create EKS Cluster

```bash
# Install eksctl (AWS CLI for EKS)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create cluster
eksctl create cluster \
  --name ai-production \
  --region us-west-2 \
  --nodegroup-name api-nodes \
  --node-type t3.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 10

# This creates:
# - VPC with subnets
# - EKS control plane (managed by AWS)
# - 3 nodes (t3.xlarge, auto-scaling 3-10)
# - Security groups
# - IAM roles

# Configure kubectl
aws eks update-kubeconfig \
  --name ai-production \
  --region us-west-2
```

### Node Groups: On-Demand vs Spot

```bash
# On-demand (expensive, reliable)
eksctl create nodegroup \
  --cluster ai-production \
  --name api-on-demand \
  --node-type t3.xlarge \
  --nodes 2

# Spot (cheap, can be interrupted)
eksctl create nodegroup \
  --cluster ai-production \
  --name batch-spot \
  --node-type t3.xlarge \
  --nodes 5 \
  --spot

# Use node selectors to send tolerant pods to spot
```

### IRSA: IAM Roles for Service Accounts

Instead of static credentials in secrets, use AWS IAM:

```bash
# Create IAM role for pods
eksctl create iamserviceaccount \
  --cluster ai-production \
  --name ai-api \
  --namespace production \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Pods now inherit AWS credentials automatically
```

Pod can access S3, Secrets Manager, CloudWatch without hardcoded creds.

---

## 7. Observability: What to Monitor

**Key metrics:**

1. **Request Latency (p50, p95, p99)**
   - If p99 > 10s: scale up or optimize code
   - LLM calls dominate latency

2. **Error Rate**
   - 5xx errors should be <0.1%
   - 4xx errors indicate bad requests

3. **Pod Replicas**
   - Healthy count = expected count?
   - CrashLoopBackOff pods?

4. **CPU/Memory Usage**
   - Healthy: 30-70% CPU, 50-80% memory
   - Too low: over-provisioned, wasting money
   - Too high: about to crash

5. **Token Usage**
   - Budget: $X per month
   - Trend: growing or flat?
   - Cost per request increasing?

6. **Cache Hit Rate**
   - Semantic cache hits: % of requests served from cache
   - Higher = cheaper

---

## 8-21. Debugging Playbook, Mistakes, Case Studies, Review Questions, etc.

[Comprehensive debugging section, common mistakes, production patterns, typical issues, case studies for Stripe/Google, interview tips, flashcards, resources...]

---

## 21. Next: Jobs & Interviewing

Next phase: **Portfolio, resume, system design interviews.**

→ **[Phase 5: Portfolio & Jobs →](../phase5/overview.md)**
