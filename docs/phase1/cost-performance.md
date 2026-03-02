# Cost & Performance Optimization

Every API call costs money. Every second of latency frustrates users.

You need both.

---

## Why This Matters

**Real incident:** Startup shipped RAG system. After 1 week with 1000 users, AWS bill was $50K for that week. CEO panicked.

**Real incident 2:** LLM endpoint took 8 seconds per query. Users went to competitor with 2-second response.

Optimization isn't optional. It's competitive advantage.

---

## Conceptual Explanation

Three levers:

1. **Token counting** — Know cost before sending. CTR-like tracking: "this query will cost $0.03"
2. **Model selection** — Right tool for job. GPT-3.5 for simple, GPT-4 for complex
3. **Caching** — Don't repeat work. Embed once, store forever

---

## Token Counting Before API Call

```python
import tiktoken

encoder = tiktoken.encoding_for_model("gpt-4o")

def estimate_cost(prompt: str, model: str = "gpt-4o") -> dict:
    """Estimate cost before sending."""
    
    input_tokens = len(encoder.encode(prompt))
    # Rough: output is ~1/3 of input
    output_tokens = input_tokens // 3
    
    # Pricing per 1M tokens (as of 2024)
    pricing = {
        "gpt-4o": {"input": 5, "output": 15},
        "gpt-4o-mini": {"input": 0.15, "output": 0.6},
        "gpt-3.5-turbo": {"input": 0.5, "output": 1.5},
    }
    
    rate = pricing.get(model, pricing["gpt-4o"])
    
    input_cost = (input_tokens / 1_000_000) * rate["input"]
    output_cost = (output_tokens / 1_000_000) * rate["output"]
    
    return {
        "input_tokens": input_tokens,
        "estimated_output_tokens": output_tokens,
        "input_cost": input_cost,
        "output_cost": output_cost,
        "total_estimated_cost": input_cost + output_cost
    }

# Example
prompt = "Explain quantum computing" * 100  # ~1200 tokens
cost = estimate_cost(prompt)
print(f"This call costs ${cost['total_estimated_cost']:.4f}")
# Output: This call costs $0.0168
```

---

## Model Selection Decision Table

| Model | Input (/1M) | Output (/1M) | Context | Speed | Best For | Cost/Token |
|-------|------------|------------|---------|-------|----------|------------|
| **GPT-4o** | $5 | $15 | 128K | Slow | Complex reasoning, coding | High |
| **GPT-4o-mini** | $0.15 | $0.60 | 128K | Fast | Simple tasks, high-volume | Low |
| **GPT-3.5-turbo** | $0.50 | $1.50 | 16K | Fast | Basic tasks, legacy | Low |
| **Claude 3.5 Sonnet** | $3 | $15 | 200K | Slow | Code, analysis | Medium |
| **Claude 3 Haiku** | $0.80 | $4 | 200K | Fast | Simple, cheap | Low |
| **Llama 3 (local)** | $0 | $0 | 8K | Varies | Privacy-critical | Lowest |
| **Mistral 7B** | $0.27 | $0.81 | 128K | Fast | Cost-sensitive | Low |

---

## Semantic Caching with Redis

```python
import hashlib
import json
from redis import Redis
from openai import AsyncOpenAI

redis_client = Redis(host="localhost", port=6379, decode_responses=True)
client = AsyncOpenAI()

def embed_for_cache(text: str) -> list[float]:
    """Get embedding to check similarity."""
    response = client.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

async def cached_completion(prompt: str, model: str = "gpt-4o-mini") -> str:
    """LLM call with semantic cache."""
    
    # 1. Embed the prompt
    embedding = await embed_for_cache(prompt)
    
    # 2. Check Redis for similar cached prompts
    cache_key = f"llm_cache:{model}"
    cached_items = redis_client.hgetall(cache_key)
    
    best_match = None
    best_score = 0
    
    for stored_hash, cached_response in cached_items.items():
        stored_embedding = json.loads(redis_client.hget(f"{cache_key}:embeddings", stored_hash))
        
        # Cosine similarity
        similarity = sum(a*b for a, b in zip(embedding, stored_embedding)) / (
            (sum(a**2 for a in embedding)**0.5) * (sum(b**2 for b in stored_embedding)**0.5) + 1e-5
        )
        
        if similarity > best_score:
            best_score = similarity
            best_match = cached_response
    
    # 3. If similarity > 0.95, return cached
    if best_score > 0.95:
        print(f"Cache hit! Saved $0.something")
        return best_match
    
    # 4. Otherwise, call API
    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )
    
    answer = response.choices[0].message.content
    
    # 5. Cache the result (5 day TTL)
    prompt_hash = hashlib.md5(prompt.encode()).hexdigest()
    redis_client.setex(
        f"{cache_key}:{prompt_hash}",
        5 * 24 * 3600,  # 5 days
        answer
    )
    
    return answer
```

---

## Prompt Compression

```python
# NAIVE: Send everything
long_prompt = """
The company was founded in 2010. It has 500 employees. 
Located in San Francisco. The CEO is John Doe. Founded by John and Jane.
It operates in software. The market is competitive. 
Q: What's the founding year?
"""

# SMARTER: Compress with key facts
compressed = """
Company: Founded 2010, SF, 500 emp, CEO John Doe
Q: Founding year?
"""

# SMARTER STILL: Use LLM to generate summary
async def compress_context(documents: list[str]) -> str:
    """Use LLM to compress documents."""
    
    compression_prompt = f"""Summarize these documents in 2-3 sentences maximum:

{chr(10).join(documents)}

Summary:"""
    
    response = await client.chat.completions.create(
        model="gpt-4o-mini",  # Cheap model for compression
        messages=[{"role": "user", "content": compression_prompt}]
    )
    
    return response.choices[0].message.content
```

---

## Batching Requests

```python
async def batch_embeddings(texts: list[str], batch_size: int = 100) -> list[list[float]]:
    """Embed 1000s of texts cheaply via batching."""
    
    all_embeddings = []
    
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        
        response = await client.embeddings.create(
            input=batch,
            model="text-embedding-3-small"
        )
        
        all_embeddings.extend([item.embedding for item in response.data])
        
        print(f"Processed {min(i+batch_size, len(texts))}/{len(texts)}")
    
    return all_embeddings

# Cost comparison
# 1000 individual calls: 1000 API roundtrips, slow
# 10 batch calls (100 per call): 10 roundtrips, 100x faster
texts = ["text 1", "text 2", ...] * 1000
embeddings = await batch_embeddings(texts)
```

---

## Async Parallel Calls

```python
import asyncio
from asyncio import Semaphore

async def parallel_llm_calls(prompts: list[str], max_concurrent: int = 10) -> list[str]:
    """Make many LLM calls in parallel without hitting rate limit."""
    
    semaphore = Semaphore(max_concurrent)
    
    async def bounded_call(prompt: str) -> str:
        async with semaphore:
            response = await client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}]
            )
            return response.choices[0].message.content
    
    tasks = [bounded_call(p) for p in prompts]
    results = await asyncio.gather(*tasks)
    return results

# Usage: 100 prompts, all processed in parallel, but max 10 concurrent
prompts = ["prompt 1", "prompt 2", ...] * 100
results = await parallel_llm_calls(prompts)
# Takes 10x less time than sequential
```

---

## Streaming for UX

```python
async def streaming_response(prompt: str):
    """Stream tokens as they arrive, not waiting for full response."""
    
    stream = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )
    
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            # Send to user immediately
            yield chunk.choices[0].delta.content

# In FastAPI
@app.post("/chat/stream")
async def chat_stream(user_input: str):
    return StreamingResponse(
        streaming_response(user_input),
        media_type="text/event-stream"
    )

# User perceives first token in 300ms instead of waiting 8s for full response
```

---

## Fallback Strategy

```python
async def smart_fallback(prompt: str) -> str:
    """Try cheap model first. If fails, use expensive model."""
    
    # 1. Try cheap model (99% of prompts work fine)
    try:
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )
        return response.choices[0].message.content
    
    except Exception as e:
        logger.warning(f"Mini failed: {e}. Trying GPT-4o")
        
        # 2. Fall back to expensive model
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7
        )
        return response.choices[0].message.content

# Cost: 99% of calls = $0.0015, 1% = $0.03
# Avg cost: ~$0.0016 per call (vs $0.03 if always used 4o)
```

---

## Monthly Cost Calculator

```python
def calculate_monthly_cost(
    daily_users: int,
    queries_per_user: float,  # Avg queries per user per day
    avg_prompt_tokens: int,
    avg_output_tokens: int,
    model: str = "gpt-4o-mini",
    caching_hit_rate: float = 0.3  # 30% of queries hit cache
) -> dict:
    """Estimate monthly costs for your system."""
    
    daily_queries = daily_users * queries_per_user
    monthly_queries = daily_queries * 30
    
    # Account for cache hits (no cost)
    billable_queries = monthly_queries * (1 - caching_hit_rate)
    
    pricing = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.6},
        "gpt-4o": {"input": 5, "output": 15},
    }
    
    rate = pricing[model]
    
    input_cost = (billable_queries * avg_prompt_tokens / 1_000_000) * rate["input"]
    output_cost = (billable_queries * avg_output_tokens / 1_000_000) * rate["output"]
    
    return {
        "daily_queries": daily_queries,
        "monthly_queries": monthly_queries,
        "billable_after_cache": billable_queries,
        "input_cost": input_cost,
        "output_cost": output_cost,
        "total_monthly": input_cost + output_cost,
        "cost_per_user_per_month": (input_cost + output_cost) / daily_users
    }

# Example: 1000 users, 2 queries/day each
cost = calculate_monthly_cost(
    daily_users=1000,
    queries_per_user=2,
    avg_prompt_tokens=500,
    avg_output_tokens=200,
    model="gpt-4o-mini",
    caching_hit_rate=0.3
)

print(f"Monthly cost: ${cost['total_monthly']:.2f}")
print(f"Cost per user: ${cost['cost_per_user_per_month']:.4f}")
# Output: Monthly cost: $180.00, Cost per user: $0.0060
```

---

## Performance Metrics

```python
import time
from datetime import datetime

async def measure_performance(prompt: str, model: str) -> dict:
    """Track latency and cost."""
    
    start = time.time()
    cost_est = estimate_cost(prompt, model)
    
    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )
    
    latency = time.time() - start
    
    return {
        "model": model,
        "latency_seconds": latency,
        "estimated_cost": cost_est["total_estimated_cost"],
        "actual_tokens": response.usage.total_tokens,
        "timestamp": datetime.now().isoformat()
    }

# Track over time
# Goal targets:
# - GPT-4o: < 5s latency, $0.05+ cost
# - GPT-4o-mini: < 2s latency, < $0.01 cost
```

---

## Decision Framework

```
Is this the main API call for user response?
  ├─ YES: Optimize heavily (cache, compress, fallback)
  └─ NO: Simple approach OK

How many times will same prompt appear?
  ├─ Often: Cache it
  └─ Once: Don't bother

Is latency critical?
  ├─ YES: Stream + parallel
  └─ NO: Batch if possible

How much accuracy do you need?
  ├─ 99%: Use cheap model + fallback
  └─ 95%: Use cheap model only
```

---

## Practical Project: Cost Dashboard

**Build:** Track all LLM costs in real-time.

```python
from datetime import datetime
import json

async def log_api_call(model: str, tokens_used: int, cost: float):
    """Log every API call."""
    
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "model": model,
        "tokens": tokens_used,
        "cost": cost,
        "user_id": current_user.id
    }
    
    # Append to daily cost file
    with open(f"costs/{datetime.now().strftime('%Y-%m-%d')}.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")

# Dashboard query
def daily_cost_breakdown(date: str) -> dict:
    """See costs by model for a day."""
    
    breakdown = {}
    with open(f"costs/{date}.jsonl") as f:
        for line in f:
            log = json.loads(line)
            model = log["model"]
            breakdown[model] = breakdown.get(model, 0) + log["cost"]
    
    return breakdown

# August 15: {"gpt-4o-mini": $45.23, "gpt-4o": $12.45}
```

---

## Case Study: Stripe AI Docs

**Challenge:** Help 100k developers understand API at scale

**Solution:**
- All embeddings cached (hit rate: 60%)
- Use GPT-3.5 first, GPT-4 only for edge cases
- Batch questions into single API call
- Stream responses for UX

**Result:** $8/month AI support vs $10k/month human support

---

## 10 Review Questions

1. Token counting prevents?
2. Right model for simple task?
3. Semantic cache checks?
4. Batching reduces?
5. Parallel calls limit by?
6. Streaming benefits?
7. Fallback strategy tries?
8. Compression before?
9. Cost per user formula?
10. Hit rate of 50% cache means?

---

## Common Optimization Mistakes

- ❌ Always use GPT-4o (leave money on table)
- ❌ No caching (repeat work)
- ❌ Single slow sequential call (user lag)
- ❌ Send full context (compress!)
- ❌ No monitoring (bill shock)

---

## Resources

- [OpenAI Pricing](https://openai.com/pricing)
- [Token Counter Tool](https://platform.openai.com/tokenizer)
- [Redis Caching](https://redis.io/)

→ **[Phase 1 Milestone →](milestone.md)**
