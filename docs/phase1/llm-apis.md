# LLM APIs & Prompt Engineering

Your app is fast. Your database is optimized. None of it matters if your LLM calls are slow, expensive, or hallucinating.

This is where you stop treating OpenAI/Anthropic as magic black boxes and start using them professionally.

---

## Why This Matters in Production

**Real incident:** Company deployed a support bot using GPT-4 without token counting. They expected 10,000 queries/month, budgeted $500. It went viral internally. First week: $3000 bill. Second month would be $15,000. They didn't even measure quality — just speed.

**Real incident 2:** Fintech startup built a loan application system with an LLM. No evaluation framework. Model decided to hallucinate interest rates. Regulatory audit caught it. System pulled. 6-month project in trash.

**Why this matters:**
- **Cost blows up fast.** One careless API call = $0.05. Multiplied by 1M users = $50,000/day.
- **Your model's wrong answer looks convincing.** Users trust it. You don't catch the error until it hits production.
- **API providers have rate limits.** Your burst of 100 concurrent requests gets throttled. Users see timeouts.
- **LLMs are non-deterministic.** Same input, slightly different output. Caching by exact input doesn't work. You need semantic caching.

You'll learn to:
- Call APIs efficiently (streaming, batching, fallback chains)
- Count tokens before sending (avoid bill shock)
- Design prompts that work reliably
- Handle rate limits and failures
- Measure quality before shipping

---

## Conceptual Explanation: What's Really Happening

**Analogy: Talking to an Expensive Expert**

Think of an LLM API like hiring a brilliant but expensive consultant. You write a question (prompt), they think for a bit, give you an answer. You pay per word they speak and per word they read.

If your question is poorly written, their answer is garbage (prompt engineering).

If you ask them the same question twice, they charge you twice (you need caching).

If you ask them really hard questions, they take longer and cost more (token optimization).

If they give you a wrong answer confidently, you have no way to know except by checking (evaluation).

**Tokens as the Currency**

A token is roughly 4 characters. Not a word. A token.

"Hello, world" = 3 tokens
"The quick brown fox" = 4 tokens
"summarization" = 2 tokens (models break words into pieces)

Why? Because models process text as a sequence of tokens, not characters. Tokens encode meaning more efficiently.

Cost and latency scale with tokens, not words. So you pay for:
- Input tokens (how long is your question + context)
- Output tokens (how long is the answer)

**Message Format**

```
[
  {"role": "system", "content": "You are a helpful assistant"},
  {"role": "user", "content": "Explain AI"},
  {"role": "assistant", "content": "AI is..."},
  {"role": "user", "content": "And ML?"}
]
```

Messages are stateless. Every call includes full history. This is why token counting matters — 10-turn conversation = all 10 messages sent to API.

**Function Calling: Tools for LLMs**

LLMs can read a description of tools and decide to use them.

```
Your tools:
- send_email(to, subject, body)
- search_database(query)
- calculate(expression)

User: "Email John about sales and find Q3 revenue"

LLM thinks: "I need to send an email AND search database"
LLM response: {"type": "tool_use", "tool": "send_email", "args": {...}}
Then: {"type": "tool_use", "tool": "search_database", "args": {...}}
```

You handle the tools, return results, and send back to LLM for final answer.

---

## Code Examples: Real, Working APIs

### OpenAI Chat Completions (Streaming)

```python
import asyncio
import os
from openai import AsyncOpenAI

# Initialize client (reads OPENAI_API_KEY from env)
client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

async def stream_response(prompt: str) -> None:
    """Stream a response token-by-token for real-time UX."""
    # ChatCompletion request with streaming enabled
    stream = await client.chat.completions.create(
        model="gpt-4o",  # Use latest model, cheaper and faster than old GPT-4
        messages=[
            {
                "role": "system",
                "content": "You are a helpful assistant. Be concise."
            },
            {"role": "user", "content": prompt}
        ],
        stream=True,  # Enable streaming: results chunk by chunk
        max_tokens=1000,  # Cap output to avoid bill shock
        temperature=0.7  # 0=deterministic (answers are same), 1=creative (much variation)
    )

    # Process stream chunks as they arrive
    full_response = ""
    async for chunk in stream:
        # Each chunk has delta.content with a piece of the response
        if chunk.choices[0].delta.content:
            token = chunk.choices[0].delta.content
            print(token, end="", flush=True)  # Print as it arrives
            full_response += token
    
    print()  # Newline after stream complete
    return full_response
```

**Why this code:**
- Streaming shows output to user immediately (feels faster)
- `max_tokens=1000` prevents runaway costs
- Async allows other requests while waiting for API
- `temperature=0.7` balances consistency with creativity

### OpenAI Function Calling

```python
import json
from openai import AsyncOpenAI

client = AsyncOpenAI()

# Define tools the LLM can use
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "search_documents",
            "description": "Search your document database for relevant chunks",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query (e.g., 'quarterly revenue')"
                    }
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "send_email",
            "description": "Send an email to a user",
            "parameters": {
                "type": "object",
                "properties": {
                    "to": {"type": "string", "description": "Email address"},
                    "subject": {"type": "string", "description": "Email subject"},
                    "body": {"type": "string", "description": "Email body"}
                },
                "required": ["to", "subject", "body"]
            }
        }
    }
]

# Implement actual tool handlers
def search_documents(query: str) -> list[str]:
    """Placeholder: in production, call RAG pipeline."""
    return [f"Document about {query}", "Another relevant doc"]

def send_email(to: str, subject: str, body: str) -> dict:
    """Placeholder: in production, call email service."""
    return {"status": "sent", "email": to}

async def handle_tool_call(name: str, args: dict) -> str:
    """Execute tool and return result."""
    if name == "search_documents":
        return json.dumps(search_documents(args["query"]))
    elif name == "send_email":
        return json.dumps(send_email(args["to"], args["subject"], args["body"]))
    else:
        return json.dumps({"error": f"Unknown tool: {name}"})

async def agent_loop(user_message: str) -> str:
    """Agent: LLM decides which tools to use, loops until task complete."""
    messages = [
        {
            "role": "system",
            "content": "You are a helpful assistant with access to tools. Use them to help the user."
        },
        {"role": "user", "content": user_message}
    ]
    
    max_iterations = 10
    for iteration in range(max_iterations):
        # Call LLM with available tools
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,  # LLM can choose to use these tools
            tool_choice="auto"  # LLM decides: use tool or return answer
        )
        
        # Check if LLM returned tool calls
        if response.choices[0].finish_reason == "tool_calls":
            # LLM wants to use tools
            assistant_message = response.choices[0].message
            messages.append({"role": "assistant", "content": assistant_message.content or ""})
            
            # Handle each tool call
            for tool_call in assistant_message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                tool_result = await handle_tool_call(tool_name, tool_args)
                
                # Feed result back to LLM for next iteration
                messages.append({
                    "role": "user",
                    "content": json.dumps({
                        "tool": tool_name,
                        "result": tool_result
                    })
                })
        else:
            # LLM returned final answer
            return response.choices[0].message.content

    return "Max iterations reached"

# Usage
answer = await agent_loop("Search for quarterly revenue and email john@company.com with results")
```

**Why this pattern:**
- LLM decides tool use, not you
- Multi-turn: results fed back, LLM can chain calls
- Handles complex workflows without explicit instructions

### Anthropic Claude API (Comparison)

```python
import anthropic

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

async def claude_streaming():
    """Claude uses different API, same concept."""
    # Anthropic with streaming
    with client.messages.stream(
        model="claude-3-5-sonnet-20241022",  # Latest Claude
        max_tokens=1000,
        system="You are a helpful assistant.",  # Claude calls it 'system'
        messages=[
            {"role": "user", "content": "Explain quantum computing"}
        ]
    ) as stream:
        full_response = ""
        for text in stream.text_stream:
            print(text, end="", flush=True)
            full_response += text
        print()
        return full_response
```

**Differences from OpenAI:**
- `system` param instead of message with role `system`
- Uses `messages.stream()` instead of `stream=True` parameter
- Same interface, different naming (expect these in production)

### Token Counting Before Sending

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    """Count tokens for OpenAI models."""
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

def estimate_cost(
    prompt_tokens: int,
    estimated_completion_tokens: int,
    model: str = "gpt-4o"
) -> float:
    """Estimate cost before making API call."""
    # Real costs as of 2024 (check OpenAI pricing for updates)
    prices = {
        "gpt-4o": {"input": 0.005, "output": 0.015},  # per 1K tokens
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015}
    }
    
    rate = prices.get(model, prices["gpt-4o"])
    input_cost = (prompt_tokens / 1000) * rate["input"]
    output_cost = (estimated_completion_tokens / 1000) * rate["output"]
    return input_cost + output_cost

# Example: check cost before calling
prompt = "Summarize this 10,000 word document..."
document = open("doc.txt").read()
full_prompt = f"{prompt}\n\n{document}"

prompt_tokens = count_tokens(full_prompt)
estimated_output = 1000  # You typically estimate completion tokens
estimated_cost = estimate_cost(prompt_tokens, estimated_output)

print(f"This call will cost ~${estimated_cost:.4f}")
# Output: This call will cost ~$0.0850

if estimated_cost > 0.10:
    print("Whoa, that's expensive. Use a cheaper model or chunk the document.")
```

### Structured Output (JSON Schema)

```python
import json
from pydantic import BaseModel
from openai import AsyncOpenAI

client = AsyncOpenAI()

# Define expected output structure
class PersonInfo(BaseModel):
    name: str
    age: int
    occupation: str

async def extract_structured():
    """Force LLM output into specific JSON schema."""
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": "Extract info: John is 30 years old and works as a doctor"
        }],
        # Force LLM to return valid JSON matching schema
        response_format={
            "type": "json_schema",
            "json_schema": {
                "name": "PersonInfo",
                "schema": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "age": {"type": "integer"},
                        "occupation": {"type": "string"}
                    },
                    "required": ["name", "age", "occupation"]
                }
            }
        }
    )
    
    # LLM HAS to return valid JSON or error
    result = json.loads(response.choices[0].message.content)
    return PersonInfo(**result)
```

### Using Instructor for Structured Outputs

```python
import instructor
from pydantic import BaseModel
from openai import AsyncOpenAI

# Patch OpenAI client with instructor
client = instructor.from_openai(AsyncOpenAI())

class DocumentSummary(BaseModel):
    title: str
    key_points: list[str]
    sentiment: str  # positive, negative, neutral

async def extract_with_instructor():
    """Instructor automatically parses LLM output to Pydantic model."""
    doc = "Tesla's Q3 revenue increased 15% YoY. Growth driven by Cybertruck sales..."
    
    summary = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"Summarize: {doc}"
        }],
        response_model=DocumentSummary  # Instructor handles validation
    )
    
    # summary is now a DocumentSummary instance, not raw JSON
    print(f"Title: {summary.title}")
    print(f"Sentiment: {summary.sentiment}")
    return summary
```

---

## Architecture Diagram

```mermaid
graph LR
    A["Your Code"] -->|"prompt + tokens"| B["OpenAI API"]
    B -->|"streaming chunks"| C["Stream Processor"]
    C -->|"buffer + parse"| D["Application"]
    D -->|"cache key"| E["Redis Cache"]
    E -->|"hit/miss"| F["Log & Monitor"]
    F -->|"cost, latency, errors"| G["Dashboard"]
```

---

## Tools Comparison: OpenAI vs Anthropic vs Open Source

| Feature | OpenAI GPT-4o | Claude 3.5 Sonnet | Llama 3.1 70B | Mistral 8x7B |
|---------|---------------|------------------|---------------|-------------|
| **Cost ($/1M input tokens)** | $5 | $3 | Free (self-hosted) | Free (self-hosted) |
| **Context Window** | 128K | 200K | 128K | 32K |
| **Prompt Engineering Sensitivity** | Medium | Low (follows instructions well) | High | High |
| **Function Calling** | Yes (native) | Yes (tool use) | No | No |
| **Streaming** | Yes | Yes | Yes | Yes |
| **Latency** | 500-2000ms | 400-1500ms | 1000-5000ms | 800-3000ms |
| **Best For** | General purpose, tool use | Long context, reasoning | Cost-sensitive, batch | Local deployment |

**Decision:**
- Start with **GPT-4o** (best for prototyping)
- Switch to **Claude** if reasoning > speed or if context window is bottleneck
- Self-host Llama if cost is critical and you can wait for responses

---

## Decision Framework

**Which model should I use?**

```
Is latency critical (sub-100ms)?
  ├─ YES: Use cached response or GPT-4o-mini
  └─ NO: Continue...

Do I need complex reasoning?
  ├─ YES: Claude 3.5 Sonnet
  └─ NO: Continue...

Is cost the primary constraint?
  ├─ YES: Llama 3.1 70B (self-hosted) or GPT-4o-mini
  └─ NO: Continue...

Is context window > 100K tokens?
  ├─ YES: Claude (200K) or Llama (128K)
  └─ NO: Any model works
```

**Which prompt engineering pattern?**

| Problem | Pattern | Why |
|---------|---------|-----|
| Short, factual answer | Simple prompt | Overhead not worth it |
| Multi-step reasoning | Chain-of-thought | "Think step by step" improves logic |
| Extracting structure | JSON schema + instructor | Forces valid output |
| Domain-specific task | Few-shot examples | Model learns from examples |
| Open-ended generation | Low temperature + length limit | Prevents rambling |

---

## Step-by-Step Tutorial: Build a Production LLM Endpoint

### 1. Setup

```bash
pip install openai instructor asyncio python-dotenv
```

### 2. Environment

```bash
# .env
OPENAI_API_KEY=sk-proj-xxx
LOG_LEVEL=info
```

### 3. Async FastAPI Endpoint

```python
# src/myapp/routes/llm.py
from fastapi import APIRouter, HTTPException, Depends
from pydantic import BaseModel
import asyncio
import tiktoken
from openai import AsyncOpenAI, RateLimitError, APIConnectionError
import instructor

router = APIRouter()
client = instructor.from_openai(AsyncOpenAI())

class ChatRequest(BaseModel):
    message: str
    session_id: str  # For conversation history

class ChatResponse(BaseModel):
    response: str
    tokens_used: int
    cost_usd: float

MODEL = "gpt-4o"
MAX_TOKENS = 1000
SYSTEM_PROMPT = "You are a helpful assistant. Be concise and accurate."

def count_tokens(text: str) -> int:
    """Count tokens for cost estimation."""
    encoding = tiktoken.encoding_for_model(MODEL)
    return len(encoding.encode(text))

async def get_conversation_history(session_id: str, db=Depends(...)) -> list:
    """Fetch previous messages for this session."""
    # In production, query your database
    return []

@router.post("/chat", response_model=ChatResponse)
async def chat(
    request: ChatRequest,
    history: list = Depends(get_conversation_history)
) -> ChatResponse:
    """Chat endpoint with streaming, cost estimation, rate limiting."""
    
    # Estimate cost BEFORE calling API
    prompt_tokens = count_tokens(request.message) + sum(
        count_tokens(m.get("content", "")) for m in history
    )
    estimated_completion = 500
    estimated_cost = (prompt_tokens / 1000) * 0.005 + (estimated_completion / 1000) * 0.015
    
    if estimated_cost > 0.50:  # $0.50 per request limit (adjust as needed)
        raise HTTPException(
            status_code=429,
            detail=f"Request too expensive: ${estimated_cost:.2f}"
        )
    
    # Build message list
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT}
    ] + history + [
        {"role": "user", "content": request.message}
    ]
    
    try:
        # Call LLM with retry logic
        for attempt in range(3):  # Retry up to 3 times
            try:
                response = await client.chat.completions.create(
                    model=MODEL,
                    messages=messages,
                    max_tokens=MAX_TOKENS,
                    temperature=0.7
                )
                break
            except RateLimitError:
                if attempt < 2:
                    # Exponential backoff: wait 2^attempt seconds
                    await asyncio.sleep(2 ** attempt)
                else:
                    raise
            except APIConnectionError as e:
                raise HTTPException(status_code=503, detail=f"LLM service unavailable: {str(e)}")
        
        content = response.choices[0].message.content
        
        # Calculate actual tokens and cost
        actual_tokens = count_tokens(content) + prompt_tokens
        actual_cost = (prompt_tokens / 1000) * 0.005 + (
            (actual_tokens - prompt_tokens) / 1000 * 0.015
        )
        
        # Save to conversation history (db operation)
        # _save_to_history(session_id, request.message, content)
        
        return ChatResponse(
            response=content,
            tokens_used=actual_tokens,
            cost_usd=round(actual_cost, 4)
        )
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"LLM error: {str(e)}")
```

**Test it:**
```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is AI?", "session_id": "user123"}'
```

---

## Practical Project: Smart Document Analyzer

**Problem:** Users upload documents. Your system extracts structured insights (key concepts, sentiment, action items).

**Features:**
- Async document processing
- Token counting to avoid bill shock
- Structured output with instructor
- Fallback to cheaper model if expensive model fails
- Cost tracking per document

**Deliverables:**
```
POST /documents/analyze
  Input: {"file_path": "agreement.pdf", "priority": "high"}
  Output: {
    "key_concepts": ["liability", "payment terms"],
    "sentiment": "neutral",
    "action_items": ["review section 3", "sign by date"],
    "cost": 0.02,
    "processing_time": 2.5  # seconds
  }
```

**Requirements:**
- Use GPT-4o for high-priority documents, GPT-4o-mini for low-priority
- Estimate cost, check against budget before calling
- Implement 3-attempt retry with exponential backoff
- Log all API calls with costs
- Return structured output (use instructor)

---

## Debugging Playbook: 10 Common Errors

### 1. `InvalidRequestError: "This model's maximum context length is 4096 tokens"`
**Symptom:** Request fails suddenly, nothing changed.

**Explanation:** You're including too much conversation history. With 10-turn conversations, tokens add up fast.

**Fix:**
```python
def truncate_history(messages: list, max_tokens: int = 3000) -> list:
    """Keep recent messages, drop old ones to stay under token limit."""
    total = count_tokens(str(messages))
    while total > max_tokens and len(messages) > 2:
        messages.pop(0)  # Remove oldest message
        total = count_tokens(str(messages))
    return messages
```

### 2. `RateLimitError: Rate limit reached`
**Symptom:** Works in dev, fails in production. A few users trigger it, everyone gets rate limited.

**Explanation:** You're calling API too fast. OpenAI throttles you.

**Fix:**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)  # 2s, 4s, 8s backoff
)
async def call_llm_with_retry(**kwargs):
    """Retry with exponential backoff."""
    return await client.chat.completions.create(**kwargs)
```

### 3. `json.JSONDecodeError: Invalid JSON from LLM`
**Symptom:** Tried to parse LLM response as JSON, failed.

**Explanation:** LLM hallucinated invalid JSON. Or you didn't enforce `response_format`.

**Fix:**
```python
# Use response_format with json_schema to enforce valid JSON
response = await client.chat.completions.create(
    response_format={
        "type": "json_schema",
        "json_schema": {...}
    }
)
# OR
response = await instructor_client.chat.completions.create(
    response_model=MyModel  # Instructor enforces this
)
```

### 4. `APIConnectionError: Connection refused`
**Symptom:** Works sometimes, fails other times.

**Explanation:** Network is flaky. API is down. Your proxy is misconfigured.

**Fix:**
```python
import httpx

client = AsyncOpenAI(
    http_client=httpx.AsyncClient(timeout=30)  # Add timeout
)

try:
    response = await client.chat.completions.create(...)
except APIConnectionError:
    # Fallback to cached response or simple heuristic
    return cached_response or "Service temporarily unavailable"
```

### 5. `OpenAIError: Invalid API key`
**Symptom:** Authentication fails.

**Explanation:** Key is wrong, expired, or accidentally committed to git.

**Fix:**
```python
import os
if not os.getenv("OPENAI_API_KEY"):
    raise ValueError("Missing OPENAI_API_KEY. Set it in .env")

# Validate key format (should start with sk-)
if not os.getenv("OPENAI_API_KEY").startswith("sk-"):
    raise ValueError("Invalid API key format")
```

### 6. `Hotel California Error: LLM never finishes`
**Symptom:** Request hangs for 30+ seconds.

**Explanation:** LLM is slow. Network is slow. You forgot `timeout`.

**Fix:**
```python
import asyncio

try:
    response = await asyncio.wait_for(
        client.chat.completions.create(...),
        timeout=10.0  # Fail after 10 seconds
    )
except asyncio.TimeoutError:
    return {"response": "Request timed out, try again", "error": True}
```

### 7. `TypeError: 'str' object is not iterable`
**Symptom:** Error when streaming.

**Explanation:** You're iterating over stream wrong, or stream was already consumed.

**Fix:**
```python
async for chunk in stream:
    # Process chunks ONE time, don't nest loops
    if chunk.choices[0].delta.content:
        content = chunk.choices[0].delta.content

# DON'T do async for chunk in stream: twice!
```

### 8. `This request exceeds token budget`
**Symptom:** Your own safeguard fails the request.

**Explanation:** Document is too large. You counted wrong. Max token buffer is hit.

**Fix:**
```python
if prompt_tokens > 100000:  # Your max context
    return {
        "error": "Document too large",
        "max_tokens": 100000,
        "actual_tokens": prompt_tokens,
        "suggestion": "Split document into chunks"
    }
```

### 9. `Hallucination: Model confidently gave wrong answer`
**Symptom:** User reports incorrect information from your LLM.

**Explanation:** LLM invented information not in the context.

**Fix:**
```python
# In system prompt, be explicit
SYSTEM_PROMPT = """You are a helpful assistant. IMPORTANT:
Only answer based on provided context. If information is not in the context,
say "I don't have information about that" rather than guessing."""
```

### 10. `Silent Cost Overrun: Expensive model was used`
**Symptom:** Monthly bill is 10x higher than expected.

**Explanation:** You switched models, added a new endpoint, or didn't enforce cost limits.

**Fix:**
```python
# Always estimate BEFORE calling
from openai import AsyncOpenAI
import tiktoken

async def safe_llm_call(messages, model="gpt-4o", max_cost=0.05):
    """Always check cost before API call."""
    prompt_tokens = count_tokens(json.dumps(messages))
    estimated_completion = len(messages) * 100  # Rough estimate
    
    costs = {
        "gpt-4o": (0.005, 0.015),
        "gpt-4o-mini": (0.00015, 0.0006)
    }
    input_rate, output_rate = costs[model]
    estimated_cost = (prompt_tokens / 1000 * input_rate) + \
                     (estimated_completion / 1000 * output_rate)
    
    if estimated_cost > max_cost:
        # Use cheaper model instead
        return await safe_llm_call(messages, model="gpt-4o-mini", max_cost=max_cost)
    
    return await client.chat.completions.create(
        model=model,
        messages=messages
    )
```

---

## Common Mistakes

!!! danger "Mistake 1: Not Counting Tokens"

You think "that's only 5 requests, can't be expensive." But each request includes full conversation history. Actually 500 requests. $50 unexpected charge.

**Don't:** Send requests blindly.
**Do:** Count tokens, estimate cost, check before sending.

!!! danger "Mistake 2: Parsing LLM Output Like It's Deterministic"

You check `if "yes" in response.lower()`. LLM says "Yes, I agree" one day and "Certainly, I concur" the next. Your parser misses it.

**Don't:** Substring matching on LLM output.
**Do:** Use `response_format` + instructor for structured output.

!!! danger "Mistake 3: No Retry Loop"

API fails once (network hiccup), you crash. Retry loop with exponential backoff costs nothing and fixes 99% of failures.

**Don't:** Assume API never fails.
**Do:** Retry with backoff:
```python
from tenacity import retry, stop_after_attempt, wait_exponential
```

!!! danger "Mistake 4: Including Secrets in Logs"

You log the full API response for debugging. It contains a user's API key they just shared. Logs are readable by everyone. Security incident.

**Don't:** Log full messages.
**Do:** Log only safe fields (tokens, cost, latency). Redact PII.

!!! danger "Mistake 5: Not Handling Streaming Properly"

You start streaming response to user, but error happens mid-stream. User sees partial response then nothing.

**Don't:** Stream without error handling.
**Do:** Catch errors, return fallback message:
```python
try:
    async for chunk in stream:
        yield chunk
except Exception as e:
    yield f"\n\n[Error: {str(e)}]"
```

---

## Production Realism: Tutorial vs Production Code

**Tutorial Code:**
```python
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response["choices"][0]["message"]["content"])
```

**Production Code:**
```python
import asyncio
import tiktoken
from openai import AsyncOpenAI, RateLimitError
from tenacity import retry, stop_after_attempt, wait_exponential

client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
encoding = tiktoken.encoding_for_model("gpt-4o")

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2))
async def call_llm(messages: list, max_cost: float = 0.10) -> str:
    """Thread-safe, cost-limited LLM call with retry logic."""
    
    # 1. Count tokens BEFORE making request
    prompt_tokens = sum(
        len(encoding.encode(msg.get("content", ""))) 
        for msg in messages
    )
    if prompt_tokens > 120000:  # Context window safety
        raise ValueError(f"Prompt too large: {prompt_tokens} tokens")
    
    # 2. Estimate cost
    estimated_output = 500
    estimated_cost = (prompt_tokens / 1000 * 0.005) + (estimated_output / 1000 * 0.015)
    if estimated_cost > max_cost:
        raise ValueError(f"Cost estimate ${estimated_cost} exceeds limit ${max_cost}")
    
    # 3. Call with timeout
    try:
        response = await asyncio.wait_for(
            client.chat.completions.create(
                model="gpt-4o",
                messages=messages,
                max_tokens=1000,
                temperature=0.7
            ),
            timeout=10.0
        )
    except asyncio.TimeoutError:
        raise TimeoutError("LLM API timeout after 10s")
    
    # 4. Log safe fields (not token content due to PII)
    output_tokens = len(encoding.encode(response.choices[0].message.content))
    actual_cost = (prompt_tokens / 1000 * 0.005) + (output_tokens / 1000 * 0.015)
    
    logger.info(
        "LLM_CALL",
        tokens=output_tokens,
        cost=actual_cost,
        latency=response.response_ms  # Not all responses have this
    )
    
    return response.choices[0].message.content
```

**Differences:**
- Tutorial: 3 lines, zero error handling
- Production: 40 lines, handles cost, timeouts, retries, logging
- Tutorial fails silently or explosively
- Production fails gracefully with useful error messages

---

## Cost & Performance Considerations

**Token Counting Overhead:**
- tiktoken.encode(): ~0.2ms for 1000 tokens
- Multiply by 1000 requests = 200ms added latency
- Worth it to avoid $1000 bill surprises

**Streaming Benefits:**
- User sees first token in 300ms instead of waiting 3s for full response
- Perceived latency drops 80%
- Token usage same, just better UX

**Batch API (OpenAI):**
- Process 1000 documents at 50% discount
- Results come next day (async)
- Use for non-urgent work (overnight indexing)

**Model Cost Ranking:**
1. `gpt-3.5-turbo`: $0.0005/1K input (cheapest, weakest)
2. `gpt-4o-mini`: $0.00015/1K input (good value)
3. `gpt-4o`: $0.005/1K input (5x cost, much better reasoning)
4. `Claude 3 Opus`: $0.015/1K input (most expensive, best reasoning)

**Token Counting Math:**
```
1 page of text ≈ 500 tokens
1000 requests × 500 tokens × $0.001 = $500
(All-in cost including output)
```

---

## Security Considerations

**Never log full messages.** They may contain:
- Customer secrets
- API keys
- Personal information
- Proprietary data

**Redact before logging:**
```python
def redact_pii(message: str) -> str:
    """Remove sensitive info from logs."""
    import re
    # Redact email
    message = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2}\b', '[EMAIL]', message)
    # Redact phone
    message = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE]', message)
    # Redact API key (starts with sk-)
    message = re.sub(r'sk-[A-Za-z0-9]{20,}', '[API_KEY]', message)
    return message

logger.info(f"Request: {redact_pii(user_message)}")
```

**API Key Rotation:**
- Never hardcode keys
- Use environment variables
- Rotate keys every 90 days
- Immediate rotation if exposed

**Rate Limiting Per User:**
```python
import redis

redis_client = redis.Redis()

async def is_rate_limited(user_id: str, max_requests: int = 100) -> bool:
    """Check if user exceeded rate limit."""
    key = f"llm_requests:{user_id}"
    count = redis_client.incr(key)
    if count == 1:
        redis_client.expire(key, 86400)  # Reset daily
    return count > max_requests
```

---

## Case Study 1: Discord Moderation Bot

**Problem:** Discord community has 500k users. Manually reviewing reports takes 2 weeks. They need AI moderation.

**Solution:**
- OpenAI API with GPT-4o
- Token counting to budget $500/month
- Semantic cache in Redis to avoid reprocessing same reports
- Fallback to previous decision if API fails

**Results:**
- Review time dropped from 2 weeks to 20 minutes
- Cost stayed under budget ($400/month)
- Hallucination rate: 2% (acceptable for "flag for human review")

**Key insight:** Streaming not needed (human review anyway). Semantic caching reduced cost 70%.

---

## Case Study 2: Legal Document Analysis Startup

**Problem:** They wanted AI to extract key terms from contracts. Lawyers to verify.

**Solution:**
- Claude 3.5 Sonnet (better reasoning than GPT-4o for legal)
- Structured output with instructor (extract specific fields)
- RAGAS evaluation (measure if extracted terms are correct)
- Fine-tuning later (no, they decided prompt engineering was enough)

**Results:**
- Extracted terms with 92% accuracy
- Cost: $0.08 per document
- Lawyers verify in 2 minutes instead of 30 (10x speedup)

**Key insight:** Better prompts sometimes beat fine-tuning. They tested both, prompts won.

---

## Interview Cheat Sheet

**Concept 1: Token Counting**

*Question: "Why does token counting matter?"*

Model answer: "Tokens are the billing unit for LLM APIs. To budget accurately and avoid surprise bills, you must count tokens in the prompt + estimated completion tokens before making the API call. Tokens != words; `"summarization"` is 2 tokens even though it's 1 word. Using tiktoken library, you can estimate cost before committing."

**Concept 2: Streaming**

*Question: "When should I use streaming?"*

Model answer: "Stream when latency matters and user is waiting (chat, real-time generation, search results). Don't stream when processing background jobs (batch indexing, overnight analysis). Streaming doesn't reduce total tokens or cost, but shows output immediately, making perceived latency drop from 3s to 300ms."

**Concept 3: Function Calling**

*Question: "How does function calling work?"*

Model answer: "You define tools (functions) with name, description, and parameters. LLM reads the definitions and decides to call them. You execute the tool, return results, and send back to LLM for next iteration. This lets LLM decide workflow without explicit if/else logic. Used for agents, RAG retrieval, API integration."

**Concept 4: Prompt Engineering**

*Question: "What's the most important prompt engineering pattern?"*

Model answer: "System prompt. It sets the role and constraints. Good system prompt: 'You are a helpful assistant. Output JSON only. Be concise. Refuse to discuss X.' Bad system prompt: 'You are helpful.' Only add few-shots if they improve results (test both). Only use chain-of-thought if reasoning is required."

**Concept 5: Cost Optimization**

*Question: "How do you optimize LLM costs?"*

Model answer: "Three levers: (1) model selection: use GPT-4o-mini instead of GPT-4o if possible (33x cheaper). (2) Prompt size: compress or summarize context. (3) Caching: don't call API twice for same request. Semantic cache: similar queries hit cache. Monitor costs in production; many startups get surprised bills."

---

## 10 Review Questions

1. What's the difference between input tokens and output tokens? Why do they have different costs?
2. You're processing 1000 documents. Same question repeated 500 times. How would you optimize?
3. OpenAI API times out. What's your retry strategy?
4. Your LLM says "yes" and "correct" and "absolutely" for the same question. How do you parse it?
5. Function calling vs string parsing. When is each better?
6. You have 10-turn conversation. Next request fails (context too large). Solution?
7. What does `response_format=json_schema` do? When should you use it?
8. Streaming vs non-streaming. Which is faster, cheaper, more reliable?
9. Your monthly bill doubled. How do you debug?
10. GPT-4o vs GPT-4o-mini. When choose each?

---

## Flashcards

**Card 1: Tokens**
*Q: What's a token?*
*A: Roughly 4 characters. Not a word. Models process text as tokens. 1000 tokens ~= 250 words. Cost scales with tokens.*

**Card 2: Context Window**
*Q: What's context window?*
*A: Max tokens the model can accept as input. GPT-4o: 128K. Claude: 200K. If prompt exceeds window, request fails.*

**Card 3: Temperature**
*Q: Temperature 0 vs 1?*
*A: Temperature 0 = deterministic (same answer every time). Temperature 1 = creative (different every time). Use 0 for facts, retrievals. Use 1 for brainstorming.*

**Card 4: Streaming**
*Q: Why stream?*
*A: Show output immediately instead of waiting for full response. Doesn't reduce latency overall, but first token visible in 300ms vs 3s for full response.*

**Card 5: Fallback**
*Q: How do you handle API failure?*
*A: Try expensive model → cheaper model → cache → static response. Each layer is fallback.*

**Card 6: Cost**
*Q: How do you prevent bill shock?*
*A: Count tokens before API call. Estimate cost. Set max limit. Fail if exceeds. Use cheaper model if possible.*

**Card 7: Function Calling**
*Q: LLM decides tool use. Can it call multiple tools in one response?*
*A: Yes, `response.choices[0].message.tool_calls` is a list. Loop through each, execute, return results.*

**Card 8: System Prompt**
*Q: What goes in system prompt?*
*A: Role, constraints, output format. Example: "You are a customer support bot. Output JSON. Never discuss politics."*

**Card 9: Retry Logic**
*Q: How many retries? How long to wait?*
*A: 3 retries. Exponential backoff: 2s, 4s, 8s. Formula: `wait = 2 ** attempt`.*

**Card 10: Structured Output**
*Q: Tutorial parsing vs production parsing?*
*A: Tutorial: `json.loads(response)` → errors if invalid JSON. Production: instructor + Pydantic → enforces schema, retries on failure.*

---

## Teach-It-Back Prompt

**Explain this like you're teaching a junior engineer:**

"You need to call OpenAI API from your FastAPI app. Design the system: what happens if API is slow? What if it's expensive? What stops a bad actor from making 1000 requests? Walk me through your error handling, cost controls, and monitoring."

**Model Answer:**

"I'd build a wrapper function called `safe_llm_call()`. Inside:

1. **Cost estimation:** Count tokens before API call. Estimate output tokens. Check if cost exceeds per-request limit. Reject if over budget.

2. **Retry logic:** Wrap in `@retry` decorator with exponential backoff (3 attempts, max 10 second wait). Catches rate limit and connection errors.

3. **Timeout:** Use `asyncio.wait_for()` with 10 second limit. If API hangs, fail fast.

4. **Circuit breaker:** Track failures over time (e.g., last 5 requests failed). Stop trying API temporarily. Return cached response instead.

5. **Rate limiting per user:** Redis counter for each user. Max 100 requests/day. Tracked with `user_id:rate_limit` key.

6. **Logging:** Log latency, tokens, cost, errors. Never log full message content (PII). Use redaction.

7. **Fallback chain:** If GPT-4o fails, try GPT-4o-mini. If that fails, return cached response. Last resort: tell user service is down.

All of this happens transparently. Caller just calls `await safe_llm_call(messages)`. Either gets result or clear error."

---

## 1-Week Review Checklist

- [ ] Built a ChatCompletion endpoint that counts tokens and estimates cost
- [ ] Implemented retry loop with exponential backoff for API failures
- [ ] Created function calling example (define tools, execute, return results)
- [ ] Tested rate limiting (called API 200 times fast, verified throttle kicks in)
- [ ] Used streaming in a FastAPI endpoint, showed output to user in real-time
- [ ] Implemented semantic caching: hash the prompt, check Redis before API
- [ ] Tested: what happens if API fails mid-stream? Can user see partial response?
- [ ] Refactored prompts: made system prompt explicit (role + constraints)
- [ ] Measured cost of 100 API calls. Know monthly projection.
- [ ] Set up `.env` with API key. Verified key is NOT in git.
- [ ] Built a fallback chain: expensive model → cheap model → cache → error
- [ ] Tested: API returns invalid JSON. Your code handles it gracefully.

---

## Resources

- [OpenAI API Documentation](https://platform.openai.com/docs/)
- [OpenAI Python Library](https://github.com/openai/openai-python)
- [Anthropic Claude API](https://console.anthropic.com/)
- [Tiktoken (token counter)](https://github.com/openai/tiktoken)
- [Instructor (structured outputs)](https://github.com/jxnl/instructor)
- [Tenacity (retry library)](https://github.com/jmoiron/tenacity)
- [OpenAI Cookbook (examples)](https://github.com/openai/openai-cookbook)

---

## Next Page

You now know how to call LLMs and handle them reliably.

Next: embeddings. You'll learn how to convert text to numbers and do semantic search.

→ **[Embeddings & Vector Search →](embeddings.md)**
