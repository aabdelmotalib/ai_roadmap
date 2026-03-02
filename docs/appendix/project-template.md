# Project Showcase: Templates and Tools for Maximum Impact

Your portfolio projects are how recruiters evaluate you.

Not your resume. Not your LinkedIn.

Your actual working code.

This guide gives you templates that make your projects shine.

---

## 1. The Perfect GitHub README

This is what recruiters read for 30-90 seconds before deciding "interview" or "skip."

### Complete README Template

```markdown
# [Project Name]

> [One-line description that explains the problem, not the solution]

[![Tests](https://img.shields.io/badge/tests-passing-brightgreen)](link-to-ci)
[![Docker Image](https://img.shields.io/badge/docker-available-blue)](link-to-image)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue)](link-to-python)
[![FastAPI](https://img.shields.io/badge/framework-fastapi-009485)](https://fastapi.tiangolo.com)

[Animated GIF showing the project working, 10 seconds max]

## Problem

[2-3 sentences: What was broken or missing before this existed?]

Example: "Customer support teams spend 30% of time on repetitive questions. This costs $200K/year per team and scales poorly with volume."

## Solution

[2-3 sentences: How does this solve the problem? What's novel?]

Example: "This system uses semantic search + LLM reasoning to answer 85% of routine questions automatically, escalating only edge cases to humans. Saves $150K/year and 90% of response time."

## Architecture

```
[Mermaid diagram or ASCII art showing the system]

User Query
    ↓
    Embedding (384-D vector) — OpenAI API
    ↓
    Vector Search — Qdrant (k=5 most similar docs)
    ↓
    Reranking — Cohere Rerank (get top-2)
    ↓
    LLM Reasoning — Claude (generate answer)
    ↓
    Confidence Check — If score < 0.7, escalate to human
    ↓
    Response
```

## Tech Stack

| Layer | Technology | Why | Alternative |
|-------|-----------|-----|--------------|
| **Backend** | FastAPI | Async, fast, modern | Flask, Django |
| **Vector DB** | Qdrant | Native vector search | Pinecone, Weaviate |
| **LLM** | Claude 3 Sonnet | Good balance of speed/quality | GPT-4, Mistral |
| **Embedding** | OpenAI embedding-3-small | 1.5M dims, fast, cheap | Cohere, local BERT |
| **Database** | PostgreSQL | Reliable, handles documents + metadata | MongoDB |
| **Caching** | Redis | Low latency | Memcached |
| **Hosting** | Railway | Cheap, Docker-native | Render, Heroku|

## Features

- ✅ Semantic search over 100K documents (sub-second retrieval)
- ✅ Reranking for accuracy (top-2 docs provide 95% precision)
- ✅ Human escalation for low-confidence responses
- ✅ Admin dashboard to add/edit documents
- ✅ Usage metrics and analytics
- ✅ Rate limiting (100 req/min per user to prevent abuse)
- ✅ Async processing for batch operations
- ✅ Health checks and monitoring

## Evaluation Results

| Metric | Score | Target | Status |
|--------|-------|--------|--------|
| RAGAS Faithfulness | 0.87 | >0.85 | ✅ |
| RAGAS Relevance | 0.92 | >0.85 | ✅ |
| P99 Latency | 450ms | <500ms | ✅ |
| System Uptime | 99.7% | >99.5% | ✅ |
| Escalation Rate | 12% | <15% | ✅ |
| Cost per Query | $0.04 | <$0.05 | ✅ |

## Setup

### Requirements
- Python 3.10+
- Docker (optional)
- OpenAI API key
- Qdrant instance (local or cloud)

### Local Setup (2 commands)

```bash
# 1. Install dependencies
poetry install

# 2. Run
cp .env.example .env  # Edit with your keys
make serve
```

Server runs at `http://localhost:8000`

### Docker Setup

```bash
docker build -t support-ai:latest .
docker run --env-file .env -p 8000:8000 support-ai:latest
```

### With Qdrant

```bash
docker-compose up -d  # Spins up Qdrant + app
```

See `docker-compose.yml` for full config.

## API Documentation

### POST /query

Ask a question and get an answer.

**Request:**
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d {
    "query": "How do I reset my password?",
    "user_id": "user123"
  }
```

**Response:**
```json
{
  "answer": "Visit the login page and click 'Forgot Password'. Enter your email...",
  "confidence": 0.92,
  "sources": [
    {
      "title": "Password Reset Guide",
      "url": "https://docs.example.com/password-reset",
      "similarity": 0.95
    }
  ],
  "escalated": false,
  "latency_ms": 234
}
```

### GET /health

Health check endpoint.

```bash
curl http://localhost:8000/health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "qdrant_connection": "ok",
  "db_connection": "ok"
}
```

### POST /admin/documents

Add documents to the knowledge base.

```bash
curl -X POST http://localhost:8000/admin/documents \
  -H "Authorization: Bearer YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d {
    "title": "Password Reset Guide",
    "content": "...",
    "url": "https://docs..."
  }
```

Full API docs: `http://localhost:8000/docs`

## Key Technical Decisions

### 1. Semantic Search + Reranking (not just keyword search)

**Decision:** Use embedding similarity (Qdrant) with Cohere reranking

**Tradeoff:** Slower (600ms → 450ms with reranking) but more accurate (87% → 92% faithfulness)

**Why:** Keyword search on "reset password" doesn't find "recovering account access" even though they're the same. Reranking ensures top results are actually relevant.

**Alternative we considered:** Fine-tune embedding model on support tickets. More accurate but 3x cost and slower iteration.

### 2. Claude 3 Sonnet (not GPT-4)

**Decision:** Claude Sonnet instead of GPT-4 for generation

**Tradeoff:** Slightly lower quality (GPT-4 0.89 vs Sonnet 0.87 faithfulness) but 3x cheaper and faster

**Why:** For support answers, 0.87 is sufficient. The cost/latency wins matter for production.

**Alternative we considered:** Mixtral or Mistral locally. Cheaper but requires GPU management and poorer quality on edge cases.

### 3. PostgreSQL + Qdrant (two databases)

**Decision:** PostgreSQL for documents + metadata, Qdrant for vectors

**Tradeoff:** Two systems to maintain vs simpler single-system design

**Why:** Qdrant is great at vector search, terrible at full-text search and filtering. PostgreSQL is the opposite. This design plays to each system's strength. Hybrid approach wins overall.

**Alternative:** Postgres with pgvector. Works, but slower vector operations (100ms vs 20ms in Qdrant).

## Lessons Learned

### 1. Embedding Model Choice Has HUGE Impact

I spent days on reranking, but switching embeddings from `text-embedding-3-small` to `small` saved 40ms latency with BETTER quality because the smaller model was properly optimized for our document lengths.

Lesson: Test before optimizing. The obvious big workload sometimes isn't the bottleneck.

### 2. Confidence Thresholds Are More Important Than Model Quality

We spent 2 weeks improving faithfulness from 0.85 → 0.87 (1.5% improvement).

Setting escalation threshold to 0.7 instead of 0.8 had 10x more impact on user experience (fewer escalations, fewer wrong answers).

Lesson: System design (when to escalate, when to return) matters as much as component quality.

### 3. Reranking Needs Ground Truth

Our first reranker (generic Cohere model) got 65% of ordering correct.

Fine-tuning a reranker on 500 support queries got us to 92% correct ordering. The problem: you need labeled data from YOUR domain.

Lesson: Off-the-shelf models work for 80% of cases. The last 20% needs domain-specific tuning.

## Future Improvements

### Short-term (Next sprint)
1. Add conversation memory (multi-turn Q&A, not just single questions)
2. Implement A/B testing (test new reranker vs old, measure impact)
3. Add feedback loop (user ratings on answers → retrain)

### Medium-term (Next quarter)
1. Fine-tune embedding model on support data (could improve relevance 5-10%)
2. Add autonomous learning (monitor escalations, rewrite documents proactively)
3. Implement knowledge base versioning (A/B test different documents)

### Long-term (Production roadmap)
1. Multimodal support (images in documents + answers)
2. Autonomous ticket triage (categorize support tickets by complexity)
3. Custom LLM fine-tuning (reduce hallucinations on company-specific data)

## Testing

```bash
# Unit tests
make test

# With coverage report
make test-coverage

# Integration tests (requires Docker Compose)
make test-integration

# RAGAS evaluation (requires eval dataset)
make evaluate-ragas
```

All tests pass with >80% coverage.

## Monitoring & Metrics

Dashboard: [link to metrics]

Key metrics tracked:
- Query latency (p50, p95, p99)
- Escalation rate
- User satisfaction (thumbs up/down)
- Cost per query
- System uptime

Alert triggers:
- P99 latency > 1000ms
- Escalation rate > 20%
- Error rate > 1%
- System downtime

## Contributing

Issues and PRs welcome.

For bigger changes, open an issue first to discuss.

## License

MIT

---

## How to Use This Project as a Base

This project is designed to be forked and modified for YOUR domain:

1. Fork the repo
2. Update the knowledge base (replace support docs with your docs)
3. Adjust temperature/prompts for your use case
4. Run `make evaluate-ragas` to see quality on your data
5. Deploy to production (Railway, Render, or your infrastructure)

Questions? Open an issue.
```

---

## 2. Architecture Diagrams: Free Tools

Your diagram choice reflects your maturity. Sloppy diagram = sloppy system.

### Excalidraw (Recommended for Hand-Drawn Style)

**Best for:** "This looks like a human drew it" (surprisingly professional)

**Link:** [excalidraw.com](https://excalidraw.com)

**Pros:**
- No account needed
- Collaborative (share link)
- Exports as PNG, SVG, PDF
- Fast and intuitive

**How to embed in README:**

```markdown
## Architecture

![System Architecture](assets/architecture.png)
```

**Workflow:**
1. Visit excalidraw.com
2. Draw boxes, arrows, labels
3. Export as PNG → add to `/assets` folder
4. Reference in README

**Example diagram:**
```
[Query] ──→ [Embedding] ──→ [Vector DB] ──→ [Rerank] ──→ [LLM] ──→ [Answer]
                                 ↑
                            [Documents]
```

Takes 10 minutes. Looks professional.

### draw.io (Best for Formal Architecture)

**Best for:** Formal system diagrams (enterprise style)

**Link:** [draw.io](https://draw.io) (can run offline)

**Pros:**
- Professional look
- Desktop app available
- Tons of shapes/icons
- Works offline

**How to use:**
1. draw.io → new diagram
2. Drag shapes from left sidebar
3. Connect with arrows
4. Export as PNG
5. Add to repo

### Mermaid (Best for Version Control)

**Best for:** Diagrams stored in code (not images)

**Markdown syntax:**
```markdown
## Architecture

\`\`\`mermaid
graph TD
    A[User Query] --> B[Embedding]
    B --> C[Vector Search]
    C --> D[Reranking]
    D --> E[LLM Generation]
    E --> F[Response]
\`\`\`
```

**Pros:**
- Renders in GitHub automatically
- Version controlled (no image files)
- Easy to update
- Clean, readable source

**When to use Mermaid:**
- Flowcharts (data flow, decision trees)
- Sequence diagrams (API interactions)
- Dependency graphs
- System components

### ASCII Diagrams (For Terminal Docs)

When you need old-school simplicity:

```markdown
## Processing Pipeline

    ┌─────────────┐
    │  Raw Input  │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Tokenize   │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │   Embed     │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Generate   │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Output     │
    └─────────────┘
```

Works in any text editor, no tools needed.

---

## 3. Loom Demo: 2-Minute Script

Recruiters watch this for 90 seconds. If it's good, they interview you.

### Complete Script Template

**[0:00-0:10] Intro (10 seconds)**
```
"Hi, I'm [name]. This is [project name]. 
It solves [problem] by [approach].
Let me show you how it works."
```

**[0:10-0:40] The Problem (30 seconds)**
```
"Before this project, [describe the problem].
This meant [impact: time wasted, money, user friction].

I built [project] to solve this."
```

Show a screenshot or short video of the problem (optional).

**[0:40-1:30] Demo (50 seconds)**
```
"Let me show you it in action."

[OPEN APP]
[User types: "How do I reset my password?"]
[System responds with answer in 0.4 seconds]
[Show the answer is good]
[Show you can ask another question]

"The system processed that in 400ms."
```

This is the most important part. It needs to:
- Work smoothly (rehearse this 3x)
- Show the interesting technical part (not boilerplate UI)
- Show it's fast (latency matters)
- Show it's reliable (no errors)

**[1:30-1:50] The Results (20 seconds)**
```
"Since deploying this, we've seen:
- 45% faster support response time
- $150K annual cost savings
- 87% accuracy on the RAGAS benchmark
- 99.7% uptime in production

The code is on GitHub: [repo link]"
```

**[1:50-2:00] Closing (10 seconds)**
```
"That's the project. Thanks for watching!
Questions? Let me know."
```

---

### Recording Setup

**Desktop:**
- Close all notifications
- Use incognito browser (no autocomplete)
- Pre-load demo data (don't show loading times)
- Close Slack, email, calendar
- Put phone on silent

**Screen Recording:**
- 1080p resolution minimum
- 1.5x zoom on text (don't strain viewer's eyes)
- Enable cursor highlighting (shows where you're clicking)
- No system sounds (notifications, alerts)

**Audio:**
- Quiet room
- USB headset mic or external mic (not laptop mic)
- Clear, medium-paced speaking
- Minimal "ums" and "ahs" (rehearse)

**Files:**
- Use Loom (direct link, no YouTube algorithm)
- Or OBS + upload to YouTube unlisted link

**Before uploading:**
- Watch it once at full speed (would a recruiter click away?)
- Watch it muted (does the visual tell the story?)
- Check audio quality (not tinny or muffled?)

---

### What NOT to Do

!!! warning "Don't: Show code in the demo"
    Recruiters don't care about code in a 2-min demo.
    
    Show working system, not implementation.

!!! warning "Don't: Long loading screens"
    If something takes 5 seconds, pre-load it before recording.
    
    Boring loses attention.

!!! warning "Don't: Talk too fast"
    Nervous energy makes you rush. Slow down.
    
    Imagine explaining to a colleague, not giving a pitch.

!!! warning "Don't: Just show the UI"
    "Here's the login page... here's the dashboard..."
    
    Boring. Show the intelligent part. What's interesting?

---

## 4. Cheap Demo Deployment Options

Deploy your project for cheap. Recruiters want to see it live.

| Platform | Free Tier | Sleep on Idle | Custom Domain | Docker | Best For | Cost |
|----------|-----------|---|---|---|---|---|
| **Railway** | $5/month | No | Yes | Yes | FastAPI apps | $5-20 |
| **Render** | Yes (45 min sleep) | Yes | Yes | Yes | Full stack | Free-$10 |
| **Fly.io** | Yes | No | Yes | Yes | Global apps | Free-$20 |
| **Vercel** | Yes | N/A | Yes | No | Frontend | Free |
| **HuggingFace Spaces** | Yes | Yes | No | Yes | Model demos | Free |
| **Replit** | Yes | N/A | Yes | Yes | Prototypes | Free |

### Railway (Recommended for AI Projects)

**Why:** Good free tier, fast, supports GPU for inference

**Steps:**
1. Create account at [railway.app](https://railway.app)
2. Connect GitHub repo
3. Railway auto-detects `Dockerfile` and deploys
4. Get public URL instantly
5. Cost: $5 baseline/month, pay-as-you-go after

**Deploy in 5 commands:**
```bash
# 1. Create Dockerfile in repo root
# 2. Commit to GitHub
# 3. Create Railway project
# 4. Connect GitHub repo
# 5. Done (auto-deploys on push)

# See logs
railway logs
```

### HuggingFace Spaces (Free, Best for Models)

**Why:** Free indefinitely, designed for AI demos, built-in Gradio support

**Steps:**
1. Create account at huggingface.co
2. Create new Space
3. Choose Gradio or Streamlit
4. Deploy your demo
5. Get public link

**Example Gradio app:**
```python
import gradio as gr
from src.rag import RAGSystem

rag = RAGSystem()

def answer_query(query: str):
    result = rag.query(query)
    return result['answer'], result['sources']

demo = gr.Interface(
    fn=answer_query,
    inputs="text",
    outputs=["text", "json"],
    title="Support AI"
)

demo.launch()
```

Deploy: push to HF Space repo → auto-deploys

---

## 5. Tech Stack Badges

Make your README visually show your tech choices.

**Start your README with badges:**

```markdown
# Project Name

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue?logo=python&logoColor=white)](https://www.python.org/downloads/)
[![FastAPI](https://img.shields.io/badge/fastapi-0.104+-009485?logo=fastapi)](https://fastapi.tiangolo.com)
[![PostgreSQL](https://img.shields.io/badge/postgresql-16+-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org)
[![Redis](https://img.shields.io/badge/redis-7+-dc382d?logo=redis&logoColor=white)](https://redis.io)
[![Qdrant](https://img.shields.io/badge/qdrant-vector%20db-8b47c2)](https://qdrant.tech)
[![Docker](https://img.shields.io/badge/docker-containerized-2496ed?logo=docker&logoColor=white)](https://www.docker.com)
[![Tests](https://img.shields.io/badge/tests-100%25-brightgreen)](link-to-ci)
```

**How to create shields.io badges:**

```markdown
![Name](https://img.shields.io/badge/LABEL-MESSAGE-COLOR?logo=LOGONAME&logoColor=white)
```

**Popular badge combinations:**

```markdown
# Backend
[![FastAPI](https://img.shields.io/badge/fastapi-009485)](https://fastapi.tiangolo.com)
[![Python](https://img.shields.io/badge/python-3.10+-3776ab?logo=python)](https://python.org)

# Databases
[![PostgreSQL](https://img.shields.io/badge/postgresql-336791?logo=postgresql)](https://postgresql.org)
[![Redis](https://img.shields.io/badge/redis-dc382d?logo=redis)](https://redis.io)
[![Qdrant](https://img.shields.io/badge/qdrant-vector-8b47c2)](https://qdrant.tech)

# ML/AI
[![OpenAI](https://img.shields.io/badge/openai-black?logo=openai)](https://openai.com)
[![HuggingFace](https://img.shields.io/badge/huggingface-FFB000)](https://huggingface.co)
[![LangChain](https://img.shields.io/badge/langchain-121212)](https://langchain.com)

# DevOps
[![Docker](https://img.shields.io/badge/docker-2496ed?logo=docker)](https://docker.com)
[![Kubernetes](https://img.shields.io/badge/kubernetes-326ce5?logo=kubernetes)](https://kubernetes.io)
[![AWS](https://img.shields.io/badge/aws-FF9900?logo=amazonaws)](https://aws.amazon.com)

# Testing
[![Pytest](https://img.shields.io/badge/pytest-0A9EDC?logo=pytest)](https://pytest.org)
[![Coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)](https://coverage.readthedocs.io)
```

---

## 6. Lessons Learned: How to Write Them Right

This section makes or breaks your credibility.

### Bad Examples (Don't Write These)

```
"I learned a lot about RAG and how powerful it is."
= Generic, vague, shows you didn't learn specifically

"RAG is super useful for production systems!"
= Enthusiasm without insight

"The tech stack was interesting to work with."
= No learning, just generic praise

"The project taught me that testing is important."
= Obviously true, not your specific lesson
```

---

### Good Examples (Write Like This)

```
Lesson 1: Embedding Model Choice Had Huge Impact on Performance

I assumed the AI/LLM was the bottleneck, so I tested different models (GPT-3.5 vs GPT-4, Mistral vs Claude).

Reality: The embedding model had 100x more impact on retrieval quality.
Switching from `text-embedding-3-large` to `text-embedding-3-small` actually improved quality because small was optimized for our document lengths AND was 5x cheaper.

Takeaway: Always measure the actual bottleneck, not what you think it is.

---

Lesson 2: Confidence Thresholds Beat Model Quality Improvements

We spent 2 weeks improving model faithfulness from 0.85 → 0.87 (1.5% improvement).

Meanwhile, adjusting our escalation-to-human threshold from 0.75 → 0.70 had 10x more impact on user experience.

Lesson: System design (when to fallback, when to escalate) matters more than incremental model improvements. The cheapest optim is often architectural, not algorithmic.

---

Lesson 3: You Can't Rerank Without Ground Truth

Our off-the-shelf Cohere reranker got 65% of relevance ordering correct on our data.

After fine-tuning on 500 labeled support tickets, it jumped to 92%.

Lesson: General-purpose models are 80% effective. The last 20% requires ground truth from YOUR domain. Plan for labeling data early.
```

---

### Template for Good Lessons

```
Lesson: [Specific topic]

Assumption I had: [What I expected]

Reality: [What actually happened, with numbers/evidence]

Why this matters: [Impact on the project or your understanding]

Takeaway: [One sentence principle you learned]
```

---

## Summary: Maximum Impact READMEs

✅ **Structure:** Problem → Solution → Architecture → Results → Lessons → Next

✅ **Visual:** Badges, diagrams, demo GIF, architecture drawings

✅ **Credible:** Specific metrics, real tradeoff decisions, honest lessons

✅ **Actionable:** Setup takes 2 commands, clear API docs, examples work

✅ **Personality:** Your voice, specific details, real story

This is how senior engineers document projects.

---

**Next:** [What's Next →](whats-next.md)
