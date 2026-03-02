# System Design Interviews for AI Engineers

System design interviews test how you **architect complex systems**.

Not: "write quick code" (that's coding interviews).
**But:** "Design a system where X users do Y at Z scale."

For AI roles, expect questions like:
- "Design a recommendation system using LLMs"
- "Design a multi-tenant prompting platform"
- "Design a content moderation system at scale"

Interviewers are testing:
- Do you know **tradeoffs** (quality vs latency, cost vs...)?
- Can you **scale thinking** (how does this break at 10x)?
- Do you **reason about unknowns** (what would you measure)?

---

## Framework: How to Approach Any Question

**Step 1: Clarify Requirements (2 min)**

Ask questions. Don't assume.

```
Good questions:
- "How many users? Concurrent?"
- "What's the latency budget?"
- "What's the accuracy requirement?"
- "Budget constraints?"
- "Geography (single region or global)?"

Don't ask:
- "Can I assume good network?" (yes, obviously)
- "Can I assume infinite resources?" (no, that's silly)
```

**Step 2: Estimate Scale (2 min)**

Back-of-envelope calculation.

```
Example: "Design a recommendation system for Netflix"

Users: 250M globally
Active per day: 100M
Concurrent: 2M (peak times)
Requests per user per day: 50 (browsing + watching)
Total QPS: (100M * 50) / 86400 = ~60K requests/second

Latency budget: <500ms (users expect snappy)
Accuracy: Top 10 recommendations must be relevant (no hard metric, discuss)
```

**Step 3: High-Level Architecture (5 min)**

Box and arrows. Explain each box.

```
User Query
  ↓
[API Gateway] (load balance, auth)
  ↓
[Service] (get user context)
  ↓
[ML Model] (inference pipeline)
  ↓
[Cache] (hot results)
  ↓
[Database] (store user history)
  ↓
Response to User
```

**Step 4: Dive Deep (5-10 min)**

Interviewer picks one component. You go deep.

```
If they ask about the ML model:
- What features go into it?
- Real-time or batch?
- How do you serve it at 60K QPS?
- What's the latency breakdown?
```

**Step 5: Tradeoffs & Fallbacks (3 min)**

Proactively discuss what you traded off.

```
"We chose embeddings + approximate nearest neighbor search over exact search
because we need <100ms latency. Trade-off: might miss a few good results.
Metric to monitor: recall@10."

"If the model fails, we fall back to trending content (cold path). Better
wrong result than no result."
```

---

## Full Design #1: Customer Support AI

**Question:** "Design an AI customer support system that answers 80% of questions without human escalation."

### Clarify

- **Scale:** 10K concurrent users, <2s latency, 95% uptime
- **Accuracy:** Must get at > 80% of ques right, confidence in answer
- **Cost:** <$50 per user per year for support

### Estimate

```
10K concurrent users → ~100 requests/second to support API
SLA: 95% uptime = 36 hours downtime per year
```

### Architecture

```
┌─────────────────────────────────┐
│  User Question                  │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│  FastAPI Endpoint               │
│  - Auth, rate limit             │
│  - Log request                  │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│  Intent Classification          │  (fast
, cheap)
│  - Question type?               │  (intent: refund, status, etc)
│  - Confidence: >0.9 → AI route  │
│          <0.5 → human                      │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│  Retrieve Context (RAG)         │  (knowledge base)
│  - Embed question               │
│  - Search docs + FAQs           │
│  - Rerank top-5 results         │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│  LLM Generation                 │  (Claude or GPT)
│  - Generate answer              │
│  - Extract confidence           │
│  - If <0.7 confid → escalate    │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│  Response                       │
│  - Answer + sources             │
│  - Feedback form (good/bad)     │
└─────────────────────────────────┘

[If escalation needed → human queue]
```

### Key Design Decisions

**Intent Classification First (Filter)**
Why? Cheap, fast (<50ms). If question is "refund", use refund specialist template.
If low confidence →  human immediately (don't waste LLM).

**RAG + LLM (Not Pure LLM)**
Why? LLM alone hallucinated. RAG grounds answers in real docs. Improves accuracy from 65% to 92%.

**Reranking**
Why? Initial retrieval might get top docs wrong (BM25 and embedding search disagree).
Reranker (small BERT model) picks the actual best doc in <100ms.

**Fallback to Human**
Why? AI confident but wrong is worse than saying "let me get a human".
Confidence threshold: <0.7 → escalate.

### Latency Breakdown

```
Total budget: <2000ms

Intent classification:     150ms (CPU inference, fast)
Embedding + retrieval:     400ms (vector DB search + network)
Reranking:                200ms (small model)
LLM generation:           900ms (token by token)
Response overhead:         350ms (serialization, network)
                         ------
Total:                   2000ms ✓
```

### Cost Analysis

```
Per user per year: <$50

Assumptions:
- 1 request per user per month
- 80% AI, 20% human
- Human support: $0.50 per request (salary)
- LLM (Claude): $0.01 per request

80% AI routes: 0.80 * $0.01 = $0.008
20% human: 0.20 * $0.50 = $0.10
Per request: $0.108

12 requests/year: $0.108 * 12 = $1.30 per user/year ✓✓✓ (well under $50)
```

### What Changes at 10x Scale?

```
100K concurrent users = 1000 QPS

- Inference latency matters more (queuing)
- LLM API rate limits (need batch)
- DB connection pooling required
- Caching layer essential (same questions asked repeatedly)
- Multi-region deployment (latency reduction)
```

---

## Full Design #2: Code Review AI (GitHub Copilot-like)

**Question:** "Design an IDE plugin that suggests code improvements in real-time."

### Scale

- 500K developers using plugin
- 100K concurrent at peak
- Must respond in <500ms (IDE blocking)
- Accuracy: suggestions should be useful 70% of time

### Architecture

```
IDE Editor
  ↓
(VSCode plugin detects: pause typing for 0.5s)
  ↓
Send code snippet + context
  ↓
[Code embedding model]  (fast, local, <100ms)
  ↓
[Retrieval]  (similar code from codebase)
  ↓
[LLM generation]  (suggest improvement)
  ↓
Display in IDE sidebar
```

### Deep Dive: The Code Embedding Model

**Why not just use GPT-4?**
- Too slow (1-2 seconds)
- IDE freezes while waiting
- Expensive ($0.01 per request = $1M/month at scale)

**Better: Local + Small Model**
- Use DistilBERT or CodeBERT-small (locally on IDE)
- <100ms, no network latency, no cost
- For retrieval only (find similar code)

**Then, use LLM for generation** (background, non-blocking)

### Code Snippet Example

**User writes:**
```python
def process_data(df):
    for i in range(len(df)):
        value = df.iloc[i]['column']
        # process value
```

**Plugin (local model) detects:**
```
This is inefficient iteration. Similar code in repo uses:
    df.apply(lambda row: process_row(row), axis=1)
```

**Plugin suggests this in sidebar (non-blocking):**
"Vectorized pandas is 100x faster. Try df.apply() instead."

**If user clicks "Generate example":**
(Background LLM call, shows result after 2 seconds, user kept typing)

```python
def process_data(df):
    return df.apply(lambda row: row['column'] * 2, axis=1)
```

### Key Decision: Local vs Remote

| Aspect | Local Model | Remote (LLM) |
|--------|---|---|
| Latency | <100ms | 500-2000ms |
| Cost | $0 (runs on laptop) | $1M/month at scale |
| Quality | Good (embedding retrieval) | Excellent (LLM generation) |
| Use | Real-time suggestions | Background, optional features |

**Hybrid approach:** Local + remote enables snappy UX without breaking bank.

---

## Full Design #3: Document Intelligence (Law Firm)

**Question:** "Design a system that extracts key info (parties, clauses, dates) from legal documents with 99% accuracy (liability-sensitive)."

### Why This is Different

Not all AI is same difficulty.

- **Web search AI:** 70% accurate is fine (user filters)
- **Medical AI:** 99% accuracy + explainability + audit trail (liability)

This system requires:
- High accuracy (wrong legal citation = lawsuit risk)
- Explainability (lawyers need to understand why system said X)
- Audit trail (who reviewed, who approved, when)
- On-prem option (documents are confidential)

### Architecture

```
Document Upload
  ↓
[Virus scan]
  ↓
[OCR]  (if PDF image)
  ↓
[PII Redaction]  (strip names, addresses before processing)
  ↓
[Chunking]  (documents are long)
  ↓
[Information Extraction]
  - Parties: GPT-4 + few-shot examples
  - Clauses: DistilBERT classifier (faster)
  - Dates: Regex + validation
  ↓
[Confidence Scoring]  (if <0.9 → flag for human review)
  ↓
[Lawyer Review UI]
  - Show extracted info
  - Show sources (which parts of doc)
  - One-click approve/correct
  ↓
[Storage + Audit Log]
  ↓
[Generate Report]  (extracted info + audit trail)
```

### Security/Privacy Layer

```
Raw Doc (confidential)
  ↓
[Anonymize]  (replace person X with PERSON_1)
  ↓
[Process]
  ↓
[Map back]  (PERSON_1 → actual name, only for authorized lawyers)
  ↓
Only authorized lawyers see original names
```

### Accuracy Strategy

```
High-accuracy pipeline:

1. GPT-4 extraction (excellent, but expensive)
2. Human lawyer review (100% accuracy, but slow/expensive)
3. Store corrections in database
4. Fine-tune cheaper model (DistilBERT) on corrections
5. Eventually, cheaper model becomes accurate

Cost:
Month 1: 100% human + GPT-4 = $10K
Month 6: 80% cheaper model + 20% human + GPT-4 = $4K
Month 12: 95% cheaper model + 5% human = $2K
```

### What Not to Do

❌ "Just use an LLM" (not accurate enough)
❌ "Fully automate" (no human review = liability)
❌ "Build our own extraction model" (legal domain requires thousands of labeled examples)

---

## Full Design #4: Content Moderation at Scale

**Question:** "Design a system that flags 99% of harmful content from 100K posts per minute."

### Challenge: Scale + Accuracy

```
100K posts/minute = 1666 QPS

Can't send every post to GPT-4 (too slow, too expensive).
```

### Solution: Tiered Approach

```
Post arrives
  ↓
Tier 1: Fast Rule-Based (regex) Filter  (<1ms)
  - Obvious profanity
  - Known bad domains
  - 60% of bad posts caught here
  
  If flagged → remove, no appeals
  ↓
  If clear safe → publish immediately
  ↓
Tier 2: Embedding Classifier (cheap, fast)  (<10ms)
  - Embed post text
  - Classify toxic/not toxic (BERT model)
  - 30% of bad posts not in Tier 1 caught here
  
  If low confidence (0.4-0.6) → Tier 3
  If confident bad → flag for human review
  ↓
Tier 3: LLM (expensive, accurate)  (async, not blocking)
  - Only run on uncertain cases (5% of posts)
  - LLM final call on edge cases
  - Flag for human moderator if LLM unsure
  ↓
Human Moderator (last resort)
  - Reviews LLM-flagged content
  - Appeals process
  - Improvement feedback loop
```

### Cost vs Accuracy Tradeoff

```
All Tier 1 (rules only):      60% caught, $0
All Tier 1 + Tier 2 (ML):     90% caught, $100K/month
All tiers (rules+ML+LLM):     98% caught, $500K/month
All tiers + humans:           99.5% caught, $1M/month
```

**Decision:** 99% target means 98% ML + 1% human review.

---

## Common Mistakes in System Design Interviews

❌ **"I don't know, let me use infinite resources"**
You: "I'll just add more servers"
Interviewer: (internal eye roll)

Instead: "Let me estimate scale and constraints first."

❌ **"The database can store everything"**
Reality: Even AWS databases have limits. You need sharding strategy.

✅ **"Let me trace through what happens at 100x scale"**
You: "At 100x, the database becomes bottleneck. Solution: sharding by user_id."
Interviewer: "Smart. What about hot shards?"

❌ **"I'll optimize later"**
No. Discuss cost/latency/accuracy tradeoffs upfront.

✅ **"Our target is <2s latency. Here's the breakdown..."**

---

## QS to Ask Interviewers (Signals Senior Level)

At end of interview, ask:

1. "What does your eval infrastructure look like? How do you measure model quality?"
2. "How do you handle model drift in production?"
3. "What's your deployment process? Can you rollback if needed?"
4. "How do you deal with data freshness vs latency?  (cache vs real-time)"
5. "What metrics do you use to evaluate success?"

Asking these shows: "I think about production systems, not just code."

---

## 21. Resources & Practice

- **System Design Interview:** https://www.youtube.com/c/ByteByteGo (highly recommended)
- **Practice:** https://www.pramp.com/ (mock interviews with real engineers)
- **Scenarios:** https://github.com/donnemartin/system-design-primer

---

## Next: Mock Interviews

Theory is great. Practice is essential.

→ **[Mock Interviews →](./mock-interviews.md)**
