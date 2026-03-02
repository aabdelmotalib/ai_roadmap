# Kubernetes Fundamentals for AI Engineers

## 1. Why This Matters in Production

Imagine you're running your RAG API on AWS with Docker Compose. You have 100 requests/second.

One container dies. **Your API goes offline.** A human has to notice, SSH into the server, restart the container. Customers are waiting.

With **Kubernetes**, your container dies. **K8s automatically restarts it in 5 seconds.** Customers don't notice.

Now imagine it's Friday, 2x traffic. You need to scale from 2 to 6 API servers.

With Compose: manually create 6 containers, configure load balancer, pray nothing breaks.

With K8s: `kubectl scale deployment api --replicas=6`. Done. 10 seconds.

Now imagine you need to update your image (new model, better performance).

With Compose: drain traffic, restart each container, hope nothing goes wrong.

With K8s: rolling update. Gradually replaces old containers with new ones. If anything goes wrong, automatic rollback.

**This is why K8s is industry standard: it handles operational complexity automatically.**

Every company running production AI uses K8s or something equivalent (Docker Swarm, Nomad). You need this skill.

---

## 2. Conceptual Explanation: Plain English

### The Core Problem K8s Solves

You built an AI system. It works. Now you need to **run it reliably on many servers.**

Challenges:
- Servers fail. Network partitions happen. You need automatic recovery.
- Traffic varies. You need to scale up and down.
- You need to update code without downtime.
- You need to monitor everything and alert on problems.
- You need this to work the same on-prem, on AWS, on GCP.

**K8s is the OS for running distributed systems.**

### Core Concepts (with Analogies)

**Cluster:** A collection of servers (nodes) managed as a single unit.

*Analogy: A restaurant chain — individual locations (nodes) managed by headquarters (control plane).*

**Node:** An individual server (physical or virtual) in the cluster.

*Analogy: Individual restaurant location.*

**Pod:** The smallest unit of deployment in K8s. Usually one container, sometimes 2-3 tightly coupled containers.

*Analogy: A dish (container) plus garnish (sidecar containers). You can't split a dish across restaurants.*

**Deployment:** "Run N replicas of this pod, and keep them running."

```
Deployment: "I want 3 copies of my FastAPI server always running"
         ↓
     │ Pod │  │ Pod │  │ Pod │
     │ API │  │ API │  │ API │
```

*Analogy: A head chef saying "I want 3 line cooks at station 1 all the time." If one leaves, hire another.*

**Service:** A stable, durable address to reach a set of pods.

Individual pods come and go. If Pod 1 dies and gets replaced with Pod 4, the address shouldn't change.

A Service is like a restaurant's phone number. The chef in the kitchen might change, but the phone number stays the same.

```
      Client calls: restaurant.com
           ↓
      Service (phone number)
           ↓
        ┌──┴──┬──────┬──────┐
        │     │      │      │
      Pod 1 Pod 2  Pod 3  Pod 4
```

**Ingress:** Routes external traffic to the right Service based on URL path.

```
example.com/chat     → Chat API Service
example.com/search   → Search API Service
example.com/health   → Health Check Service
```

*Analogy: Hotel front desk routing guests to correct room.*

**ConfigMap:** A configuration file that lives in K8s, injected into pods as environment variables or files.

*Analogy: A printed name placard you hand to each employee.*

**Secret:** Same as ConfigMap, but encrypted. For sensitive data (API keys, passwords).

**Namespace:** A virtual cluster within a cluster. For organization and access control.

*Analogy: Separate subdivisions of the restaurant (front-of-house, kitchen, management have separate tools).*

**PersistentVolume (PV) + PersistentVolumeClaim (PVC):** Storage that outlives a pod.

When a pod dies, its local storage dies with it. But if your database needs persistent storage, you use a PVC. K8s finds or creates a PV for you.

**StatefulSet:** Like Deployment, but for stateful services (databases, caches).

- Pods have stable, predictable identities (postgres-0, postgres-1, postgres-2)
- Pods scale up/down in order
- Each pod has its own storage volume

*Analogy: Deployment is like hiring line cooks (interchangeable). StatefulSet is like CEO succession (specific ordered roles).*

---

## 3. Code Examples (Complete YAML)

### Pod YAML (Educational, Not Used in Production)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-ai
  namespace: default
spec:
  containers:
  - name: api
    image: myregistry.azurecr.io/ai-api:latest
    ports:
    - containerPort: 8000
    env:
    - name: MODEL_NAME
      value: "gpt-3.5"
    - name: LOG_LEVEL
      value: "INFO"
    resources:
      requests:
        memory: "256Mi"      # guaranteed
        cpu: "500m"          # 0.5 cores
      limits:
        memory: "512Mi"      # max allowed
        cpu: "1000m"         # max 1 core
```

**Why we don't use this in production:** Pods are ephemeral. Dies without restart. Use Deployment instead.

### Deployment YAML (Production Pattern)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-api
  namespace: default
spec:
  replicas: 3                    # Run 3 copies
  strategy:
    type: RollingUpdate          # Gradual update, no downtime
    rollingUpdate:
      maxSurge: 1                # max 4 pods during update
      maxUnavailable: 0          # never drop below 3
  selector:
    matchLabels:
      app: ai-api               # This matches label below
  template:
    metadata:
      labels:
        app: ai-api             # Pod label
    spec:
      containers:
      - name: api
        image: myregistry/ai-api:v1.2.3
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8000
        
        # Load config from ConfigMap
        envFrom:
        - configMapRef:
            name: ai-config
        
        # Load secrets from Secret
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-secrets
              key: openai-key
        
        # Health checks
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30    # wait 30s for model load
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3        # fail after 3 timeouts
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 3        # fail and restart after 3 timeouts
        
        # Resource management
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        # Write logs to stdout (collected by K8s)
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      
      # Pod placement
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
              topologyKey: kubernetes.io/hostname  # spread across nodes
      
      volumes:
      - name: tmp
        emptyDir: {}              # temporary storage per pod
```

### Service YAML (ClusterIP, for Internal Routing)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-api
spec:
  type: ClusterIP              # internal IP only
  selector:
    app: ai-api                # Route to pods with this label
  ports:
  - name: http
    port: 80                   # external port
    targetPort: 8000           # pod port
    protocol: TCP
```

Within K8s, other pods reach this service at: `http://ai-api:80`

### Service YAML (LoadBalancer, for External Traffic)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-api-external
spec:
  type: LoadBalancer           # AWS creates an NLB
  selector:
    app: ai-api
  ports:
  - port: 80
    targetPort: 8000
```

AWS automatically provisions a Network Load Balancer with a public IP.

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-config
data:
  MODEL_NAME: "gpt-4"
  TEMPERATURE: "0.7"
  MAX_TOKENS: "2048"
  CHUNK_SIZE: "512"
  LOG_LEVEL: "INFO"
  ENVIRONMENT: "production"
```

### Secret YAML (base64 encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-secrets
type: Opaque
data:
  # base64 encode: echo -n "sk-..." | base64
  openai-key: c2stYW50aHJvcGljLWhoaWtwZXJ0aWZpeDAwMDAwMDAwMDAw
  db-password: cG9zdGdyZXNwYXNzd29yZDEyMw==
```

**Create from literal (safer than YAML):**
```bash
kubectl create secret generic ai-secrets \
  --from-literal=openai-key=sk-... \
  --from-literal=db-password=postgres123
```

### Namespace YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

Run stuff in a namespace:
```bash
kubectl apply -f deployment.yaml -n production
```

### StatefulSet YAML (for Stateful Services like Databases)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
spec:
  serviceName: qdrant-headless    # headless service (no ClusterIP)
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
        volumeMounts:
        - name: qdrant-storage
          mountPath: /qdrant/storage
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  
  # Create persistent volume for each replica
  volumeClaimTemplates:
  - metadata:
      name: qdrant-storage
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: fast-ssd   # AWS gp3
      resources:
        requests:
          storage: 50Gi
```

Pods are named: qdrant-0, qdrant-1, qdrant-2 (stable identities).

---

## 4. Architecture Diagram

```
┌────────────────────────────────────────────────────────────┐
│               Kubernetes Cluster                            │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  Control Plane (1 master node)                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ API Server   Scheduler   Controller   etcd DB        │  │
│  │ (REST API)   (assign pods)(desired state)(storage)   │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│    ┌─────────────────────┼─────────────────────┐           │
│    │                     │                     │           │
│  Node 1 (4CPU, 16GB)   Node 2 (4CPU, 16GB)   Node 3   │
│  ┌──────────────┐      ┌──────────────┐      ┌─────────┐ │
│  │ kubelet      │      │ kubelet      │      │ kubelet │ │
│  │ (node agent) │      │ (node agent) │      │         │ │
│  │              │      │              │      │         │ │
│  │ ┌──────────┐ │      │ ┌──────────┐ │      │ ┌─────┐ │ │
│  │ │Pod API 1 │ │      │ │Pod API 2 │ │      │ │Pod  │ │ │
│  │ │(FastAPI) │ │      │ │(FastAPI) │ │      │ │Data │ │ │
│  │ └──────────┘ │      │ └──────────┘ │      │ │     │ │ │
│  │              │      │              │      │ └─────┘ │ │
│  │ ┌──────────┐ │      │ ┌──────────┐ │      │         │ │
│  │ │Pod Cache │ │      │ │Pod Cache │ │      │         │ │
│  │ │(Redis)   │ │      │ │(Redis)   │ │      │         │ │
│  │ └──────────┘ │      │ └──────────┘ │      │         │ │
│  └──────────────┘      └──────────────┘      └─────────┘ │
│                                                              │
│  Services (stable addresses)                               │
│  ├─ api (ClusterIP) → pods with app=ai-api               │
│  ├─ cache (ClusterIP) → pods with app=redis              │
│  └─ api-external (LoadBalancer) → AWS NLB → pods         │
│                                                              │
│  Ingress (external routing)                               │
│  /chat   → api Service                                     │
│  /search → search Service                                  │
│  /health → health Service                                  │
└────────────────────────────────────────────────────────────┘
```

---

## 5. Tools and Alternatives

| Tool | Purpose | Comparison |
|------|---------|-----------|
| **kubectl** | CLI for managing K8s | Standard. Use this always. |
| **Helm** | Package manager for K8s | Templating YAML. Essential for production. |
| **kube-ops-view** | Web UI for cluster visualization | Better than default dashboard. |
| **Lens** | IDE for K8s (free or enterprise) | IDE-like interface, very user-friendly. |
| **Minikube** | Local K8s for development | Run K8s on your laptop. Essential for learning. |
| **Kind** | Lightweight K8s in Docker | Faster than Minikube, fewer features. |
| **Docker Swarm** | Alternative to K8s (simpler) | Easier to learn, scales to ~10K containers. After that, use K8s. |
| **AWS ECS** | AWS-specific container orchestration | Easier on AWS, less portable than K8s. |
| **Nomad** | By HashiCorp, general orchestration | More flexible than K8s, less mature for Kubernetes workloads. |

---

## 6. Decision Framework

**Should you use Kubernetes?**

✅ **Use K8s if:**
- You have 3+ microservices
- You need auto-scaling
- You have 5+ engineers managing infrastructure
- You want to run on multiple cloud providers

❌ **Don't use K8s if:**
- You have a single service running on 1-2 servers (use bare VMs or ECS)
- Your infrastructure team is 1-2 people (overhead not worth it)
- Your traffic is highly predictable (no auto-scaling needed)

**For AI engineers:** Almost always use K8s in production roles. If a company doesn't use K8s, they're either very small or very specialized. Learning K8s is a safe bet.

**Minikube vs EKS:**
- **Minikube:** Local development. Free. Single-node.
- **EKS:** Production on AWS. Managed (AWS updates for you). Multi-node recommended.

Start with Minikube (today), then deploy to EKS (week 3).

---

## 7. Step-by-Step Mini Tutorial: Minikube Setup

### Install Minikube

```bash
# macOS
brew install minikube

# Linux
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Windows
choco install minikube
```

### Start Minikube

```bash
# Start with 4 CPUs and 8GB RAM (adjust to your machine)
minikube start --cpus=4 --memory=8192

# Should print: minikube has been successfully configured
```

### Configure kubectl

```bash
# Minikube sets up kubectl automatically
kubectl config current-context
# Should print: minikube

# Verify cluster is running
kubectl get nodes
# Should show: minikube   Ready    master,worker

# Verify cluster info
kubectl cluster-info
# Should show: Kubernetes master is running...
```

### Deploy a Simple Container

```bash
# Create namespace
kubectl create namespace dev

# Deploy
kubectl create deployment hello \
  --image=nginx:latest \
  --replicas=2 \
  -n dev

# Verify
kubectl get pods -n dev
# Should show 2 running nginx pods

# Get more details
kubectl describe pod <pod-name> -n dev
```

### Expose with Service

```bash
# Create a service
kubectl expose deployment hello \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80 \
  -n dev

# Get service info
kubectl get svc -n dev
# Note the EXTERNAL-IP

# On Minikube, use minikube service to access
minikube service hello -n dev
# Opens browser to the service

# Alternatively, port-forward
kubectl port-forward svc/hello 8080:80 -n dev
# Now access: http://localhost:8080
```

### View Logs

```bash
# Logs from a pod
kubectl logs <pod-name> -n dev

# Follow logs (like tail -f)
kubectl logs -f <pod-name> -n dev

# Logs from all pods in deployment
kubectl logs -f deployment/hello -n dev
```

### Execute Commands in Pod

```bash
# Run bash in a pod (if container has bash)
kubectl exec -it <pod-name> -n dev -- /bin/bash

# Run a single command
kubectl exec <pod-name> -n dev -- curl localhost:8000/health
```

### Rolling Update

```bash
# Update image
kubectl set image deployment/hello \
  hello=nginx:1.21 \
  -n dev

# Watch rollout
kubectl rollout status deployment/hello -n dev

# See history
kubectl rollout history deployment/hello -n dev

# Rollback if something goes wrong
kubectl rollout undo deployment/hello -n dev
```

### Cleanup

```bash
# Delete deployment
kubectl delete deployment hello -n dev

# Delete service
kubectl delete svc hello -n dev

# Delete namespace
kubectl delete namespace dev

# Stop Minikube
minikube stop
```

---

## 8. Practical Project: Deploy Your AI API to Minikube

**Objective:** Deploy your Phase 1-2 FastAPI + Qdrant setup to Minikube with proper health checks, resource limits, and service exposure.

**Deliverables:**
1. Dockerfile for your FastAPI (optimized)
2. ConfigMap for model and chunking config
3. Secret for API keys
4. Deployment YAML for API (with health checks, resources)
5. StatefulSet YAML for Qdrant
6. Service YAML to expose both internally
7. Script to deploy, verify, and test

**Steps:**
1. Create Dockerfile (multi-stage, non-root user)
2. Build and push to local registry (or minikube docker-env)
3. Create ConfigMap and Secret for configuration
4. Write Deployment YAML with resource limits
5. Write StatefulSet YAML for Qdrant with PVC
6. Create Service for both
7. kubectl apply -f all YAMLs
8. Test: port-forward and make API calls
9. Scale: kubectl scale deployment api --replicas=5
10. Verify all 5 pods are running and healthy

**Success Criteria:**
- [ ] API pod starts and reports healthy
- [ ] Qdrant pod starts and has persistent storage
- [ ] Pods can reach each other by hostname
- [ ] Health check endpoint responds with 200
- [ ] Scaling to 5 replicas works
- [ ] Logs are accessible via kubectl logs
- [ ] Graceful shutdown on pod deletion

---

## 9. Debugging Playbook: 10 Common K8s Errors

### 1. **ImagePullBackOff**
```
kubectl describe pod api-123
→ Failed to pull image "myregistry/api:v1.0": "unknown: tag not found"
```

**Root cause:** Image doesn't exist in registry or tag is wrong.

**Fix:**
```bash
# Check image exists
aws ecr describe-images --repository-name ai-api

# Push correct image
docker build -t myregistry/ai-api:v1.0 .
docker push myregistry/ai-api:v1.0

# Or fix YAML
kubectl set image deployment/api \
  api=myregistry/ai-api:v1.0.1
```

### 2. **CrashLoopBackOff**
```
kubectl logs api-123
→ ModuleNotFoundError: No module named 'torch'
```

**Root cause:** Missing dependencyinstalled in container.

**Fix:**
```dockerfile
# Add to Dockerfile
RUN pip install torch transformers
```

Re-build, push, then update:
```bash
kubectl rollout restart deployment/api
```

### 3. **Pod Pending**
```
kubectl describe pod api-123
→ 0/3 nodes available: 3 Insufficient memory
```

**Root cause:** Pod requests more memory than available on any node.

**Fix:**
```yaml
resources:
  requests:
    memory: "512Mi"  # lower from 2Gi
```

Or add more nodes to cluster.

### 4. **ReadinessProbe Failing**
```
kubectl describe pod api-123
→ Probe failed: HTTP request returned code 502
```

**Root cause:** Health check endpoint not responding. API might be loading model.

**Fix:**
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 60  # wait longer for model load
  periodSeconds: 10
```

### 5. **Service Addresses Not Resolving**
```
curl api:80
→ curl: (6) Could not resolve host: api
```

**Root cause:** Service or pods not created, or DNS not working in pod.

**Fix:**
```bash
# Check service exists
kubectl get svc

# Check pod can reach DNS
kubectl exec <pod> -- nslookup api
# Should resolve to service IP

# Restart coredns if broken
kubectl rollout restart deployment coredns -n kube-system
```

### 6. **OOMKilled (Out of Memory)**
```
kubectl describe pod api-123
→ Reason: OOMKilled
→ Limits: 512Mi memory
```

**Root cause:** Container using more memory than limit.

**Fix:**
```yaml
resources:
  limits:
    memory: "2Gi"  # increase from 512Mi
```

Or optimize code (load model in parallel, use streaming for large files).

### 7. **Port Already in Use**
```
kubectl describe pod api-123
→ bind: address already in use
```

**Root cause:** Port 8000 already taken on node.

**Fix:**
```yaml
ports:
- containerPort: 8000
  hostPort: 9000  # use different port on host
```

Or use ClusterIP service (no hostPort), then port-forward:
```bash
kubectl port-forward svc/api 8000:8000
```

### 8. **Pod scheduling to wrong node**
```
# Pod stuck on overloaded node instead of spreading
kubectl get pods -o wide
→ all 5 api pods on node-1 (leaving node-2 empty)
```

**Root cause:** No anti-affinity rules.

**Fix:**
```yaml
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
            - api
        topologyKey: kubernetes.io/hostname
```

### 9. **Secret Mount Failing**
```
kubectl exec <pod> -- env | grep OPENAI
→ (empty — secret not mounted)
```

**Root cause:** Secret name wrong or doesn't exist.

**Fix:**
```bash
# Check secret exists
kubectl get secret openai-key

# Check secret value (base64 decoded)
kubectl get secret openai-key -o jsonpath='{.data.key}' | base64 -d

# Fix YAML
env:
- name: OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: openai-key      # must match secret name
      key: key              # must match secret data key
```

### 10. **StatefulSet pod not starting with ordered startup**
```
# All 3 pods started simultaneously instead of ordered
kubectl get pods -o wide
→ qdrant-0, qdrant-1, qdrant-2 all starting (not waiting for -0 first)
```

**Root cause:** Using Deployment for stateful service (should use StatefulSet).

**Fix:**
```yaml
kind: StatefulSet        # not Deployment
serviceName: qdrant-headless
```

StatefulSet ensures qdrant-0 fully runs before qdrant-1 starts.

---

## 10. Common Mistakes (5+ Issues)

!!! danger "Mistake 1: Requests Too High, Pod Never Starts"
    ```yaml
    resources:
      requests:
        memory: "100Gi"     # ❌ Your node doesn't have 100Gi
    ```
    Pod stays Pending forever. Always check `kubectl top nodes` before setting requests.

!!! danger "Mistake 2: No Health Checks = Crashed Pod Still Gets Traffic"
    ```yaml
    # ❌ No readinessProbe or livenessProbe
    containers:
    - name: api
      image: myapi
    ```
    Pod crashes during startup. Requests still routed to it. Set both probes.

!!! danger "Mistake 3: Image Tag 'latest' = Random Deployments"
    ```yaml
    image: myregistry/api:latest  # ❌ Tag keeps changing
    ```
    You deploy. Days later, 'latest' has a bug. Pods restart with buggy version. Always use explicit version tags.

!!! danger "Mistake 4: Hardcoded API Keys in Dockerfile"
    ```dockerfile
    ENV OPENAI_API_KEY=sk-...    # ❌ Leaks secret in image layers
    ```
    Anyone with Docker access sees your keys. Use Secrets, not ENV in Dockerfile.

!!! danger "Mistake 5: No Resource Limits = Noisy Neighbor"
    ```yaml
    # ❌ No limits specified
    containers:
    - name: api
    ```
    Pod eats all CPU and memory. Other pods get throttled. Always set limits.

---

## 11. Production Realism

**The Truth about K8s:**

✅ Automatically handles pod restart, scaling, rolling updates, multi-node orchestration.

❌ But:
- Steep learning curve (lots of new concepts)
- Debugging is trickier than single-machine debugging
- K8s has its own failure modes (control plane issues, etcd corruption, network partition)
- "simple" setups take 5x longer to set up right
- You need monitoring and alerting from day 1 (see Phase 4 Scaling)

**What "works" locally vs production:**
- Minikube works for single-node development
- EKS needed for production (managed control plane, multiple nodes, HA)
- Your local YAML might fail on EKS due to: storage classes, ingress differences, networking

**Operational reality:**
- Day 1: kubectl apply, things work
- Week 2: Pod crashes at 3am. You have to understand why.
- Month 2: Data corruption because persistent volume filled up
- Month 3: Network partition between nodes, services unreachable

You need observability (Phase 4 Scaling file).

---

## 12. Cost & Performance Considerations

**Compute costs:**

On AWS EKS:
- Control plane: $0.10 per hour (fixed)
- Nodes: t3.xlarge (4 CPU, 16GB) = $0.1664/hour
- For 3 nodes: $0.50/hour (always running) = $360/month

vs EC2 auto-scaling:
- Same 3 t3.xlarge: similar cost
- But EKS adds overhead (~10-15% more for managed service)

**Storage costs:**

- gp3 EBS volume: $0.10 per GB/month
- 100GB storage: $10/month
- Qdrant with 500GB: $50/month

**Optimization:**
- Use Spot instances for batch jobs (70% discount, but can be interrupted)
- Scale down to 1 node at night if load is predictable
- Use Reserved Instances for baseline (30% discount)

**Performance:**

- Pod startup latency: 2-5 seconds (network + container init)
- For real-time inference: pre-warm pods (minReplicas=2, don't scale to zero)
- Network latency between pods: <1ms (same node) to 10-50ms (cross-zone)

---

## 13. Security Considerations

**Secrets Management:**

❌ **Don't:**
```yaml
env:
- name: OPENAI_API_KEY
  value: "sk-..."
```

✅ **Do:**
```yaml
env:
- name: OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: ai-secrets
      key: openai-key
```

**Network Security:**

Use NetworkPolicy to restrict traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432
```

**RBAC (Role-Based Access Control):**

Don't give everyone admin access.

```bash
# Create limited serviceaccount
kubectl create serviceaccount deployer -n production

# Create role (can only deploy to production)
kubectl create role deployer \
  --verb=create,get,list,update \
  --resource=deployments,services \
  -n production

# Bind role to serviceaccount
kubectl create rolebinding deployer \
  --role=deployer \
  --serviceaccount=production:deployer \
  -n production
```

---

## 14. Case Studies

### Netflix: K8s at Massive Scale

Netflix runs **thousands of microservices** in K8s. Problems they solved:

- **Traffic surges:** On premiere nights, 10x baseline traffic. K8s auto-scales in seconds.
- **Canary deployments:** 1% of traffic to new image, if OK then full rollout. Zero-downtime updates.
- **Failure resilience:** Single container dies? Feature pod crashes? K8s kills it and starts a new one.
- **Cost optimization:** Mix on-demand + spot instances. Save 40% on infrastructure.

**Result:** 99.99% uptime for recommendation engine.

### Stripe: ML Models in K8s

Stripe runs fraud detection models in K8s:

- **Model versioning:** Each model release versioned, easy rollback.
- **A/B testing:** Deploy new model as Deployment, route 10% traffic via Ingress, measure.
- **Replicas:** During payment surges, auto-scale fraud model replicas from 2 to 20.
- **Stability:** Pod crash doesn't drop requests, new pod starts in <5s.

**Result:** Fraud caught improved from 92% to 98% with model improvements.

---

## 15. Interview Cheat Sheet

**5 Core Concepts to Know:**

1. **Pod:** Smallest K8s unit. One or more containers. Ephemeral (dies, gets replaced).

2. **Deployment:** Says "run 3 replicas of this pod always." K8s ensures that happens. Rolling updates with zero downtime.

3. **Service:** Stable IP address for reaching a set of pods. Pods come and go, service stays.

4. **ConfigMap + Secret:** How to get config and sensitive data to pods. ConfigMap is plaintext, Secret is encrypted.

5. **Persistent Volume:** Storage that outlives pod. Required for databases and stateful services.

**Interview question likely to be asked:**
- "Your pod crashed, how would you debug?"
  Answer: `kubectl describe pod [name]` to see status, `kubectl logs [name]` to see errors.

- "How would you ensure 10 API servers are always running?"
  Answer: Deployment with 10 replicas. K8s ensures that count automatically.

---

## 16. Review Questions

1. What's the difference between a Pod and a Deployment?
2. Why do you need a Service if you have a Deployment?
3. What does `kubectl rollout undo` do and when would you use it?
4. Explain readinessProbe and livenessProbe. Why do you need both?
5. What's the difference between ConfigMap and Secret?
6. How do you scale a Deployment from 2 to 10 replicas?
7. Explain StatefulSet vs Deployment. When would you use each?
8. How do you debug a Pod that's stuck in "Pending" state?
9. What's a Namespace and why would you use multiple namespaces?
10. Explain PersistentVolume and PersistentVolumeClaim.

---

## 17. Flashcards

**Q: What is a Pod?**
A: The smallest deployable unit in K8s. One or more containers that share network namespace (same IP).

**Q: What is a Deployment?**
A: Desired state declaration. Says "run N replicas, update gradually, don't lose traffic during updates."

**Q: What's the difference between Service and Ingress?**
A: Service gives a stable IP for pods. Ingress routes external traffic by URL path to multiple services.

**Q: What does `initialDelaySeconds` do in readinessProbe?**
A: Waits that many seconds before starting health checks. Gives pod time to load model.

**Q: What's in a Secret?**
A: Encrypted configuration data. API keys, passwords, certificates. Never in environment variables.

**Q: When would you use StatefulSet instead of Deployment?**
A: For stateful services (databases, caches) that need stable Pod names and persistent storage.

**Q: What happens when a Pod crashes?**
A: If it's in a Deployment, K8s detects it and starts a new Pod with the same configuration.

**Q: How do you update an image without downtime?**
A: Deployment rolling update. Gradually replaces old pods with new image. See rollout status.

**Q: What does `limits` mean in resources?**
A: Maximum CPU/memory the pod can use. Exceed it and pod gets killed (OOMKilled, throttled).

**Q: What's a Namespace?**
A: Virtual cluster within cluster. Isolates resources and RBAC. `kubectl apply -f file.yaml -n production`.

---

## 18. Teach-It-Back Prompt

"Explain Kubernetes to someone who knows Docker but not K8s. Use an analogy. Why is Kubernetes better than Docker Compose for production?"

**Model Answer:**
"Docker Compose is like managing a restaurant. You say: run 3 cooks, 2 dishwashers, 1 manager. If a cook quits, the restaurant closes.

Kubernetes is like having an HR manager watching the restaurant. You say: 'I want 3 cooks always working.' If a cook collapses, HR hires a replacement automatically. If customer surge happens (10x), HR hires more cooks. If sales drop, HR reduces staff. The restaurant never closes.

K8s advantages:
- Automatic recovery: crashed pod restarts
- Auto-scaling: more traffic = more replicas
- Rolling updates: update code without downtime
- Multi-node: spread across many servers
- Works anywhere: AWS, GCP, Azure, on-prem

Docker Compose is great for development. K8s is for production at scale."

---

## 19. 1-Week Review Checklist

- [ ] Understand core concepts: Pod, Deployment, Service, ConfigMap, Secret
- [ ] Installed Minikube and started cluster
- [ ] Deployed a simple Nginx container
- [ ] Created Deployment YAML from scratch
- [ ] Exposed a service and accessed it locally
- [ ] Scaled a deployment up and down
- [ ] Performed rolling update and rollback
- [ ] Debugged a failing pod (describe, logs, exec)
- [ ] Created ConfigMap and injected into pod
- [ ] Created Secret and injected into pod

---

## 20. Resources

- **Kubernetes Official Docs:** https://kubernetes.io/docs/
- **Minikube:** https://minikube.sigs.k8s.io/docs/
- **kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **K8s by Example:** https://kubernetesbyexample.com/
- **Video Tutorial:** https://www.youtube.com/watch?v=X48VuDVv0do (4 hours, comprehensive)
- **Interactive Tutorial:** https://katacoda.com/kubernetes (hands-on practice)

---

## 21. Next: AI on Kubernetes

You now understand K8s fundamentals.

Next: **How to deploy your AI stack (FastAPI + Qdrant + PostgreSQL) on K8s.**

→ **[AI on Kubernetes →](./ai-on-k8s.md)**
