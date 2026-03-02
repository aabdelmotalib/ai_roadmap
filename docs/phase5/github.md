# GitHub Portfolio: Getting Hired on Code

You have 8 amazing projects from this roadmap. Now make them portfolio-worthy.

Recruiters spend 90 seconds on your GitHub. In that time:
- They scan your pinned repos
- Check if README exists
- Look at recent commits
- Judge code quality

Your job: **make it impossible to miss.**

---

## The 8 Projects to Feature (in Order)

### 1. **RAG System with Evaluation** (Phase 1)
**What it shows:** You understand embeddings, vector DBs, retrieval quality evaluation (RAGAS).
**Who cares:** Every AI company needs RAG.
**Title:** `rag-evaluator` or `semantic-search-system`

### 2. **Fine-Tuned Model with Metrics** (Phase 1)
**What it shows:** You can train, evaluate, and iterate on models.
**Title:** `lora-finetuning-pipeline`

### 3. **Agent with Tools** (Phase 1)
**What it shows:** You can orchestrate LLM agent loops, handle tool use.
**Title:** `llm-agent-framework`

### 4. **FastAPI Full Stack Service** (Phase 2)
**What it shows:** Production API design, health checks, error handling.
**Title:** `ai-api-server`

### 5. **Docker + AWS Deployment** (Phase 2)
**What it shows:** Containerization and cloud deployment.
**Title:** `containerized-ai-service` or `ai-on-aws`

### 6. **MLOps Pipeline** (Phase 3)
**What it shows:** Experiment tracking, data versioning, automation (the hardest part).
**Title:** `mlops-training-pipeline` or `ml-orchestration`

### 7. **Kubernetes Deployment** (Phase 4)
**What it shows:** Production ops, autoscaling, monitoring.
**Title:** `k8s-ai-platform` or `kubernetes-setup`

### 8. **Full Stack Demo** (Integration)
**What it shows:** Everything together: AI + API + deployment + monitoring.
**Title:** `full-stack-ai-system` or `ai-platform-end-to-end`

---

## README Template: The Secret Weapon

Your README is the entry point. Make it **undeniable**.

### Perfect README Structure

```markdown
# Project Title: One-Line Value Statement

## 🎯 Problem

What problem does this solve? Be specific.
✓ "Semantic search over 500+ documents in <100ms with 95% relevance"
✗ "Search system"

## 📺 Demo

[GIF or Loom link showing it working]

```html
<a href="https://www.loom.com/share/xyz">
  <p>Click to watch 2-minute demo</p>
</a>
```

(More details below on recording)

## 🏗️ Architecture

Mermaid diagram showing: Input → Components → Output

```
User Query
  ↓
[Embedding Model]
  ↓
[Vector Search (Qdrant)]
  ↓
[LLM + RAG]
  ↓
Response with Sources
```

## 📦 Tech Stack

Badges (shields.io):
```
![Python](https://img.shields.io/badge/Python-3.11-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.95-green)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-335e99)
![Docker](https://img.shields.io/badge/Docker-24-2496ed)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27-326ce5)
![Tests Passing](https://img.shields.io/badge/tests-passing-brightgreen)
```

## 🚀 Features

- ✅ Semantic search with vector embeddings
- ✅ FastAPI REST API with authentication
- ✅ PostgreSQL for metadata
- ✅ Qdrant for vector storage
- ✅ Docker containerization
- ✅ Kubernetes deployment (3-10 replicas HPA)
- ✅ Prometheus metric

s + Grafana monitoring
- ✅ Unit tests (pytest)
- ✅ Load tested (1000 req/sec)

## 📊 Results & Metrics

**Performance:**
- Search latency: 45ms p99 (target: <100ms) ✅
- Throughput: 2000 requests/second ✅
- Uptime: 99.9% across 3 deployments ✅

**Quality:**
- RAGAS Faithfulness: 0.87 (target: >0.75) ✅
- Exact match rate: 92% ✅
- False positive rate: <2% ✅

**Infrastructure:**
- Cost: $120/month (3x t3.xlarge + storage) ✅
- Scaling: auto-scales 3→20 pods under load ✅

## 🛠️ Setup (2 Commands)

```bash
# Install dependencies
pip install -r requirements.txt

# Run locally (dev mode)
docker-compose up

# Visit http://localhost:8000/docs (Swagger UI)
```

**Then test:**
```bash
curl -X POST http://localhost:8000/search \
  -json '{"query": "how does RAG work?"}' \
  | jq .
```

## 📚 API Documentation

### POST /search

Search semantic database.

**Request:**
```json
{
  "query": "how does semantic search work?",
  "top_k": 5,
  "score_threshold": 0.7
}
```

**Response:**
```json
{
  "results": [
    {
      "document": "...",
      "score": 0.95,
      "source": "docs/rag.md"
    }
  ],
  "latency_ms": 45
}
```

### GET /health

Health check (used by K8s).

**Response:**
```json
{
  "status": "ok",
  "models_loaded": true,
  "db_connected": true
}
```

## 🎓 Lessons Learned

3 specific technical lessons:

1. **Vector Similarity Pitfall:** L2 distance and cosine similarity give different rankings. We switched to cosine (normalized embeddings) because our queries and documents are short. Learned: understand your distance metric's assumptions.

2. **Chunking Strategy Matters:** 256-token chunks worked best for our domain (documentation). Longer chunks had lower precision, shorter had gaps. Evaluated 3 strategies, picked empirically based on RAGAS.

3. **Cold Start Problem:** First query is slow (model loading). Solution: container's readinessProbe waits 30s, healthcheck includes model load check. K8s only sends traffic when ready.

## 🔄 What's Next

- [ ] Fine-tune embeddings on domain-specific data (+5% RAGAS)
- [ ] Add reranking layer for top-5 results (better quality)
- [ ] Implement semantic caching (reduce cost, improve latency)
- [ ] Multi-language support (translate query → English → search)
- [ ] A/B testing pipeline for algorithm changes

## 📜 License

MIT

## 🤝 Contributing

Ideas? Issues? PRs welcome!

```
```

---

## Badges (Copy-Paste Friendly)

```markdown
![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square)
![FastAPI](https://img.shields.io/badge/FastAPI-0.95.0-009688?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-336791?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-20-2496ED?style=flat-square)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27-326CE5?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![Tests](https://img.shields.io/github/actions/workflow/status/user/repo/tests.yml?branch=main&label=Tests&style=flat-square)
![Coverage](https://img.shields.io/codecov/c/github/user/repo?style=flat-square)
```

---

## Recording a 2-Minute Demo (Loom)

### Script Template (90 seconds)

**0-10s: Problem**
"Here's the problem: searching through hundreds of documents is slow and irrelevant. I built a semantic search system."

**10-40s: Demo**
(Show the thing working without talking)
- User types query
- Results appear in <100ms
- Shows relevance scores, sources
- Maybe run 3 queries to show it works

**40-60s: Results**
"Key metrics: 45ms latency, 0.87 RAGAS faithfulness, <$200/month to run."

**60-90s: Closing**
"Built with FastAPI, Qdrant, and deployed to Kubernetes with auto-scaling. See the code on GitHub."

### Recording Tool

https://www.loom.com/ — free tier, 5 minute videos max.

**Pro tips:**
- Close background apps (increase focus on screen)
- Use fullscreen browser (remove address bar clutter)
- 1.5x zoom for readability
- Cursor trails on (shows what you're clicking)
- Background music: optional (makes it feel polished)

---

## Commit Message Conventions

Recruiters see this in your GitHub history. Make it professional.

```bash
# Conventional Commits format
git commit -m "feat: add semantic caching for 30% latency improvement"
git commit -m "fix: handle null embeddings in similarity search"
git commit -m "docs: add architecture diagram and API docs"
git commit -m "test: add RAGAS evaluation tests"
git commit -m "chore: upgrade numpy to 1.24.0"
```

Not:
```bash
git commit -m "fix stuff"  # ❌
git commit -m "asdf"      # ❌
```

---

## Pinned Repositories Strategy

Go to your GitHub profile, click "Customize your pins".

Pin:
1. **Your best project** (most stars/impressive)
2. **Your most complete project** (full stack)
3. **Your learning project** (shows T-shaped depth)
4. **1 open source contribution** (if you have one)

(Only show your 3-4 best. 8 projects is too many.)

---

## GitHub Profile README (Optional but Impressive)

Create a repo called `[your-username]/[your-username]` (it's special).

```markdown
### Hi, I'm [Your Name]

AI engineer building production systems.

#####  Skills

🔹 **AI/ML:** PyTorch, HuggingFace, MLflow, RAGAS evaluation
🔹 **Backend:** FastAPI, PostgreSQL, Redis, Qdrant
🔹 **DevOps:** Docker, Kubernetes, AWS EKS, GitHub Actions
🔹 **Data:** DVC, pandas, numpy, vector embeddings

#### Featured Projects

- [RAG Evaluator](github.com/you/rag-evaluator) — Semantic search with quality metrics
- [K8s AI Platform](github.com/you/k8s-ai) — Production deployment with auto-scaling
- [Fine-Tuning Pipeline](github.com/you/lora) — LoRA training with MLflow tracking

#### Current Focus

Building AI systems that are fast, cost-effective, and production-ready.

#### Contact

- Email: your@email.com
- LinkedIn: linkedin.com/in/you
- Twitter: @you
```

---

## GitHub Actions: The Hidden Powerful Tool

Recruiters *love* seeing CI/CD working.

### Add a Tests Badge

Create `.github/workflows/tests.yml`:

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - run: pip install -r requirements-dev.txt
    - run: pytest --cov=src tests/
    - run: flake8 src/
```

Then add to README:

```markdown
![Tests](https://github.com/you/repo/workflows/Tests/badge.svg)
```

Now every merge shows "Tests ✅" — signals code quality.

---

## What NOT to Pin

❌ Forks (clones of other projects)
❌ Tutorials you completed (doesn't show original work)
❌ Incomplete projects (abandon doesn't look good)
❌ Projects without READMEs (looks unfinished)
❌ Code with no comments/structure (looks messy)

---

## Projects You Already Have (From This Roadmap)

| Phase | Project | Portfolio Quality |
|-------|---------|------------------|
| 1 | RAG System | ⭐⭐⭐⭐⭐ (must feature) |
| 1 | Fine-tuning | ⭐⭐⭐⭐⭐ (must feature) |
| 1 | Agent Framework | ⭐⭐⭐⭐ |
| 2 | FastAPI Service | ⭐⭐⭐⭐⭐ (must feature) |
| 2 | Docker Compose | ⭐⭐⭐ (supplementary) |
| 3 | MLflow Tracking | ⭐⭐⭐⭐ (nice demo) |
| 3 | DVC Pipeline | ⭐⭐⭐⭐ |
| 3 | GitHub Actions | ⭐⭐⭐ (supplementary) |
| 4 | K8s Deployment | ⭐⭐⭐⭐⭐ (must feature) |
| 4 | Monitoring Stack | ⭐⭐⭐ (supplementary) |

**Recommendation:**
- Feature the ⭐⭐⭐⭐⭐ projects (pin 3-4 of them)
- Reference others in READMEs
- Create 1-2 **integration projects** combining multiple phases

---

## Integration Project: The Ultimate Demo

Create a "showcase" repo combining everything:
- Phase 1 AI system
- Phase 2 FastAPI deployment
- Phase 3 MLOps pipeline
- Phase 4 Kubernetes infrastructure

```
📁 full-stack-ai-system/
├── src/
│   ├── ai/          (Phase 1: embeddings, RAG, evaluation)
│   └── api/         (Phase 2: FastAPI service)
├── ml/
│   ├── train.py     (Phase 3: training with MLflow)
│   └── dvc.yaml     (Phase 3: data versioning)
├── k8s/
│   ├── api.yaml     (Phase 4: Kubernetes deployment)
│   └── monitoring/  (Phase 4: Prometheus + Grafana)
├── .github/
│   └── workflows/   (Phase 3: CI/CD automation)
├── docker-compose.yml   (Phase 2)
├── Dockerfile           (Phase 2)
└── README.md            (Comprehensive guide)
```

README: "Here's how I built, deployed, and operate an AI system at production scale."

Recruiters see this and think: "This person isn't a junior."

---

## The 1-Week Portfolio Polish

- [ ] Day 1: Audit your 8 projects (README, code quality, working state)
- [ ] Day 2: Improve READMEs (architecture, metrics, setup)
- [ ] Day 3: Record Loom demos (2 min each for top 3 projects)
- [ ] Day 4: Fix CI/CD (add test badges)
- [ ] Day 5: Pin best repos + update profile README
- [ ] Day 6: Commit message cleanup (rewrite bad ones)
- [ ] Day 7: Final review (ask a friend to check)

---

## 21. Next: Resume & LinkedIn

Your GitHub is your proof.

Next: **How to talk about it in your resume and LinkedIn.**

→ **[Resume & LinkedIn →](./resume.md)**
