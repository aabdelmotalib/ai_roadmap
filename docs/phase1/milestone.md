# Phase 1 Milestone: AI Document Intelligence Platform

Phase 1 is **hard**. You're learning 6 new concepts with real code depth.

By the end, you should be able to build production AI systems.

This milestone validates you're ready for Phase 2.

---

## Exit Criteria Checklist (25+ Items)

### LLM APIs (Must Pass All)

- [ ] Can write async OpenAI client code without reference
- [ ] Understand token counting (can estimate cost before API call)
- [ ] Have implemented function calling (multi-step tool loop)
- [ ] Know difference between stop, max_tokens, temperature
- [ ] Can implement fallback chain (cheap → expensive → cache)
- [ ] Have used structured output (JSON schema, instructor library)
- [ ] Can explain why streaming helps UX
- [ ] Know when to use GPT-4 vs 4o-mini

### Embeddings (Must Pass All)

- [ ] Can embed text with OpenAI API
- [ ] Understand cosine similarity (why angle, not distance)
- [ ] Know difference between different embedding models (3-small vs 3-large)
- [ ] Have set up Qdrant locally and queried it
- [ ] Can implement semantic caching
- [ ] Understand HNSW indexing (why it's fast)
- [ ] Know pros/cons of vector databases (Qdrant, Pinecone, pgvector)

### RAG (Must Pass All)

- [ ] Can load documents (PDF, Word, web)
- [ ] Understand chunking strategies (fixed vs semantic)
- [ ] Have implemented hybrid search (BM25 + dense)
- [ ] Can use reranking to improve results
- [ ] Have built complete RAG endpoint (query → retrieve → generate)
- [ ] Understand RAGAS metrics (know when to worry)
- [ ] Know failure modes (wrong docs, hallucination, stale info)

### Agents (Must Pass All)

- [ ] Understand ReAct pattern (reason → act → observe)
- [ ] Can define tools with JSON schemas
- [ ] Have written agent loop (handle tool_calls)
- [ ] Know iteration limits (prevent infinite loops)
- [ ] Understand tool safety (permissions, approval, sandboxing)
- [ ] Know cost per agent task (rough estimate)

### Fine-Tuning (Must Pass All)

- [ ] Understand when to fine-tune vs RAG vs prompt engineering
- [ ] Know data format (JSONL messages format)
- [ ] Have used OpenAI fine-tuning API OR HuggingFace LoRA
- [ ] Understand LoRA config (r, alpha, target_modules)
- [ ] Know catastrophic forgetting (how to prevent)
- [ ] Understand dataset size tradeoff (quality > quantity)

### Evaluation (Must Pass All)

- [ ] Can interpret 4 RAGAS metrics (faithfulness, relevancy, recall, precision)
- [ ] Have built golden test set (50+ examples)
- [ ] Understand LLM-as-judge approach
- [ ] Know how to A/B test two prompts
- [ ] Can set up monitoring with alerts

### Production Readiness (Must Pass All)

- [ ] Know security risks (prompt injection, PII, secrets)
- [ ] Can estimate monthly costs for your system
- [ ] Know how to optimize (caching, model selection, batching)
- [ ] Have written production code (not just scripts)
- [ ] Understand error handling (graceful failures)

---

## Capstone Project: AI Document Intelligence Platform

**What You'll Build:** Production system that answers questions about documents.

### Problem Statement

Companies have 1000s of documents (PDFs, Word, web). Employees waste 10+ hours/week searching.

You'll build an AI system that:
- Ingests documents
- Answers questions with sources
- Learns from feedback
- Costs < $100/month to operate

### What to Build

**Core Features:**

1. **Document Ingestion**
   - Accept PDF/Word/web uploads
   - Parse and chunk intelligently
   - Store in vector database
   - Track document version (update not replace)

2. **Question Answering**
   - Semantic search for relevant docs
   - Rerank top results
   - Generate answer with LLM
   - Return cited sources
   - Streaming response

3. **Feedback Loop**
   - User rates answer quality (👍 👎)
   - Log query + answer + feedback
   - Monthly analysis: which docs need updating?
   - Which queries should be handled differently?

4. **Cost Tracking**
   - Log every API call cost
   - Daily dashboard
   - Alert if monthly cost exceeds budget

### Required API Endpoints

```
POST   /documents/upload       # Upload PDF/Word/URL
POST   /documents/{id}/delete  # Remove document
POST   /search                 # Semantic search (debug)
POST   /query                  # Answer question with LLM
POST   /feedback               # Log user feedback
GET    /costs/daily            # Daily cost breakdown
GET    /analytics/quality      # Answer quality trends
```

### Success Criteria

- [ ] Ingests 100+ documents
- [ ] Answers 50 test questions with > 0.8 faithfulness score
- [ ] Response time < 3 seconds
- [ ] Monthly cost < $100 for 100 queries/day
- [ ] Zero prompt injection exploits
- [ ] Graceful error handling
- [ ] Monitoring + alerting enabled

### Stretch Goals

- [ ] Fine-tune small model for your domain
- [ ] Implement multi-language support
- [ ] Add user authentication
- [ ] Deploy to AWS Lambda
- [ ] Custom UI

---

## Self-Assessment Rubric

### Beginner (Don't Pass Phase 1 Yet)

**Indicators:**
- Can follow tutorial, but can't modify code
- Don't understand why things work
- Copy-paste solutions
- No error handling
- Cost estimation missing
- No tests
- Can't debug when things break

**Example:** "I followed the RAG tutorial and it works, but when I change the prompt, everything breaks"

### Competent (Ready for Phase 2)

**Indicators:**
- Can write working code without reference
- Understand core concepts (embeddings, RAG, agents)
- Can debug when things break
- Implement error handling
- Monitor costs and performance
- Write simple tests
- Can explain to another developer

**Example:** "I built RAG for company docs. When retrieval failed, I debugged by checking embedding quality, improved chunking, and now it works for 95% of questions"

### Hire-Ready (Exceptional)

**Indicators:**
- Optimize without being asked (caching, model selection, parallel)
- Anticipate failure modes
- Security-first thinking
- Comprehensive monitoring
- Version control discipline
- Document decisions
- Mentor others

**Example:** "I shipped document intelligence system with 0.88 RAGAS score, sub-2s latency, $30/month cost, prompt injection defense, and daily monitoring dashboard"

---

## What Failed Milestone Looks Like

⚠️ **You're not ready if:**

1. ❌ "I don't understand embeddings, just use them"
   - Need: Deep understanding of what embedding is, why cos similarity works, when to switch models

2. ❌ "RAG always hallucination-free"
   - You haven't thought about failure modes. RAGAS will show you.

3. ❌ "Fine-tuning didn't work, giving up"
   - You need to debug: Is data bad? Overfitting? Wrong model? Wrong eval?

4. ❌ "No idea how much I'm spending"
   - You MUST know daily costs. Set alerts. Know your unit economics.

5. ❌ "Security? That's for later"
   - No. Prompt injection will burn you. Implement now.

6. ❌ "No tests, hope it works in prod"
   - At least golden test set. Measure quality.

7. ❌ "I can't explain what I did"
   - Document your decisions. You'll ship better code and learn faster.

8. ❌ "It works on my laptop, shipping now"
   - Nope. You need: error handling, monitoring, cost tracking, security.

---

## Common Rush Reasons & Consequences

### "Evaluation is tedious, I'll skip it"
**Consequence:** Ship broken system. Customer reports "answers are wrong." Too late to fix.
**Reality:** RAGAS takes 30 min. Saves you 10 hours of debugging.

### "Caching is too complex"
**Consequence:** $50k bill. CEO angry.
**Reality:** Semantic cache is 20 lines. Saves 80% of costs.

### "I don't need security yet"
**Consequence:** Attacker injects prompt. Your RAG returns customer passwords. Lawsuit.
**Reality:** Security is not optional. Hardened prompt takes 10 min.

### "Fine-tuning doesn't work, should I retry?"
**Consequence:** You've wasted 3 days. Move on.
**Reality:** Before fine-tuning, ask: Is this knowledge or behavior? Do I have 200+ examples? If no → use RAG or prompt engineering.

### "Cost estimation takes time"
**Consequence:** Partner cancels because system is 10x too expensive.
**Reality:** Estimation saves 80% of costs. Do it first.

---

## Capstone Implementation Levels

### Minimum Viable (Must Do)

```
Document Upload → Embed → Store → Query → Retrieve → Answer
```

- FastAPI endpoint uploads documents
- Store in Qdrant
- Answer questions
- Return sources
- Works on laptop

### Production (Should Do)

```
+ Error handling
+ Cost tracking
+ Rate limiting
+ Monitoring
+ Graceful degradation
+ Timeouts
+ Retry logic
```

### Advanced (Nice To Have)

```
+ Fine-tuning
+ Multi-language
+ User feedback loop
+ Dashboard
+ Reranking
+ Caching
+ Security hardening
```

---

## Final Checklist

- [ ] Completed all 8 Phase 1 lectures (APIs, embeddings, RAG, agents, fine-tuning, evaluation, security, cost)
- [ ] Capstone project runs without errors
- [ ] RAGAS score > 0.75 on 50 test queries
- [ ] Monthly cost estimated < $200
- [ ] Zero hardcoded secrets
- [ ] Error handling on all API calls
- [ ] Documentation of architecture
- [ ] Able to explain every decision to peer
- [ ] Can debug common errors (embedding mismatch, retrieval failures, token overruns)
- [ ] Know what to do when things break in production

---

## You're Not Alone

Phase 1 is hard. Many people:
- Feel overwhelmed (embeddings are confusing)
- Hit unexpected errors (RAGAS says 0.4 faithfulness, why?)
- Spend days debugging (wrong chunking, bad model, corrupt data)
- Second-guess themselves (am I doing this right?)

**Normal.** Press forward. Back yourself up to working checkpoint frequently.

---

## What's in Phase 2

Phase 2 (6 weeks): **Docker, Kubernetes & Production Infrastructure**

Your system needs:
- Containerization (Docker)
- Orchestration (Kubernetes or Cloud Run)
- CI/CD pipeline
- Secrets management
- Monitoring at scale
- Database migrations
- Zero-downtime deployments

Phase 1 was **software engineering**.
Phase 2 is **production engineering**.

You'll learn:
- Docker: Package your AI app
- Kubernetes: Scale to 1000s of requests
- AWS/GCP: Managed infrastructure
- PostgreSQL with pgvector: Production vector DB
- CI/CD: Every commit ships safely
- Terraform: Infrastructure as code

→ **[Phase 2: Docker & Kubernetes →](../../phase2/overview.md)**

---

## Resources

- [OWASP Top 10 LLMs](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [RAGAS Framework](https://ragas.io/)
- [Qdrant](https://qdrant.tech/)
- [OpenAI API](https://platform.openai.com/docs)
- [LangChain Reference](https://python.langchain.com/)

---

## Feedback

You made it to Phase 1 milestone. That's significant.

You learned:
- How to talk to LLMs (APIs, tokens, models)
- How to find relevant information (embeddings, search)
- How to combine knowledge + generation (RAG)
- How to build autonomous systems (agents)
- How to teach models (fine-tuning)
- How to measure quality (evaluation)
- How to ship securely (security)
- How to not bankrupt yourself (cost)

**You're now capable of building real AI systems.**

Next: Make them reliable at scale (Phase 2).

---

**Congratulations on Phase 1.** 🎓
