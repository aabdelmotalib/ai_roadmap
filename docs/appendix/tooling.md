# Tooling and Productivity for AI Engineers

The best tool is the one you'll actually use.

This guide covers the tools and workflows that let you move fast without sacrificing quality. Not just "what tools exist," but "how to use them effectively in actual work."

---

## 1. Using AI Assistants for Coding: Become a Senior Pair Programmer

Claude and ChatGPT are not code machines. They're thinking partners.

The difference between "AI writes all my code" (you learn nothing) and "AI helps me solve hard problems" (you become faster and smarter) is how you prompt them.

### The Right Mindset

!!! note "AI Assistant Anti-Pattern"
    User: "Write me a RAG system"
    Claude: [500 lines of code]
    User: [copies, pastes, doesn't understand, code breaks]
    
    This is how you learn nothing and the code is bad.

!!! tip "AI Assistant The Right Way"
    User: "I'm building RAG. Here's my problem: embeddings are fast but retrieval gets slow at 100K vectors. I tried [X], [Y], still slow. What are the most likely causes?"
    Claude: [explains bottleneck analysis, suggests causes, helps you investigate]
    User: [understands the tradeoff, fixes efficiently, learns something real]

---

### Prompt Pattern 1: Debugging (My Favorite)

**Situation:** Something is broken. You're stuck.

**Bad prompt:**
```
My code doesn't work. Fix it.
```

**Good prompt:**
```
[Include full error message]
[Include relevant code (20-50 lines)]
[Include what you've already tried]
[Include your hypothesis about what's wrong]

What are the most likely causes? What should I check first?
```

**Example (real case):**
```
Error: "CUDA out of memory" on A100 with 80GB VRAM

Batch size: 32
Sequence length: 2048
Model: mistral-7b with LoRA (rank=16)

Code:
```python
model = AutoModelForCausalLM.from_pretrained(
    "mistral-7b",
    torch_dtype=torch.float16,
    device_map="auto"
)
peft_model = get_peft_model(model, lora_config)
for batch in dataloader:
    outputs = peft_model(**batch)  # Dies here
```

I've tried:
- Gradient accumulation (doesn't help, limit is inference)
- Half precision (already using float16)
- Smaller batch size (48GB → 32GB used, still okay, but 64Gb → OOM)

My hypothesis: batch size 64 with sequence length 2048 = too many attention tokens at once. 
64 * 2048 * 2048 = 268M attention matrix. Should fit, but maybe overhead?

What am I missing?
```

**Claude's response will be:**
- Attention memory calculation (confirms your math)
- KV cache size (ah! that's the hidden cost)
- Specific fix (use flash-attention, reduce seq_len, or smaller batch)
- Why each matters

You learn the actual bottleneck. You fix it properly. Next time, you won't even need to ask.

---

### Prompt Pattern 2: Code Review

**Purpose:** Get a critical pair's perspective before committing.

```
Review this code for production readiness:
- Production concerns (will this work at scale?)
- Security issues (any vulnerabilities?)
- Performance (obvious inefficiencies?)
- Edge cases (what breaks?)
- Maintainability (will others understand this 6 months from now?)

Be specific and critical. Point out exact lines.

[Include code]
```

**Real example:**
```
Review this embedding caching layer:

```python
def cache_embedding(query: str, embedding: list[float]):
    key = query.lower()  # Normalize
    cache[key] = embedding
    return cache[key]

def get_from_cache(query: str) -> Optional[list[float]]:
    key = query.lower()
    return cache.get(key, None)
```

Edge cases I'm worried about:
- What if query is longer than expected?
- Does lower() break for non-English?
- Cache unbounded memory leak?
```

**Claude will catch:**
- Security: injection via memcached if you use external cache
- Edge case: case-folding breaks some languages (Turkish ı/I)
- Scaling: unbounded cache = eventual OOM
- Performance: .lower() is fine, but dict lookup is O(1) only if key is hashable
- Suggestion: add TTL, max size, semantic similarity threshold

You get a checklist of real concerns. You fix them before they're prod issues.

---

### Prompt Pattern 3: Architecture Design

```
I need to design [system]. 
Constraints: [specific numbers: 1000 QPS, <500ms latency, <$1000/month]
Trade-offs I'm willing to make: [speed > cost, or accuracy > latency]

What are 3 different architecture approaches with pros/cons for each?
```

**Real example:**
```
Design a semantic search over 10M documents.
Constraints:
- <500ms P99 latency
- Cold start must complete in <5 minutes
- <$500/month operating cost
- Can re-index once weekly

I'm willing to trade: slightly lower relevance for cheaper embed model
```

**Claude will suggest:**
1. Pinecone (fully managed, fast, expensive)
   - Pros: 0 ops, guaranteed latency
   - Cons: $8K/month at scale
2. Elasticsearch (hybrid search, cheaper)
   - Pros: full control, half the cost
   - Cons: you manage infrastructure
3. Qdrant self-hosted (vector-native, cheapest)
   - Pros: complete control, $200/month
   - Cons: you run/scale/backup yourself

For YOUR constraints (budget-conscious), option 3 makes sense.

You make an informed choice instead of copying a tutorial.

---

### Prompt Pattern 4: Explanation with Analogies

Use when you understand *something* but not the deep intuition.

```
Explain [concept] as if I understand [related concept].
Use an analogy from [domain I know].

For example: explain attention like it's [domain thing]
```

**Real example:**
```
Explain the difference between embedding fine-tuning and adapter layers (LoRA).

I understand fine-tuning for classification models already.

Use an analogy from music or finance (pick whichever makes more sense).
```

**Claude will explain:**
- Fine-tuning: "Like rearranging every word in an orchestra, changes everything"
- LoRA: "Like bringing in one extra musician (the adapter), changes only slightly"
- When to use: LoRA if you want to preserve knowledge, fine-tuning if you want specialization

Now you have the *intuition* not just the technique.

---

### Prompt Pattern 5: Documentation

Your code should explain *why* not *what*.

```
Write docstrings for this function. Focus on WHY we do each step, not WHAT the code does.
Include:
- Why we chose this algorithm (vs alternatives)
- What assumptions we're making
- When this function isn't appropriate
- Performance characteristics

[Include function]
```

**Before:**
```python
def chunk_text(text: str, size: int = 512) -> list[str]:
    """Split text into chunks."""
    return [text[i:i+size] for i in range(0, len(text), size)]
```

**After (what Claude suggests):**
```python
def chunk_text(text: str, size: int = 512, overlap: int = 0) -> list[str]:
    """
    Split text into fixed-size chunks for embedding or RAG.
    
    Uses fixed-size chunking, not sentence-aware splitting.
    This was chosen because:
    - Simple and deterministic (same chunks every time)
    - Consistent token counts (better for batching)
    - Faster than sentence boundaries (no NLP parsing)
    
    Tradeoff: may cut sentences mid-way. If you need semantic chunks, 
    use sentence_split() instead (slower but higher quality).
    
    Args:
        text: Text to chunk
        size: Chunk size in characters (adjust if using tokens; 1 token ≈ 4 chars)
        overlap: Characters to overlap between chunks (useful for RAG to preserve context)
        
    Returns:
        List of text chunks
        
    Performance: O(n) where n = len(text). Very fast.
    
    Limitations:
    - Doesn't break on sentence boundaries (may cut mid-sentence)
    - Overlap increases return list size (10% overlap = 10% more overlap chunks)
    
    Example:
        >>> chunks = chunk_text("Hello world. This is AI.", size=10, overlap=2)
        >>> chunks
        ['Hello worl', 'worldd. Th', 'This is AI']
    """
```

The second version is maintainable. Future you will thank you.

---

### Anti-Patterns: What NOT to Do

!!! warning "Don't: Accept First Answer Without Question"
    Claude gives you code. You test it. It works. You're done.
    
    Better: Test it, read it, understand it. Does it handle edge cases?
    Is there a better way? You should be able to explain every line.

!!! warning "Don't: Use Code You Don't Understand"
    "This code works, I don't need to understand it."
    
    When it breaks (and it will), you're helpless. You're a copy-paste engineer.
    Spend 10 minutes understanding it. Then you own it.

!!! warning "Don't: Let AI Write All Your Code"
    If Claude writes 100% of your code, you're learning 0%.
    
    Better workflow: You write 70%, Claude helps with the hard 30%.

!!! warning "Don't: Ask Yes/No Questions"
    "Is this code secure?" → Claude might say yes when it's not.
    
    Better: "Review for security issues. Point out specific vulnerabilities."
    Claude has to *find* something, not just agree with you.

---

## 2. Reading Documentation Efficiently: The 5-Step Method

Most engineers waste hours reading docs wrong.

### The 5 Steps

**1. Scan the README/Overview (2 minutes)**
- What problem does this solve?
- What type of tool is it? (framework, library, service, database)
- Do I actually need this, or is there a better option?

Example: "FastAPI is a web framework for APIs in Python. Good for production. Alternative: Flask." Done.

**2. Run the quickstart (5 minutes)**
- Copy the first example
- Run it
- Make sure it works
- Don't read the explanation, just run it

```bash
# Installation
pip install fastapi uvicorn

# Create main.py
from fastapi import FastAPI
app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello"}

# Run
uvicorn main.py:app --reload
```

You now have working proof before learning *why* it works. This matters.

**3. Find the API Reference (10 seconds)**
- Ctrl+F "API reference" or "API docs"
- Note where it is
- Don't read it thoroughly (too boring)
- Know it exists so you can search it later

You'll be back here 50 times. "What's the parameter name? What's the exact behavior?" Reference docs answer this in 30 seconds.

**4. Search for Your Specific Use Case (5-10 minutes)**
- Ctrl+F "streaming"
- Ctrl+F "batch"
- Ctrl+F "async"
- Find examples that match your need
- Copy example code, don't read explanations

Skip the narrative sections. You don't need "Chapter 4: Async Basics." You need "How do I stream responses in FastAPI?"

**5. Read the "How It Works" Section (10-20 minutes)**
- Now that you have the what, understand the why
- Especially useful for architecture understanding
- Read this AFTER you have working code, not before

---

### Reading Research Papers (Without a PhD)

You'll need to read papers. No way around it.

Real researchers read papers like this:
- Abstract (does this matter to me?) → 30 seconds
- Figures and tables (what did they build and how well did it work?) → 2 minutes
- Conclusion (what's the takeaway?) → 1 minute
- If it matters: Intro (what problem did they solve?) → 2 minutes
- Skip unless directly implementing: Methodology section (the heavy math) → skip it

Total: 5 minutes to understand 90% of what matters.

!!! note "Real Example: Reading a RAG Paper"
    Paper title: "Self-RAG: Learning to Retrieve, Generate, and Critique for Self-Improved Generation"
    
    **Abstract:** "Self-RAG learns when to retrieve and how to use retrieved documents"
    Decision: Matters? Yes, relevant to RAG improvements.
    
    **Figures:** Figure 1 shows retriever, generator, critic components
    Understanding: Train a model to decide "should I retrieve?" and "is this generation good?"
    
    **Results table:** Self-RAG achieves 75.3% on benchmark (vs 72.1% baseline)
    Takeaway: 3% improvement by adding critique module
    
    **Conclusion:** "Models learn to retrieve only when needed"
    Key insight: Retrieval every time is wasteful; learn when to retrieve
    
    **Done.** You understand the paper in 5 minutes.
    
    You don't need the methodology (math). You need the idea and results.

---

## 3. Git Workflow for Solo AI Projects

Clean git history is a form of respect for future you.

### Branch Naming Convention

```
feature/[what-you-added]
fix/[what-you-fixed]
experiment/[what-you-tried]
refactor/[what-you-reorganized]
docs/[what-you-documented]
```

**Examples:**
```
feature/add-semantic-caching-layer
fix/handle-openai-rate-limits
experiment/try-bge-reranker
refactor/extract-embedding-logic-to-module
docs/add-architecture-diagram
```

### Commit Message Format: Conventional Commits

```
<type>(<scope>): <description>
```

**Types:**
- `feat`: A new feature (RAG retrieval function)
- `fix`: Bug fix (handle timeout errors)
- `refactor`: Code restructuring (no functionality change)
- `test`: Adding tests (new test suite)
- `docs`: Documentation (README, docstrings)
- `perf`: Performance improvement (optimize embedding query)
- `chore`: Maintenance (update dependencies)

**Examples:**
```
feat(rag): add semantic caching with Redis backend
fix(api): handle OpenAI 429 rate limit error gracefully
refactor(embedding): extract chunking strategy to separate module
test(evaluation): add RAGAS faithfulness test suite
docs(readme): add architecture diagram and setup instructions
perf(inference): reduce latency by 40% with batch processing
```

### The Workflow

```bash
# Start new feature
git checkout -b feature/your-feature
git add .
git commit -m "feat(rag): add retrieval functionality"

# Multiple commits are fine during development
git add .
git commit -m "fix(rag): handle empty query edge case"

git add .
git commit -m "test(rag): add unit tests for retrieval"

# Before merging, clean up: squash related commits
git rebase -i main  # Interactive rebase

# In the editor, mark commits as 'squash' to combine them
# Result: clean linear history
pick abc1234 feat(rag): add retrieval functionality
squash def5678 fix(rag): handle empty query edge case
squash ghi9012 test(rag): add unit tests

# Final commit message:
feat(rag): add retrieval with tests and edge case handling

# Merge
git checkout main
git merge feature/your-feature
```

**Result:** Clean history. When you read the log 6 months later, you see the logic of development, not implementation noise.

### .gitignore for AI Projects

```bash
# Environment
.env
.env.local
.venv/
venv/
env/

# Model files (large, don't commit)
models/
*.pkl
*.pt
*.pth
*.safetensors
*.onnx
*.bin

# Data (large, don't commit)
data/raw/
data/processed/
*.csv
*.parquet
*.json.gz

# Training artifacts
runs/
wandb/
mlruns/
outputs/
checkpoints/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Python
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/

# Jupyter
.ipynb_checkpoints/
*.ipynb_checkpoints

# OS
.DS_Store
Thumbs.db
```

---

## 4. Jupyter Notebooks: Exploration → Production

**Use notebooks for:** Exploration, data analysis, one-time scripts, visualization
**Never use notebooks for:** Production APIs, reusable libraries, version control nightmare code

### The Refactoring Flow

```
1. Notebook: Explore and experiment
2. Extract functions into notebook cells
3. Move cells into Python module (src/module.py)
4. Add tests (tests/test_module.py)
5. Integrate into main app (main.py imports your module)
```

**Real example:**

**Step 1: Notebook (exploration)**
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

# Experiment with chunking
text = """Long document..."""
chunks = [text[i:i+512] for i in range(0, len(text), 512)]

embeddings = model.encode(chunks)
print(embeddings.shape)  # (5, 384)

# Test similarity
from sklearn.metrics.pairwise import cosine_similarity
sim = cosine_similarity(embeddings)
```

**Step 2: Extract to module (src/embedding.py)**
```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity

class EmbeddingModel:
    def __init__(self, model_name: str = 'all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
    
    def chunk_text(self, text: str, chunk_size: int = 512) -> list[str]:
        """Split text into fixed-size chunks."""
        return [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
    
    def embed(self, texts: list[str]) -> list[list[float]]:
        """Embed a list of texts."""
        return self.model.encode(texts).tolist()
    
    def similarity(self, embedding1, embedding2) -> float:
        """Cosine similarity between two embeddings."""
        return cosine_similarity([embedding1], [embedding2])[0][0]
```

**Step 3: Tests (tests/test_embedding.py)**
```python
import pytest
from src.embedding import EmbeddingModel

def test_chunk_text():
    model = EmbeddingModel()
    text = "a" * 1000
    chunks = model.chunk_text(text, chunk_size=100)
    assert len(chunks) == 10
    assert len(chunks[0]) == 100

def test_embed():
    model = EmbeddingModel()
    texts = ["hello world", "goodbye world"]
    embeddings = model.embed(texts)
    assert len(embeddings) == 2
    assert len(embeddings[0]) == 384  # Model dimension

def test_similarity():
    model = EmbeddingModel()
    text = "hello world"
    embedding = model.embed([text])[0]
    # Same text should have similarity ~1.0
    similarity = model.similarity(embedding, embedding)
    assert 0.99 < similarity <= 1.0
```

**Step 4: Integration (main.py)**
```python
from fastapi import FastAPI
from src.embedding import EmbeddingModel

app = FastAPI()
embedding_model = EmbeddingModel()

@app.post("/embed")
def embed_endpoint(text: str):
    chunks = embedding_model.chunk_text(text)
    embeddings = embedding_model.embed(chunks)
    return {"chunks": chunks, "embeddings": embeddings}
```

---

### Converting Notebooks to HTML Docs

Share your experimentation! Use `nbconvert`:

```bash
pip install nbconvert
jupyter nbconvert --to html my_notebook.ipynb

# Output: my_notebook.html (can view in browser)
```

Great for documentation, blog posts, and sharing findings.

### Clean Notebooks Before Git Commit

Notebooks with output take massive space. Use `nbstripout`:

```bash
pip install nbstripout
nbstripout my_notebook.ipynb  # Removes all outputs

# For all notebooks in a folder
find . -name "*.ipynb" -exec nbstripout {} \;
```

---

## 5. Makefile for AI Projects

A Makefile is your command reference. No more scrolling through scripts.

```makefile
# Variables
PYTHON := python
PIP := pip
POETRY := poetry

# Installation
install:
	$(POETRY) install
.PHONY: install

# Running
serve:
	uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
.PHONY: serve

serve-prod:
	uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 4
.PHONY: serve-prod

# Testing
test:
	$(POETRY) run pytest tests/ -v --cov=src --cov-report=html
.PHONY: test

test-quick:
	$(POETRY) run pytest tests/ -v
.PHONY: test-quick

# Code quality
lint:
	$(POETRY) run ruff check src/ tests/
	$(POETRY) run black --check src/ tests/
.PHONY: lint

format:
	$(POETRY) run ruff check --fix src/ tests/
	$(POETRY) run black src/ tests/
.PHONY: format

typecheck:
	$(POETRY) run mypy src/ --strict
.PHONY: typecheck

security:
	$(POETRY) run bandit -r src/ -ll
.PHONY: security

# Docker
docker-build:
	docker build -t ai-app:latest .
.PHONY: docker-build

docker-run:
	docker run --env-file .env -p 8000:8000 ai-app:latest
.PHONY: docker-run

docker-push:
	docker tag ai-app:latest $(REGISTRY)/ai-app:latest
	docker push $(REGISTRY)/ai-app:latest
.PHONY: docker-push

# Training & Evaluation
train:
	$(POETRY) run $(PYTHON) src/train.py --config configs/training.yaml
.PHONY: train

evaluate:
	$(POETRY) run $(PYTHON) src/evaluate.py
.PHONY: evaluate

evaluate-ragas:
	$(POETRY) run $(PYTHON) src/evaluate_ragas.py --output results.json
.PHONY: evaluate-ragas

# Deployment
deploy-staging:
	$(POETRY) run $(PYTHON) scripts/deploy.py --environment staging
.PHONY: deploy-staging

deploy-prod:
	$(POETRY) run $(PYTHON) scripts/deploy.py --environment production
.PHONY: deploy-prod

# Cleanup
clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete
	rm -rf .pytest_cache/
	rm -rf .mypy_cache/
	rm -rf htmlcov/
	rm -rf dist/ build/ *.egg-info
.PHONY: clean

# Help
help:
	@echo "Available targets:"
	@echo "  make install        - Install dependencies"
	@echo "  make serve          - Run development server (with reload)"
	@echo "  make serve-prod     - Run production server"
	@echo "  make test           - Run tests with coverage"
	@echo "  make test-quick     - Run tests without coverage"
	@echo "  make lint           - Check code style"
	@echo "  make format         - Auto-format code"
	@echo "  make typecheck      - Run type checking"
	@echo "  make security       - Run security audit"
	@echo "  make docker-build   - Build Docker image"
	@echo "  make docker-run     - Run Docker container"
	@echo "  make train          - Run training script"
	@echo "  make evaluate       - Evaluate model"
	@echo "  make evaluate-ragas - Evaluate RAG with RAGAS"
	@echo "  make deploy-staging - Deploy to staging"
	@echo "  make deploy-prod    - Deploy to production"
	@echo "  make clean          - Clean build artifacts"
.PHONY: help
```

**Usage:**
```bash
make install      # One-command setup
make serve        # Start dev server
make test         # One command to test
make format       # Fix formatting
make help         # Remind yourself what's available
```

No more "what's the command to run tests?" It's always `make test`.

---

## 6. Pre-commit Hooks: Automated Quality Gates

Pre-commit hooks run before you commit, catching issues early.

**.pre-commit-config.yaml:**
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3
        args: [--line-length=88]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.0.260
    hooks:
      - id: ruff
        args: [--select=E,F,I,W]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.2.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
        args: [--strict]

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: [-r, src/, -ll]
```

**Setup:**
```bash
pip install pre-commit
pre-commit install
```

**What happens:**
```bash
git add .
git commit -m "feat: new feature"

# Pre-commit runs:
# - Removes trailing whitespace
# - Fixes line endings
# - Checks YAML syntax
# - Formats with Black
# - Lints with Ruff
# - Type checks with MyPy
# - Security check with Bandit

# If everything passes: commit succeeds
# If something fails: fix, then try again
```

**Emergency skip (only in rare cases):**
```bash
git commit --no-verify  # Skip hooks, commit anyway
```

What each hook does:
| Hook | Prevents |
|------|----------|
| trailing-whitespace | Messy diffs from whitespace |
| check-yaml | Invalid YAML (breaks configs) |
| detect-private-key | Accidentally committing secrets |
| black | Code style inconsistency |
| ruff | Import sorting, unused imports |
| mypy | Type errors in Python |
| bandit | Security vulnerabilities |

---

## Summary: The Workflow That Works

1. **For hard problems:** Use Claude/ChatGPT with proper prompts (debugging, architecture, explanation)
2. **For learning:** 5-step method (scan, run, reference, search, understand)
3. **For organization:** Clean git history with conventional commits
4. **For reusability:** Extract notebooks to modules with tests
5. **For speed:** Makefile with all commands, pre-commit hooks automated
6. **For quality:** Type checking, linting, security scanning before commit

This workflow lets you move fast without the chaos.

---

## Common Issues and Fixes

!!! warning "Issue: Pre-commit hooks too slow"
    Solution: Run only on changed files, exclude large directories
    ```yaml
    - id: mypy
      exclude: ^(tests/fixtures/|.venv)
    ```

!!! warning "Issue: Black and Ruff fight with each other"
    Solution: Configure both to same line length
    ```yaml
    - id: black
      args: [--line-length=88]
    - id: ruff
      args: [--select=E,F,I,W, --line-length=88]
    ```

!!! warning "Issue: Mypy too strict for rapid prototyping"
    Solution: Use `--warn-only` during development, `--strict` in CI
    ```bash
    # Local: warnings don't block
    mypy src/ --warn-only
    
    # CI: strict checking
    mypy src/ --strict
    ```

---

**Ready to be productive? Set up your Makefile and pre-commit hooks today.**

**Next:** [Study Planner →](study-planner.md)
