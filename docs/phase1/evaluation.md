# Evaluation & Benchmarking

What you don't measure, you can't improve. What you don't evaluate, hallucinating LLM ships to production.

---

## Why This Matters

**Real incident:** Company shipped RAG without RAGAS eval. Chatbot hallucinated on 30% of queries. Customer noticed, left bad review.

**Real incident 2:** Fine-tuned model improved on test set but degraded on production queries. They didn't evaluate against general knowledge.

Evaluation separates working systems from broken ones.

---

## Conceptual Explanation

**Evaluation is grading your AI's homework.**

Without evaluation: "This looks good" (subjective).
With evaluation: "Faithfulness 0.87, Answer Relevancy 0.92" (measurable).

Three levels:
1. **Automated metrics** (RAGAS, BLEU) — quick, cheap, but proxy measures
2. **LLM-as-judge** (Claude scores your output) — better than metrics, costs money
3. **Human evaluation** (expert reviews) — gold standard, very expensive

---

## Code Examples: RAGAS Framework

### Install and Setup

```bash
pip install ragas datasets
```

### Basic Evaluation

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision
)
from datasets import Dataset

# Your RAG outputs
data = {
    "question": [
        "What's our return policy?",
        "How do I reset my password?"
    ],
    "answer": [
        "30 days from purchase...",
        "Click forgot password..."
    ],
    "contexts": [
        [["30 days from purchase for full refund..."]],
        [["Go to login, click forgot password..."]]
    ],
    "ground_truth": [
        "30 days full refund",
        "Click forgot password on login"
    ]
}

dataset = Dataset.from_dict(data)

# Evaluate
scores = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
)

print(f"Faithfulness: {scores['faithfulness']:.2f}")  # 0-1, how grounded is answer?
print(f"Answer Relevancy: {scores['answer_relevancy']:.2f}")  # 0-1, does it answer?
print(f"Context Recall: {scores['context_recall']:.2f}")  # 0-1, did we retrieve right docs?
print(f"Context Precision: {scores['context_precision']:.2f}")  # 0-1, were docs useful?
```

### LLM-as-Judge

```python
from openai import AsyncOpenAI

async def evaluate_with_llm(
    question: str,
    answer: str,
    context: str
) -> dict:
    """Use GPT-4 to grade answer."""
    client = AsyncOpenAI()
    
    prompt = f"""Grade this answer on 1-10:
    Q: {question}
    Context: {context}
    Answer: {answer}
    
    Score 10 = perfect, based on context
    Score 1 = completely wrong/hallucinated
    
    Respond with: SCORE: 8, REASON: answer accurately reflects context"""
    
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Parse response
    lines = response.choices[0].message.content.split("\n")
    score = int(lines[0].split(":")[-1].strip())
    reason = "\n".join(lines[1:])
    
    return {"score": score, "reason": reason}

# Evaluate 100 examples
test_set = [...]  # Your Q&A pairs
scores = []
for q, a, c in test_set:
    eval_result = await evaluate_with_llm(q, a, c)
    scores.append(eval_result["score"])

avg_score = sum(scores) / len(scores)
print(f"Average score: {avg_score:.1f}/10")
# Alert if < 7
```

### A/B Testing Prompts

```python
async def ab_test_prompts(test_queries: list[str]):
    """Compare two prompts."""
    client = AsyncOpenAI()
    
    prompt_a = "You are a helpful assistant. Be concise."
    prompt_b = "You are a helpful assistant. Provide detailed answers with examples."
    
    results_a = []
    results_b = []
    
    for query in test_queries:
        # Test prompt A
        response_a = await client.chat.completions.create(
            model="gpt-4o",
            system_prompt=prompt_a,
            messages=[{"role": "user", "content": query}]
        )
        results_a.append(response_a.choices[0].message.content)
        
        # Test prompt B
        response_b = await client.chat.completions.create(
            model="gpt-4o",
            system_prompt=prompt_b,
            messages=[{"role": "user", "content": query}]
        )
        results_b.append(response_b.choices[0].message.content)
    
    # Evaluate both
    scores_a = [
        (await evaluate_with_llm(q, a, "")).to_dict()["score"]
        for q, a in zip(test_queries, results_a)
    ]
    scores_b = [
        (await evaluate_with_llm(q, b, "")).to_dict()["score"]
        for q, b in zip(test_queries, results_b)
    ]
    
    return {
        "prompt_a_avg": sum(scores_a) / len(scores_a),
        "prompt_b_avg": sum(scores_b) / len(scores_b),
        "winner": "A" if sum(scores_a) > sum(scores_b) else "B"
    }
```

---

## Tools Comparison

| Tool | Cost | Speed | Accuracy |
|------|------|-------|----------|
| **RAGAS** | $0 | fast | Medium (proxy metrics) |
| **LLM-as-Judge** | $0.01-0.10/eval | medium | High |
| **Human** | $10-100/eval | slow | Gold standard |
| **Custom Model** | varies | fast | Domain-specific |

---

## Decision Framework

```
Evaluating RAG or generation?
  ├─ RAG: Use RAGAS (faithfulness, context recall)
  └─ Generation: Use LLM-as-judge (custom rubric)

100+ examples or <100?
  ├─ 100+: Start with RAGAS, supplement with LLM-judge
  └─ <100: LLM-judge only (RAGAS needs data to be reliable)

Budget?
  ├─ <$1: RAGAS free
  ├─ $10: LLM-judge 10 evals
  └─ >$100: Human eval
```

---

## Step-by-Step: Build Evaluation Pipeline

### 1. Collect Golden Dataset

```python
# Create 50 gold-standard Q&A pairs
golden_data = [
    {
        "question": "What's return policy?",
        "answer": "30 days",
        "expected_sources": ["policy.pdf"],
        "rating": "excellent"
    },
    # ... 49 more
]

with open("golden_test_set.json", "w") as f:
    json.dump(golden_data, f)
```

### 2. Run Your System

```python
results = []
for item in golden_data:
    # Call your RAG system
    response = await rag_pipeline.ask(item["question"])
    results.append({
        "question": item["question"],
        "predicted_answer": response["answer"],
        "expected_answer": item["answer"],
        "sources_used": response["sources"]
    })
```

### 3. Evaluate

```python
# RAGAS
dataset = Dataset.from_dict({
    "question": [r["question"] for r in results],
    "answer": [r["predicted_answer"] for r in results],
    "ground_truth": [r["expected_answer"] for r in results],
    "contexts": [r["sources_used"] for r in results]
})

scores = evaluate(dataset)

if scores["faithfulness"] < 0.80:
    print("Alert: RAG is hallucinating")
```

### 4. Monitor

```python
# Log metrics over time
import json
from datetime import datetime

metrics = {
    "timestamp": datetime.now().isoformat(),
    "faithfulness": scores["faithfulness"],
    "answer_relevancy": scores["answer_relevancy"],
    "num_examples": len(results)
}

with open("eval_history.jsonl", "a") as f:
    f.write(json.dumps(metrics) + "\n")

# Alert if metric drops
prev_faithfulness = 0.85
if scores["faithfulness"] < prev_faithfulness - 0.05:
    send_alert("RAG quality dropped!")
```

---

## Practical Project: Eval Dashboard

**Build:** Dashboard showing RAGAS scores over time.

**Metrics:**
- Faithfulness (is answer grounded?)
- Answer Relevancy (does it answer question?)
- Context Recall (did we retrieve right docs?)
- Context Precision (were retrieved docs useful?)

**Alert:** If any metric drops 5% in last week, alert team.

---

## Debugging Checklist: Bad Evaluation Scores

### Faithfulness Low (< 0.75)

- Retrieval is weak (wrong docs) → improve chunking/retrieval
- LLM is hallucinating → add "refuse if not in context" to system prompt
- Context is confusing → improve doc preprocessing

### Answer Relevancy Low (< 0.80)

- Query and answer mismatch → better prompt engineering
- LLM not understanding question → try clearer phrasing
- Model is too weak → upgrade to GPT-4o from 3.5

### Context Recall Low (< 0.70)

- Retrieval is missing relevant docs → rerank, improve embedding model
- Documents aren't in database → re-index
- Query is poorly matched to docs → query expansion

---

## Common Mistakes

!!! danger "Mistake 1: No Evaluation, Just Vibes"

You think system is good. It's not, you just didn't test enough cases.

**Don't:** Deploy without metrics.
**Do:** RAGAS on 50+ examples before deploy.

!!! danger "Mistake 2: Evaluating on Training Data"

You evaluate on same queries you optimized for. Overfitting.

**Don't:** Test on train set.
**Do:** Hold out separate test set.

!!! danger "Mistake 3: Metric Doesn't Matter"

You're optimizing for wrong metric. Faithfulness is 0.95 but answer is useless.

**Don't:** Trust single metric.
**Do:** Evaluate multiple angles (faithfulness, relevancy, latency).

!!! danger "Mistake 4: Evaluator LLM Is Weak**

You use GPT-3.5 to judge. It's lenient. Misleading scores.

**Don't:** Use cheap models for eval.
**Do:** Use GPT-4o with detailed rubric.

!!! danger "Mistake 5: No Baseline**

You don't know if fine-tuned model helped. No before/after comparison.

**Don't:** Skip baseline evaluation.
**Do:** Eval base + fine-tuned side-by-side.

---

## Production Realism

**Tutorial:** RAGAS score of 0.85, ship it.

**Production:**
- Baseline evaluation (base model score)
- A/B testing new version vs baseline
- Statistical significance testing (maybe just lucky)
- Continuous monitoring (scores over time)
- Alert thresholds
- Rollback plan if it degrades

---

## Cost & Performance

**Evaluating 100 examples:**
- RAGAS: $0 (free)
- LLM-as-judge (GPT-4o): $0.50-1.00
- Human eval (contractor): $500-1000

**Continuous eval in production:**
- Sample 10 queries/day
- LLM-judge scores: $0.05/day = $18/year
- Alert if faithfulness drops

---

## Case Study: Anthropic Constitutional AI

**Problem:** Fine-tune Claude to be helpful, harmless, honest.

**Solution:** Evaluate on 1000+ dimensions using LLM-as-judge.

**Result:** More aligned AI, measurable improvements.

---

## Interview Cheat Sheet

**Concept 1: RAGAS**
*Q: What's RAGAS?*
*A: Framework for evaluating RAG systems. Measures faithfulness (grounded in context), answer relevancy, context recall, context precision.*

**Concept 2: Faithfulness**
*Q: High faithfulness means?*
*A: Answer is grounded in retrieved context. Didn't hallucinate.*

**Concept 3: LLM-as-Judge**
*Q: Can LLM evaluate AI?*
*A: Yes, surprisingly well. Give detailed rubric, LLM grades. Beats simple metrics.*

**Concept 4: A/B Testing**
*Q: Why A/B test?*
*A: New prompt/model might just be lucky. A/B on 50+ queries shows if really better.*

**Concept 5: Monitoring**
*Q: When evaluate?*
*A: Before deploy (one-time). In production (continuous, alerting).*

---

## 10 Review Questions

1. RAGAS has 4 metrics. Which one measures if answer came from context?
2. Faithfulness 0.92, Answer Relevancy 0.60. Who's the problem?
3. You evaluate on training queries. What bias?
4. LLM-as-judge: why detailed rubric matters?
5. A/B test 10 examples. Margin 51% vs 49%. Conclusion?
6. Metric dropped from 0.85 to 0.80. Alert?
7. Evaluate base model, score 0.70. Fine-tune, score 0.68. Conclusion?
8. Golden test set: how many examples?
9. Continuous eval in production: frequency?
10. No eval before deploy. Risk?

---

## Flashcards

**Card 1: RAGAS**
*Q: Framework for?*
*A: RAG system evaluation (faithfulness, relevancy, recall, precision).*

**Card 2: Faithfulness**
*Q: Definition?*
*A: Is answer grounded in context? Or hallucinated?*

**Card 3: LLM Judge**
*Q: How grade?*
*A: Give rubric to GPT-4, it scores 1-10, explains.*

**Card 4: Golden Set**
*Q: Size?*
*A: 50-200 examples of perfect Q&A.*

**Card 5: Baseline**
*Q: Why compare to baseline?*
*A: Know if improvement is real or lucky.*

**Card 6: Alert**
*Q: When alert on metric drop?*
*A: Usually 5% drop triggers investigation.*

**Card 7: A/B Testing**
*Q: Min sample size?*
*A: 30-50 examples for statistical confidence.*

**Card 8: Continuous Eval**
*Q: Frequency?*
*A: Sample 10 queries/day, track metrics.*

**Card 9: Cost**
*Q: Evaluate 1000 examples with LLM-judge?*
*A: $10-20 (depends on model).*

**Card 10: Rollback**
*Q: New version is worse. Action?*
*A: Rollback immediately, investigate offline.*

---

## Teach-It-Back Prompt

**Explain evaluation to a non-technical stakeholder:**

"We built an AI support bot. It seems good, but is it really? How do we measure?"

**Model Answer:**

"We pick 50 real customer questions. For each, we: (1) run our bot, get answer, (2) grade it 1-10 (does it answer correctly, is it grounded, not making stuff up?), (3) average scores.

If average = 8/10, we're good. If 5/10, something's broken.

We also compare: old bot gets 6/10, new bot gets 8/10. So new is better.

And we keep measuring: every week, sample 10 questions, grade them. If scores suddenly drop, we know something broke and can fix it immediately."

---

## 1-Week Checklist

- [ ] Built golden test set (50+ examples)
- [ ] Installed RAGAS
- [ ] Evaluated RAG system (measured faithfulness, relevancy)
- [ ] Scores acceptable (> 0.80)?
- [ ] Created LLM-as-judge routine
- [ ] Compared base vs fine-tuned on same test set
- [ ] Created monitoring script (tracks scores over time)
- [ ] A/B tested two prompts
- [ ] Winner is statistically significant?
- [ ] Set up alert for metric drop
- [ ] Documented metrics and thresholds
- [ ] Explained scores to team

---

## Resources

- [RAGAS Documentation](https://github.com/explodinggradients/ragas)
- [Evaluating LLMs](https://www.youtube.com/watch?v=r0dGLvs_ycs)
- [LLM Eval Metrics](https://github.com/microsoft/promptflow-evals)

---

## Next Page

You can measure AI quality now.

Next: security. Keep it safe.

→ **[AI Security & Defense →](security.md)**
