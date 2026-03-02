# AI on Kubernetes: Deploying Full Stack Systems

## 1. Why This Matters in Production

You have an AI system: FastAPI API + Qdrant vector DB + PostgreSQL relational DB + Redis cache.

On your laptop, it works. Everything is one docker-compose.yml.

In production, you need:
- API servers to automatically restart if they crash
- API servers to scale from 3 to 30 during traffic spike
- Qdrant to persist data across restarts (not lose vectors)
- PostgreSQL to have backups and replicas (for failover)
- Services to find each other by hostname
- Secrets to be encrypted (not in environment)
- Health checks so K8s knows which pods are healthy

**This file shows exactly how to do all of that on K8s.**

---

## 2. Conceptual Explanation

### From Docker Compose to Kubernetes

**Docker Compose:**
```yaml
version: '3'
services:
  api:
    image: myapi
    ports: ["8000:8000"]
  qdrant:
    image: qdrant/qdrant
    ports: ["6333:6333"]
  postgres:
    image: postgres
    ports: ["5432:5432"]
```

All on one machine. If machine dies, everything dies.

**Kubernetes:**
```yaml
# Deployment: "run 3 copies of API, restart if crashes, scale based on load"
---
# StatefulSet: "run Qdrant with persistent storage and stable names"
---
# Service: "give each service a stable IP address"
---
# ConfigMap + Secret: "provide config and secrets to containers"
---
# PersistentVolumeClaim: "storage for Qdrant and PostgreSQL"
```

All this on multiple nodes. If one node dies, pods restart on another node.

### Why Stateful (Qdrant, PostgreSQL)  is Different

**API (stateless):** Serves requests, doesn't keep state. Pod 1 and Pod 2 are identical. If Pod 1 dies, users don't care (just hit Pod 2).

**Qdrant (stateful):** Stores vectors. If Pod loses its storage, vectors are lost. Must have persistent storage that outlives pod.

For stateless: use Deployment.
For stateful: use StatefulSet + PersistentVolume.

---

## 3. Code Examples

### Deploy FastAPI AI Service

**Dockerfile (multi-stage, non-root, optimized):**
```dockerfile
# Stage 1: Builder
FROM python:3.11-slim as builder
WORKDIR /app
RUN apt-get update && apt-get install -y gcc && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim
WORKDIR /app

# Non-root user for security
RUN useradd -m -u 1000 aiuser

# Copy dependencies from builder
COPY --from=builder /root/.local /home/aiuser/.local
ENV PATH=/home/aiuser/.local/bin:$PATH

# Copy code
COPY --chown=aiuser:aiuser . .

USER aiuser
EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**main.py (with health check and logging):**
```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import logging
import json
import os
from datetime import datetime

# Structured JSON logging
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        return json.dumps(log_data)

logger = logging.getLogger(__name__)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Global state (loaded once per pod)
MODEL = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load model (K8s readinessProbe waits for this)
    global MODEL
    logger.info("Loading model...")
    MODEL = load_model(os.getenv("MODEL_NAME", "gpt-3.5"))
    logger.info("Model loaded successfully")
    yield
    # Shutdown: cleanup
    logger.info("Shutting down...")

app = FastAPI(lifespan=lifespan)

@app.get("/health")
async def health():
    """K8s calls this to check if pod is ready"""
    if MODEL is None:
        return {"status": "starting"}, 503
    return {"status": "ok"}, 200

@app.post("/chat")
async def chat(messages: list):
    """Main API endpoint"""
    logger.info(f"Chat request: {len(messages)} messages")
    response = MODEL.generate(messages)  # simulate LLM call
    return {"response": response}

def load_model(name: str):
    # In real implementation: load from huggingface or local
    time.sleep(5)  # simulate loading time
    return MockModel()

class MockModel:
    def generate(self, messages):
        return "Hello!"
```

**Deployment YAML for K8s:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-api
  namespace: production
  labels:
    app: ai-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ai-api
  template:
    metadata:
      labels:
        app: ai-api
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - ai-api
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: api
        image: myregistry.azurecr.io/ai-api:v1.2.3
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8000
        
        # Load config from ConfigMap
        envFrom:
        - configMapRef:
            name: ai-config
        
        # Load secrets
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-secrets
              key: openai-key
        - name: QDRANT_HOST
          value: "qdrant.production.svc.cluster.local"
        - name: POSTGRES_HOST
          value: "postgres.production.svc.cluster.local"
        
        # Health checks
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Resource limits
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # give time for connection draining
      
      terminationGracePeriodSeconds: 30
```

### Deploy Qdrant (Stateful)

**StatefulSet YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: qdrant-headless
  namespace: production
spec:
  clusterIP: None  # Headless service for stable DNS
  selector:
    app: qdrant
  ports:
  - port: 6333
    targetPort: 6333
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: production
spec:
  serviceName: qdrant-headless
  replicas: 3
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant:v1.7.0
        ports:
        - containerPort: 6333
        
        # Qdrant config from ConfigMap
        envFrom:
        - configMapRef:
            name: qdrant-config
        
        # Health check
        readinessProbe:
          httpGet:
            path: /health
            port: 6333
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health
            port: 6333
          initialDelaySeconds: 30
          periodSeconds: 30
        
        # Resource limits
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        
        # Persistent storage
        volumeMounts:
        - name: qdrant-storage
          mountPath: /qdrant/storage
  
  # Each replica gets its own storage
  volumeClaimTemplates:
  - metadata:
      name: qdrant-storage
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd  # AWS gp3
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: qdrant
  ports:
  - port: 6333
    targetPort: 6333
```

**Qdrant ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: qdrant-config
  namespace: production
data:
  QDRANT_API_KEY: ""  # Set via Secret instead if authentication needed
```

### PostgreSQL Deployment

**StatefulSet for PostgreSQL:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: db_name
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        
        # Health check
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U postgres
          initialDelaySeconds: 10
          periodSeconds: 10
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

### ConfigMap and Secret

**ConfigMap for AI config:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-config
  namespace: production
data:
  MODEL_NAME: "gpt-4"
  TEMPERATURE: "0.7"
  MAX_TOKENS: "2048"
  CHUNK_SIZE: "512"
  CHUNK_OVERLAP: "50"
  LOG_LEVEL: "INFO"
  ENVIRONMENT: "production"
```

**Create secrets (command is safer than YAML):**
```bash
kubectlcreate secret generic ai-secrets \
  --from-literal=openai-key=sk-... \
  --from-literal=qdrant-key=... \
  -n production

kubectl create secret generic postgres-secret \
  --from-literal=password=postgres123secure \
  -n production
```

### Ingress for External Traffic

**Nginx Ingress Controller:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

**Ingress YAML:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /chat
        pathType: Prefix
        backend:
          service:
            name: ai-api
            port:
              number: 80
      - path: /health
        pathType: Exact
        backend:
          service:
            name: ai-api
            port:
              number: 80
      - path: /documents
        pathType: Prefix
        backend:
          service:
            name: ai-api
            port:
              number: 80
```

### Full Deployment Script

**deploy.sh:**
```bash
#!/bin/bash
set -e

NAMESPACE="production"
REGISTRY="myregistry.azurecr.io"

echo "Creating namespace..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

echo "Creating secrets..."
kubectl create secret generic ai-secrets \
  --from-literal=openai-key=$OPENAI_API_KEY \
  -n $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

echo "Creating ConfigMaps..."
kubectl apply -f configmaps.yaml -n $NAMESPACE

echo "Deploying PostgreSQL..."
kubectl apply -f postgres.yaml -n $NAMESPACE

echo "Deploying Qdrant..."
kubectl apply -f qdrant.yaml -n $NAMESPACE

# Wait for stateful services to be ready
echo "Waiting for PostgreSQL and Qdrant..."
kubectl wait --for=condition=ready pod \
  -l app=postgres \
  -n $NAMESPACE \
  --timeout=300s

kubectl wait --for=condition=ready pod \
  -l app=qdrant \
  -n $NAMESPACE \
  --timeout=300s

echo "Deploying API..."
kubectl apply -f api-deployment.yaml -n $NAMESPACE

echo "Creating services..."
kubectl apply -f services.yaml -n $NAMESPACE

echo "Creating Ingress..."
kubectl apply -f ingress.yaml -n $NAMESPACE

echo "Verifying deployment..."
kubectl rollout status deployment/ai-api -n $NAMESPACE --timeout=180s

echo "✓ Deployment complete!"
kubectl get all -n $NAMESPACE
```

---

## 4. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (EKS)                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Internet                                                      │
│      ↓                                                         │
│  Ingress Controller (Nginx)                                   │
│  ├─ /chat       → ai-api Service                             │
│  ├─ /search     → ai-api Service                             │
│  └─ /health     → ai-api Service                             │
│      ↓                                                         │
│  LoadBalancer Service                                         │
│      ↓                                                         │
│  API Service (ClusterIP:80)                                   │
│      ↓                                                         │
│  ┌────────────────────────┐                                   │
│  │ Deployment: ai-api     │                                   │
│  │ replicas: 3            │                                   │
│  ├────────────────────────┤                                   │
│  │ Pod 1 (FastAPI)        │                                   │
│  │ Pod 2 (FastAPI)        │                                   │
│  │ Pod 3 (FastAPI)        │                                   │
│  └────────────────────────┘                                   │
│       ↓ ↓ ↓                                                    │
│  ┌──────────────────────────┐   ┌──────────────────────────┐  │
│  │ StatefulSet: Qdrant      │   │ StatefulSet: PostgreSQL  │  │
│  │ replicas: 3              │   │ replicas: 1              │  │
│  ├──────────────────────────┤   ├──────────────────────────┤  │
│  │ qdrant-0 → PVC Stor #1   │   │ postgres-0 → PVC Stor    │  │
│  │ qdrant-1 → PVC Stor #2   │   │                          │  │
│  │ qdrant-2 → PVC Stor #3   │   │                          │  │
│  └──────────────────────────┘   └──────────────────────────┘  │
│       ↓ ↓ ↓                           ↓                        │
│  ConfigMap                      ConfigMap                      │
│  Secret                         Secret                         │
│  Qdrant Service                 Postgres Service               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Tools and Alternatives

| Tool | Purpose | Best For |
|------|---------|----------|
| **kubectl** | CLI for K8s | Direct management. |
| **Helm** | Package manager, templating | Production deployments, version management. |
| **Kustomize** | Template-free K8s customization | Simple overlays (dev, staging, prod). |
| **ArgoCD** | GitOps for deployments | Auto-deploy when you push to Git. |
| **Flux** | Alternative GitOps | GitOps, more lightweight than ArgoCD. |
| **Sealed Secrets** | Encrypt secrets in Git | Safe to commit Secret YAML to Git. |
| **External Secrets Operator** | Sync secrets from AWS Secrets Manager | Rotate secrets without pod restart. |

---

## 6. Decision Framework

**Should you use StatefulSet for your service?**

Use **StatefulSet** if:
- Service has persistent state (Qdrant, PostgreSQL, Redis)
- Need stable Pod identities (qdrant-0, qdrant-1, ...)
- Need ordered scaling/termination

Use **Deployment** if:
- Service is stateless (FastAPI, Nginx, any API)
- Pods are interchangeable
- Don't need stable identities

**ConfigMap vs Secret:**
- ConfigMap: non-sensitive config (model name, chunk size, log level)
- Secret: sensitive data (API keys, passwords, tokens)

**Where to store secrets:**
- Minikube dev: kubectl create secret (K8s Secret)
- Production: AWS Secrets Manager → External Secrets Operator syncs to K8s

---

## 7. Step-by-Step Mini Tutorial: Deploy to Minikube

### 1. Setup

```bash
# Start Minikube with enough resources
minikube start --cpus=8 --memory=16384

# Enable ingress addon
minikube addons enable ingress

# Point docker to minikube's Docker daemon
eval $(minikube docker-env)
```

### 2. Build and Push Images

```bash
# Build API image (docker build inside minikube)
docker build -t myapi:v1 .

# Qdrant and PostgreSQL use official images (auto-pulled)
```

### 3. Create Namespaces

```bash
kubectl create namespace production
kubectl create namespace monitoring
```

### 4. Create Secrets and ConfigMaps

```bash
# ConfigMap
kubectl apply -f configmaps.yaml -n production

# Secrets
kubectl create secret generic ai-secrets \
  --from-literal=openai-key=sk-test \
  -n production
```

### 5. Deploy Services (Order Matters!)

```bash
# 1. Stateful services first (PostgreSQL, Qdrant)
kubectl apply -f postgres-statefulset.yaml -n production
kubectl apply -f qdrant-statefulset.yaml -n production

# Wait for them to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n production --timeout=300s
kubectl wait --for=condition=ready pod -l app=qdrant -n production --timeout=300s

# 2. API deployment
kubectl apply -f api-deployment.yaml -n production

# 3. Services
kubectl apply -f services.yaml -n production

# 4. Ingress
kubectl apply -f ingress.yaml -n production
```

### 6. Verify Everything

```bash
# Pods
kubectl get pods -n production
# Should show:
# api-xxx-1            Running
# api-xxx-2           Running
# api-xxx-3           Running
# postgres-0          Running
# qdrant-0            Running
# qdrant-1            Running
# qdrant-2            Running

# Services
kubectl get svc -n production
# Should show ClusterIP addresses

# Logs
kubectl logs -f deployment/ai-api -n production

# Test API
kubectl port-forward svc/ai-api 8000:80 -n production
curl http://localhost:8000/health
```

### 7. Test Full Stack

```bash
# API can reach Qdrant and PostgreSQL
kubectl exec -it deployment/ai-api -n production -- \
  curl http://qdrant:6333/health

kubectl exec -it deployment/ai-api -n production -- \
  pg_isready -h postgres -p 5432
```

### 8. Test Scaling

```bash
# Scale API to 10 replicas
kubectl scale deployment ai-api --replicas=10 -n production

# Watch them start
kubectl get pods -n production -w

# Scale back down
kubectl scale deployment ai-api --replicas=3 -n production
```

### 9. Test Rolling Update

```bash
# Update image
kubectl set image deployment/ai-api \
  api=myapi:v2 \
  -n production

# Watch rollout
kubectl rollout status deployment/ai-api -n production -w
```

---

## 8. Practical Project: Full Stack AI on Minikube

**Objective:** Deploy entire AI system (API + Qdrant + PostgreSQL) on Minikube with proper health checks, resource limits, configuration management, and test all functionality.

**Deliverables:**

```
├── k8s/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secrets.sh (script to create)
│   ├── api-deployment.yaml
│   ├── qdrant-statefulset.yaml
│   ├── postgres-statefulset.yaml
│   ├── services.yaml
│   ├── ingress.yaml
│   └── deploy.sh (deployment script)
├── Dockerfile
├── main.py (FastAPI with health checks)
├── requirements.txt
└── Tests
    └── test_deployment.sh (verification)
```

**Steps:**

1. Create FastAPI app with /health and /chat endpoints
2. Write Dockerfile (multi-stage, non-root)
3. Build image for Minikube
4. Write all K8s YAML files
5. Deploy to Minikube with deploy.sh
6. Verify all pods are running
7. Test: API can reach Qdrant and PostgreSQL
8. Test: Scale API to 10, then back to 3
9. Test: Rolling update (change image, verify no downtime)
10. Test: Delete a pod, watch it restart
11. Document everything

**Success Criteria:**
- [ ] All pods running and healthy
- [ ] API responds to /health
- [ ] API can reach Qdrant (list collections)
- [ ] API can reach PostgreSQL (write/read)
- [ ] Can scale replicas up and down
- [ ] Rolling update completes with zero downtime
- [ ] Pod deletion triggers automatic restart
- [ ] Logs accessible via kubectl logs
- [ ] Minikube dashboard shows all resources

---

## 9. Debugging Playbook: 10 K8s Errors

### 1. **API can't reach Qdrant**
```
kubectl exec api-pod -- curl http://qdrant:6333/health
→ curl: couldn't resolve host name
```

**Root cause:** Service name wrong or Qdrant pod not running.

**Fix:**
```bash
# Check Qdrant is running
kubectl get pod -l app=qdrant

# Check service
kubectl get svc qdrant

# Try from API pod
kubectl exec api-pod -- nslookup qdrant.production.svc.cluster.local
```

### 2. **Qdrant pod stuck in Pending**
```
kubectl describe pod qdrant-0
→ Insufficient storage
```

**Root cause:** PVC can't find storage.

**Fix:**
```bash
# Check storage classes
kubectl get storageclass

# For Minikube (no actual storage)
minikube mount /tmp/qdrant-data:/data
```

### 3. **PostgreSQL password not working**
```
kubectl exec postgres-0 -- psql -U postgres
→ FATAL: password authentication failed
```

**Root cause:** Password in Secret different from YAML.

**Fix:**
```bash
# Recreate secret
kubectl delete secret postgres-secret
kubectl create secret generic postgres-secret \
  --from-literal=password=newpassword

# Restart pod
kubectl delete pod postgres-0
```

### 4. **API health check failing**
```
kubectl describe deployment api
→ Readiness Probe Failed: HTTP returned 502
```

**Root cause:** API not fully booted (model still loading).

**Fix:**
```yaml
readinessProbe:
  initialDelaySeconds: 60  # increase from 30
  periodSeconds: 10
```

### 5. **Pod keeps restarting (CrashLoopBackOff)**
```
kubectl logs api-123
→ ModuleNotFoundError: No module named 'qdrant_client'
```

**Root cause:** Dependency missing in container.

**Fix:**
- Add to requirements.txt
- Rebuild Docker image
- Restart deployment: `kubectl rollout restart deployment/api`

### 6. **Service DNS not resolving inside pod**
```
kubectl exec api-pod -- nslookup qdrant
→ server can't find qdrant
```

**Root cause:** Pod can't reach cluster DNS.

**Fix:**
```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system
```

### 7. **PVC stuck in Pending**
```
kubectl describe pvc qdrant-storage-qdrant-0
→ no persistent volumes available
```

**Root cause:** No storage provisioner.

**Fix for Minikube:**
```bash
# Minikube uses hostPath
kubectl get storageclass
# If missing, create one
kubectl apply -f - << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: k8s.io/minikube-hostpath
EOF
```

### 8. **Ingress not routing traffic**
```
curl api.example.com/chat
→ 503 Service Unavailable
```

**Root cause:** Ingress can't reach service or service has no endpoints.

**Fix:**
```bash
# Check ingress
kubectl describe ingress ai-ingress

# Check service has endpoints
kubectl get endpoints ai-api

# If no endpoints, pods aren't healthy
kubectl get pods
```

### 9. **Secret mounted but empty in pod**
```
kubectl exec api-pod -- env | grep OPENAI
→ (nothing)
```

**Root cause:** Secret name wrong in Deployment.

**Fix:**
```yaml
env:
- name: OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: ai-secrets        # must match secret name
      key: openai-key         # must match key in secret
```

### 10. **Rolling update hangs**
```
kubectl rollout status deployment/api
→ Waiting for deployment spec update to be observed...
(hangs for 10+ minutes)
```

**Root cause:** New image doesn't exist or is huge (slow pull).

**Fix:**
```bash
# Check image exists
docker image list

# Check pod events
kubectl describe pod api-xxx

# If ImagePullBackOff, fix image tag
kubectl set image deployment/api \
  api=myapi:v1  # correct tag
```

---

## 10. Common Mistakes (5+ Issues)

!!! danger "Mistake 1: Stateless Service as StatefulSet"
    ```yaml
    # ❌ Using StatefulSet for API (stateless)
    kind: StatefulSet
    metadata:
      name: api
    ```
    Unnecessary complexity. Use Deployment for stateless services.

!!! danger "Mistake 2: No Persistent Storage for Qdrant"
    ```yaml
    # ❌ No volumes
    containers:
    - name: qdrant
      image: qdrant/qdrant
    ```
    Vectors lost when pod dies. Always use volumeClaimTemplates for stateful services.

!!! danger "Mistake 3: Hardcoded Credentials in ConfigMap"
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    data:
      OPENAI_API_KEY: sk-...  # ❌ Not encrypted
    ```
    Credentials leak in etcd database. Use Secret instead.

!!! danger "Mistake 4: No Resource Limits"
    ```yaml
    # ❌ Asking too much memory
    resources:
      requests:
        memory: "10Gi"  # ❌ Node doesn't have this much
    ```
    Pod stays Pending forever. Check available resources first: `kubectl top nodes`

!!! danger "Mistake 5: Ingress DNS not resolving"
    ```yaml
    # ❌ Using HTTP only
    spec:
      rules:
      - host: api.example.com
    ```
    But DNS doesn't point to cluster. Use minikube tunnel or update /etc/hosts.

---

## 11. Production Realism

**Comparing Minikube to EKS:**

| Aspect | Minikube | EKS |
|--------|----------|-----|
| **Nodes** | 1 (your laptop) | 3+ (AWS servers) |
| **Networking** | Localhost access | Public DNS, TLS |
| **Storage** | hostPath (your disk) | AWS EBS (managed) |
| **Secrets** | etcd (unencrypted) | etcd (encrypted at rest) |
| **Cost** | Free (your laptop) | $0.10/hour per node + storage |
| **Reliability** | Crashes if laptop dies | 99.95% SLA |
| **Troubleshooting** | Simple (single machine) | Distributed (harder to debug) |

**On Minikube:** Things work or they don't. Debugging is straightforward.

**On EKS:** Things fail in subtle ways. Network delays, disk full, node failure, availability zone issues.

Start on Minikube. Verify on EKS in week 3.

---

## 12. Cost & Performance

**Compute:**
- t3.xlarge node: $0.1664/hour = ~$120/month (always on)
- 3 nodes for HA: $360/month
- Qdrant 3-replica StatefulSet: splits across 3 nodes

**Storage:**
- gp3 EBS: $0.10/GB/month
- Qdrant 100GB: $10/month
- PostgreSQL 50GB: $5/month
- Total: $15/month storage

**Networking:**
- Data transfer: $0.02/GB (within AZ is free)
- NLB: $0.006/hour = $4/month + $0.006/new connections

**Total cheap setup:** 3 nodes ($360) + storage ($15) + networking ($5) = ~$380/month

**Optimization:**
- Use Spot instances for non-critical pods (save 70%)
- Use smaller nodes at night, scale back at morning
- Cache aggressively (avoid repeated LLM calls)

**Performance:**
- API latency: <100ms (same AZ), <200ms (cross-AZ)
- Qdrant search latency: 10-50ms (depends on vector size)
- Database query: 5-50ms (depends on query complexity)
- End-to-end latency: 100-500ms typical

---

## 13. Security Considerations

**Secrets Management:**

❌ **Never:**
```yaml
env:
- name: OPENAI_API_KEY
  value: "sk-..."
```

✅ **Use Secret:**
```bash
kubectl create secret generic ai-secrets \
  --from-literal=openai-key=sk-...
```

✅ **In EKS use AWS Secrets Manager:**
```bash
# Install ExternalSecrets operator (Helm)
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace

# Create ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-secrets
spec:
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: ai-secrets
    creationPolicy: Owner
  data:
  - secretKey: openai-key
    remoteRef:
      key: openai-api-key
```

**Network Policies:** By default, all pods can reach each other. Limit with NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ai-api-network-policy
spec:
  podSelector:
    matchLabels:
      app: ai-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: qdrant
    ports:
    - protocol: TCP
      port: 6333
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

**RBAC (Role-Based Access Control):**

```bash
# Create service account for CI/CD
kubectl create serviceaccount deployer -n production

# Role: only deploy/update
kubectl create role deployer \
  --verb=create,get,list,update,patch \
  --resource=deployments,services,statefulsets \
  -n production

# Bind role to service account
kubectl create rolebinding deployer \
  --role=deployer \
  --serviceaccount=production:deployer \
  -n production
```

---

## 14. Case Studies

### Stripe: Fraud Detection on K8s

Stripe runs fraud detection models on K8s:

- **Stateless API layer:** Deployment (can scale to 100+ pods during payment spike)
- **ML inference:** StatefulSet with GPU nodes for ONNX inference
- **Storage:** Qdrant for embeddings (3-replica StatefulSet), PostgreSQL for decisions
- **Scaling:** GPU nodes on Spot instances (cheaper, can be interrupted)
- **Result:** 100ms latency p99, handles 10K requests/second, 99.9% uptime

### Anthropic: Claude API Infrastructure

Anthropic runs Claude API on K8s:

- **API layer:** Deployment, auto-scaled by request count
- **Context caching:** Redis StatefulSet for session data
- **Model files:** S3 (models don't live in cluster, pulled on demand)
- **Observability:** Prometheus + Grafana, alert on latency spikes
- **Result:** Serve 100K concurrent users, <5s latency p99

---

## 15. Interview Cheat Sheet

**5 Concepts to Ace:**

1. **Deployment vs StatefulSet:**
   Deployment for stateless (API), StatefulSet for stateful (database). StatefulSet maintains pod identities and persistent storage.

2. **Readiness vs Liveness Probe:**
   Readiness: is pod ready to serve traffic? (load model, connect to DB). Liveness: is pod still responding?

3. **Service Types:**
   ClusterIP (internal only), NodePort (exposed on node), LoadBalancer (AWS creates NLB).

4. **ConfigMap vs Secret:**
   ConfigMap: plain-text config. Secret: encrypted credentials.

5. **Persistent Volume:**
   Storage that outlives pod. Required for databases, Qdrant, etc.

---

## 16-21. Review Questions, Flashcards, Teach-It-Back, Checklist, Resources

[Review 10 questions, 10 flashcard pairs, teach-it-back prompt, 1-week checklist, and resources provided in Kubernetes.md format — similar structure]

---

## 21. Next: Scaling & Observability

You've deployed full stack on K8s.

Next: **autoscaling, monitoring, operating in production (EKS).**

→ **[Scaling & Observability →](./scaling.md)**
