# Resume & LinkedIn: Getting Interviews

Your GitHub shows what you can do. Your resume gets you in the door.

This section is **tactical:** what actually works to get AI engineering interviews.

---

## Resume Strategy: ATS Meets Human

Your resume gets 6 seconds from a human... but first it passes through ATS (Applicant Tracking System) software that scans for keywords.

**Top ATS Keywords for AI Roles:**

```
Python, PyTorch, TensorFlow, Hugging Face, LLM
RAG, Semantic Search, Vector Embeddings, Qdrant
Fine-tuning, LoRA, Prompt Engineering, Agents
MLflow, DVC, Weights & Biases, Experiment Tracking
FastAPI, Flask, Django, REST APIs
PostgreSQL, Redis, Vector Databases, Database Design
Docker, Kubernetes, AWS ECS, AWS Lambda
GitHub Actions, CI/CD, Automation
Evaluation Metrics, RAGAS, Model Evaluation
Cost Optimization, Latency Reduction, System Design
```

**Strategy:** Sprinkle these naturally through your resume. Don't keyword-stuff—it's obvious and fails.

---

## Resume Structure (One Page for Junior, Two for Senior)

### Header

```
JOHN SMITH | AI Engineer | San Francisco, CA | john@email.com | github.com/johnsmith | linkedin.com/in/johnsmith
```

### Skills (Ordered by Hirability)

Section **MUST** follow this order:

```
SKILLS
• Languages: Python, SQL, JavaScript
• AI/ML: PyTorch, Transformers, LLM APIs (OpenAI, Claude), RAG, Fine-tuning (LoRA)
• Vector Databases: Qdrant, Pinecone, FAISS, embeddings (semantic search)
• ML Operations: MLflow, DVC, Weights & Biases, Experiment tracking, evaluation (RAGAS)
• Backend: FastAPI, PostgreSQL, Redis, system design
• DevOps: Docker, Kubernetes, AWS ECS, GitHub Actions
• Tools: Jupyter, Git, VS Code, Langchain, LlamaIndex
```

**Why this order?** Most AI roles want: Python → AI/ML frameworks → Vector DBs → MLOps → Backend → DevOps. Recruiters scan top-to-bottom.

---

## Experience Section: CAR Format

**Challenge → Action → Result**

**What NOT to do:**
```
Senior Software Engineer, Company X (2022-2024)
• Worked on Python backend
• Implemented REST APIs
• Managed database layer
```

❌ These are job duties, not achievements. Doesn't differentiate you.

**What TO do:**
```
AI Engineer, Company X (2022-2024)
• Built RAG system for customer support (semantic search + LLM): improved resolution
  from 65% to 92%, reduced support team cost by $500K/year
• Deployed LLM inference on AWS ECS (FastAPI, 1000 requests/sec): achieved <2s p99
  latency, scaled 3→20 pods under load, cost $400/month
• Trained and evaluated LoRA fine-tuned models with MLflow + Weights & Biases:
  achieved 0.92 RAGAS faithfulness, iterated 50+ experiments in 2 weeks
• Optimized embedding inference speed by 40% (caching + batch processing):
  reduced API latency by 200ms
```

✅ Specific metrics. Shows progression. Actionable.

---

## Format: ATS-Friendly

```
❌ DON'T:
• Multi-column layouts (breaks ATS parsing)
• Fancy fonts or colors
• Graphics/infographics
• Tables (parsed wrong)

✅ DO:
• Single column, plain text
• Standard fonts (Arial, Calibri, Helvetica)
• Black text on white background
• Bullet points
• Save as PDF (preserves formatting)
```

---

## Projects Section

For junior roles, projects section is critical (you have less work experience).

```
PROJECTS (From Portfolio)
• RAG Semantic Search System | github.com/you/rag-evaluator
  Built Python RAG system (embeddings + Qdrant + LLM). Achieved 0.87 RAGAS
  faithfulness. Deployed with FastAPI, containerized with Docker.
  
• Production AI Platform on Kubernetes | github.com/you/k8s-ai
  Deployed FastAPI + Qdrant + PostgreSQL on AWS EKS with 3-replica StatefulSets.
  Configured HPA (auto-scale 3→20 pods). Monitoring: Prometheus + Grafana.
  Latency: 50ms p99.
  
• Fine-Tuning Pipeline with MLflow | github.com/you/lora-training
  Trained LoRA models with MLflow tracking + DVC versioning. Evaluated with RAGAS.
  GitHub Actions auto-deploys best model. 50+ experiments tracked.
```

---

## Which Projects to Highlight by Role

### AI Engineer Role

1. RAG system (must)
2. Fine-tuning pipeline (must)
3. K8s deployment (strong)

### MLOps Engineer Role

1. MLflow tracking system (must)
2. DVC pipeline (must)
3. GitHub Actions automation (must)

### Full-Stack AI Engineer Role

1. FastAPI API (must)
2. K8s deployment (must)
3. MLOps pipeline (must)

### ML Infrastructure / Platform Engineer Role

1. K8s deployment (must)
2. Monitoring setup (must)
3. Autoscaling implementation (must)

---

## ATS Keywords: Real Example

**BEFORE:**
```
Developed Python backend, implemented APIs, worked with databases
```

**AFTER:**
```
Developed FastAPI backend serving 1000 req/sec. Implemented REST APIs for
RAG system using LLM inference. Managed PostgreSQL + Redis, optimized query
latency by 40%.
```

See? Keywords (FastAPI, REST, RAG, LLM, PostgreSQL, Redis) are naturally there because you're being specific.

---

## LinkedIn: Getting Recruiter Attention

### Headline Formula

```
[Skill 1] + [Skill 2] | [What You Build] | [Differentiator]

Examples:
AI Engineer specializing in RAG systems | Building production LLM products | Ex-[Company]
Full-Stack AI Engineer | LLMs + Infrastructure | Open source contributor
ML Systems Engineer | Kubernetes + MLOps | 20+ models in production
```

**NOT:**
```
❌ "High achiever passionate about AI" (means nothing)
❌ "Looking for opportunity in machine learning" (everyone is)
❌ "Software Engineer" (too vague, no AI signal)
```

### About Section (3-Paragraph Formula)

```
Paragraph 1: Who you are
AI engineer building production-scale LLM systems. Specialized in RAG, fine-tuning,
and infrastructure. Recently worked with [models/companies].

Paragraph 2: What differentiates you
Completed end-to-end AI systems from training → deployment → monitoring. 
Published [X] projects on GitHub ([Y] stars). Comfortable with both model 
optimization and infrastructure concerns (not just model code).

Paragraph 3: What you're looking for + CTA
Interested in roles where I can impact users directly (not academic research).
Prefer teams that value shipping over perfection. Let's talk:
📧 john@email.com
🔗 github.com/johnsmith
```

### LinkedIn Content: The Secret Weapon

Post publicly (1x per month):
- Project you built
- Lesson you learned
- Question you investigated

Example:
```
Built a rag system for PDFs. Thought semantic search would be best, but keyword
search was faster for short queries. Ended up with a hybrid approach (keyword first,
semantic for edge cases). Big lesson: benchmark your assumptions.
Repo: github.com/you/rag

[Get 100+ likes from AI community, recruiters see it]
```

---

## LinkedIn Endorsements (Strategic)

Get endorsed for:
1. Python
2. PyTorch or TensorFlow
3. LLMs or Large Language Models
4. AWS or Google Cloud
5. Kubernetes (if doing platform eng)

**How to get endorsements:** Ask 5-10 AI people you know. "I'd take it as a favor."

---

## Endorsements Worth Having (by Role)

| Role | Top 3 Endorsements |
|------|---|
| AI Engineer | Python, PyTorch, Large Language Models |
| MLOps Engineer | Python, MLflow, Kubernetes |
| Full Stack | FastAPI, AWS, Kubernetes |
| Data Engineer | Python, PostgreSQL, AWS |

---

## Cover Letter: Why It Matters

Most people ignore cover letters. Don't.

**Cover Letter Formula:**

```
Paragraph 1: Hook
"I noticed [Company Name] released [specific model/feature]. I built something similar
(link) and am excited to solve this problem at scale."

Paragraph 2: Fit
Show you understand their specific problem. Name someone on the team if you can find them.
"Your challenges around [specific tech] align with my experience building [your thing]."

Paragraph 3: You
Why you specifically. What value do you bring?
"I've shipped production AI systems on AWS ECS and Kubernetes, understand the cost
tradeoffs, and debug at the infrastructure level."

Paragraph 4: Close
"I'd love to talk. I've attached my resume, GitHub, and a working demo of my recent project."
```

**Example (Real):**

```
Hi [Hiring Manager],

I saw Anthropic released prompt caching. I built something similar (semantic caching for embeddings) and achieved 35% latency improvement for RAG queries.

Your challenges scaling Claude at production scale match my experience operating LLM inference on AWS ECS (1000 req/sec). I've debugged distributed systems, optimized latency, and managed costs across infrastructure.

I'd love to chat about how I can contribute.

John
[Resume]
[GitHub: github.com/johnsmith]
[5-min project demo: loom.com/share/...]
```

**Strong points:**
- Shows you researched the company
- Proves technical capability with concrete example
- Mentions scale (their pain point)
- Provides proof (demo link)

---

## Cover Letter DON'Ts

❌ "I'm passionate about AI" (everyone says this)
❌ Long cover letter (keep it <200 words)
❌ Generic template (customize for EACH role)
❌ "I'm excited to learn" (hiring, not training)

---

## Where to Find AI Jobs

**Direct applications:**

1. **Anthropic:** careers.anthropic.com (direct Claude work)
2. **OpenAI:** openai.com/careers (GPT work)
3. **Google:** careers.google.com/jobs (search "AI Engineer")
4. **Amazon AWS:** AWS ML/AI roles highly competitive
5. **Stripe:** stripe.com/jobs (high-quality bar)

**Job boards:**

- **Linkedin Jobs:** linkedin.com/jobs (best recruiter volume)
- **HN Jobs:** news.ycombinator.com/jobs (quality startups)
- **Angel List:** angel.co (startup equity)
- **RemOK:** remoteok.com (Fully remote)

**Networking (actually works):**

- Discord communities (LangChain, Hugging Face)
- Twitter/X (follow AI researchers and engineers)
- Conferences (NeurIPS, AI Summit)
- University connections (if recent grad)

---

## Job Search Strategy

**Week 1:**
- Polish resume (2 hours)
- Update LinkedIn (1 hour)
- Record demo videos (3 hours)
- Total: 6 hours

**Week 2-3:** Start applying
- Create application tracker (Google Sheet tracking applications, responses)
- Apply to 5-10 companies per day (customize each)
- Target: 50-100 applications

**Week 4:** Interviews start
- Most companies interview within 5-7 days
- Do mock interviews (see Phase 5: Mock Interviews)
- Practice system design (see Phase 5: System Design)

**Expected outcome:** 3-5 interviews from 50 applications (5-10% callback rate)

---

## 21. Next: System Design Interviews

Your resume gets you the interview.

System design interviews are how they evaluate seniority.

→ **[System Design Interviews →](./system-design.md)**
