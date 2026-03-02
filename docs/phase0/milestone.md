# Phase 0 Milestone: Production-Ready AI Backend

Congratulations. You've learned Python patterns, FastAPI, databases, and Linux operations. Now you'll build something real: a production-grade AI backend that serves as the foundation for your entire 6-phase journey.

This isn't a tutorial project. It's a portfolio piece.

---

## Exit Criteria Checklist

Before moving to Phase 1, you must be able to do ALL of these. Not 90%. Not "close enough." All.

### Python Mastery

- [ ] Write type hints in complete functions (arguments, return types, generics). Run `mypy --strict` with zero errors.
- [ ] Build a Pydantic v2 model with `Field`, `@field_validator`, and `@model_validator`. Validate cross-field constraints.
- [ ] Write an async function that fetches data 100 times in parallel using `asyncio.gather`. Explain why it's faster than sequential.
- [ ] Build a async context manager (`async with`) that properly cleans up resources.
- [ ] Write a decorator that retries a function up to 3 times with exponential backoff.
- [ ] Write pytest tests for async functions using `@pytest.mark.asyncio` with `AsyncMock`.
- [ ] Use dependency injection (not global state). Explain why testability matters.
- [ ] Create a Python package with src layout, __init__.py, and install with `pip install -e .`.

### FastAPI

- [ ] Build an API with at least 5 endpoints (POST, GET, DELETE). Request/response models are Pydantic.
- [ ] Implement API key authentication. Verify it blocks unauthorized requests.
- [ ] Add rate limiting (max 10 requests per minute). Verify requests are blocked after limit.
- [ ] Implement streaming response for a slow operation (generator + StreamingResponse).
- [ ] Use `Depends()` for dependency injection. Verify mocking works in tests.
- [ ] Implement custom exception handler that returns structured error JSON.
- [ ] Add CORS middleware. Verify it allows requests from specific domains.
- [ ] Write tests using TestClient. All tests pass.

### Databases

- [ ] Create PostgreSQL schema with 5+ tables. Include proper types, constraints, indexes.
- [ ] Write SQLAlchemy ORM models matching the schema. Define relationships.
- [ ] Write at least 3 async queries: insert, update, semantic search with embeddings.
- [ ] Create an Alembic migration. Apply and rollback successfully.
- [ ] Store and query embeddings using pgvector. Cosine similarity search returns correct results.
- [ ] Set up Redis caching. Cache an expensive operation with TTL. Verify it works.
- [ ] Write a function that handles database transactions (begin, commit, rollback).

### Linux & Operations

- [ ] Deploy FastAPI app using systemd (not terminal). Verify `systemctl status` shows it's running.
- [ ] Configure nginx reverse proxy. Verify requests go through nginx not directly to port 8000.
- [ ] Set up HTTPS with Let's Encrypt. Certificate auto-renews.
- [ ] View application logs using `journalctl -u myapp -f`. Logs are structured.
- [ ] Create a health check endpoint. Monitor script tests it via curl.
- [ ] SSH into a server using keys (no password). Understand host key verification.
- [ ] Use tmux to run a background job. Detach, reconnect, verify it's still running.
- [ ] Check resource usage (CPU, memory, disk) using top, free, df. Know how to increase limits.

### Testing & CI

- [ ] Build a pytest test suite with 20+ tests. All pass locally.
- [ ] Use fixtures for setup/teardown. Dependency injection in tests.
- [ ] Mock external services (LLM API, database). Tests don't call real APIs.
- [ ] Test error cases (403, 404, 500). Verify error responses are correct.
- [ ] Measure test coverage. At least 70% of code is tested.

### Deployment & Monitoring

- [ ] Create .env and .env.example files. Secrets are separated from code.
- [ ] Write a Dockerfile. Build it. Run it. Verify app works in container.
- [ ] Create docker-compose.yml with postgres, redis, your app. Everything starts together.
- [ ] Set up structured logging. JSON logs with timestamp, level, message.
- [ ] Create monitoring script that checks service health every minute. Auto-restart if down.

### Documentation & Portfolio

- [ ] Commit all code to GitHub. Clean commit history (not 1000 messy commits).
- [ ] Write a README.md that explains: what it does, how to install, how to run, example API calls.
- [ ] Document API endpoints in OpenAPI (FastAPI auto-generates this).
- [ ] Have runnable code examples. Someone can clone, `poetry install`, `docker-compose up`, and it works.

### Interview Readiness

- [ ] Can explain your architecture decisions: "I chose PostgreSQL + Redis because..."
- [ ] Can explain trade-offs: "FastAPI has faster startup than Django, but Django has more database ORM features..."
- [ ] Can discuss what you'd do differently: "I'd use async task queue for background jobs instead of BackgroundTasks..."
- [ ] Can describe what breaks at scale: "At 1M requests/day, I'd add caching and database replication..."

---

## Capstone Project: Document Processing & Chat Backend

### Problem Statement

You're building the backend for an AI document analyst. Users upload documents (PDFs, text), the system processes them (chunks, embeddings), and answers questions about them.

This is a RAG (Retrieval-Augmented Generation) system backend.

### What You'll Build

A FastAPI service with:
- **Document Upload API** — Accept documents, process asynchronously
- **Chat API** — Query documents, get answers with semantically relevant context
- **Database** — PostgreSQL + pgvector for embeddings + Redis for caching
- **Authentication** — API key based (expandable to JWT)
- **Rate Limiting** — Fair access for all users
- **Monitoring** — Health checks, structured logs
- **Deployment** — Docker, systemd, nginx, HTTPS

### Tech Stack

**Required:**
- FastAPI (API framework)
- PostgreSQL (structured data)
- pgvector (semantic search)
- Redis (caching)
- SQLAlchemy 2.0 (async ORM)
- Pydantic v2 (validation)
- pytest (testing)
- Docker + docker-compose (containerization)
- nginx (reverse proxy)
- systemd (service management)

**Optional (for LLM integration):**
- OpenAI (or Anthropic/Groq) for embeddings & chat
- Instructor (structured LLM outputs)

### API Endpoints to Implement

**Authentication:**
```
POST /auth/api-key
  Create API key for user. Return: {"api_key": "..."}

Headers required on all other endpoints:
  X-API-Key: <value>
```

**Document Management:**
```
POST /documents
  Upload a document.
  Request: {"title": str, "content": str}
  Response: {"id": str, "status": "processing"}

GET /documents/{doc_id}
  Retrieve document metadata + chunks

DELETE /documents/{doc_id}
  Delete document + chunks + embeddings

GET /documents
  List user's documents

POST /documents/{doc_id}/chunks
  Force re-chunking (admin only)
```

**Chat / Semantic Search:**
```
POST /chat
  Query documents with conversation context
  Request: {"message": str, "session_id": str}
  Response: {"response": str, "context": [chunks]}

GET /history/{session_id}
  Get conversation history

DELETE /history/{session_id}
  Clear conversation
```

**System:**
```
GET /health
  Liveness check. Return: {"status": "healthy"}

GET /metrics
  Usage metrics (documents, chunks, messages)
```

### Detailed Specifications

**Document Processing Pipeline**
1. User uploads document → saved to PostgreSQL
2. Background task chunks it (512-char overlapping)
3. Chunks stored in PostgreSQL
4. Each chunk embedded by OpenAI API (cached in Redis)
5. Embeddings stored in PostgreSQL + pgvector
6. Ready for semantic search

**Chat Flow**
1. User sends message → query embedded
2. Semantic search finds top-5 similar chunks
3. Chunks sent as context to LLM
4. LLM generates response
5. Response cached to avoid recomputation
6. Conversation saved to PostgreSQL

**Database Schema**
```sql
users (user_id, email, api_key_hash, created_at)
documents (doc_id, user_id, title, content, created_at)
chunks (chunk_id, doc_id, content, start_pos, created_at)
embeddings (embedding_id, chunk_id, vector, created_at)
sessions (session_id, user_id, created_at)
messages (message_id, session_id, role, content, created_at)
```

**Authentication**
- API key in `X-API-Key` header
- Hash API keys (never store plaintext)
- Rate limit per user (10 requests/minute free tier)
- Rate limit per API key

### Success Criteria (Definition of "Done")

- [ ] Code compiles. Zero type errors (mypy --strict).
- [ ] All tests pass (pytest). Coverage > 70%.
- [ ] All 6 API endpoint categories implemented.
- [ ] Document upload → chunking → embedding works end-to-end.
- [ ] Chat returns semantic-relevant context.
- [ ] Runs in Docker with `docker-compose up`.
- [ ] Accessible at https://localhost with self-signed cert (or http in dev).
- [ ] Deploys on Linux server with nginx, systemd, HTTPS.
- [ ] Health check endpoint responds in <100ms.
- [ ] Logs are structured JSON. No sensitive data in logs.
- [ ] README explains everything. Someone unfamiliar can deploy it.

### Project Structure

```
my-ai-backend/
├── src/
│   ├── myapp/
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI app
│   │   ├── config.py               # Settings from .env
│   │   ├── schemas.py              # Pydantic models
│   │   ├── database.py             # SQLAlchemy setup
│   │   ├── redis_client.py         # Redis setup
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   └── security.py         # API key verification
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── embedding_service.py   # Chunk embedding
│   │   │   ├── chat_service.py        # LLM calls
│   │   │   └── document_service.py    # Document processing
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── documents.py
│   │   │   ├── chat.py
│   │   │   └── health.py
│   │   ├── models.py               # SQLAlchemy models
│   │   └── utils/
│   │       ├── __init__.py
│   │       └── logging.py          # Structured logging
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py             # Pytest fixtures
│   │   ├── test_auth.py
│   │   ├── test_documents.py
│   │   ├── test_chat.py
│   │   └── test_integration.py
├── migrations/                      # Alembic
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example
├── .gitignore
├── .pre-commit-config.yaml
├── Makefile
├── deploy/
│   ├── ai-api.service              # systemd service
│   └── nginx.conf                  # nginx config
├── README.md
└── LICENSE

```

### Stretch Goals (If you finish early)

- [ ] MongoDB for raw document storage (not just PostgreSQL)
- [ ] Scheduled batch re-indexing of documents
- [ ] Kubernetes deployment (Phase 4 content, but you could preview)
- [ ] Webhook notifications on document processing complete
- [ ] Audit logging (who accessed what, when)
- [ ] Fine-tuning pipeline (cheap embedding model vs premium)
- [ ] Analytics dashboard (document stats, query patterns)

---

## Self-Assessment Rubric

### Beginner: "I Finished the Cutscenes"

You completed the project following instructions but don't deeply understand the pieces.

**Indicators:**
- Can deploy the project with the exact commands from README, but if something breaks, you're stuck
- Understand what each tool does (FastAPI, PostgreSQL) but can't explain trade-offs
- Tests exist but you don't write new ones confidently
- Database queries mostly copied from examples. Don't understand JOINs
- Don't know why you chose PostgreSQL instead of MongoDB
- Can list systemd commands but don't know what `/etc/systemd/system/` is
- Logs exist but you don't monitor them

**You are at this level if:**
- Copy/paste code to make things work
- Don't modify dependencies without it breaking
- Tests fail when you add a feature

**To level up:** Understand every line of code. Modify something. Understand why it breaks. Fix it.

---

### Competent: "I Built This and Understand It"

You built the project and could explain it to another engineer.

**Indicators:**
- Understand each component and how it fits together
- Can explain: "I chose PostgreSQL because transactions matter. MongoDB for raw documents because schema is flexible."
- Tests comprehensive, would catch real bugs
- SQL queries correct (JOINs work, N+1 is avoided, indexes used)
- Chose reasonable defaults (pool_size, cache TTL, rate limits)
- Systemd service works, logs are clean, monitoring alerts make sense
- Deployed it yourself, not just followed instructions

**You are at this level if:**
- Can modify the codebase without breakage
- Write new tests for new features
- Understand the failure modes ("If this fails, the app will...")

**To level up:** Think about scale. "What breaks at 1M requests/day?"

---

### Hire-Ready: "I'd Hire You"

You built something production-grade. An engineer looking at your code wouldn't have notes.

**Indicators:**
- Code is self-documenting. Types everywhere, docstrings explain *why* not *what*
- Error handling is comprehensive. Edge cases handled. User sees helpful messages.
- Performance is thoughtful. Indexes are there. Caching strategy is clear. N+1 is impossible.
- Security is baked in. Secrets never in code. Rate limiting prevents abuse.
- Deployment is automated. Docs are complete. Someone unfamiliar can deploy in 15 minutes.
- Testing demonstrates mastery. 80%+ coverage. Fixtures prevent duplicate setup. Mocks are used correctly.
- Architecture decisions are justified. "We use Redis instead of application cache because..."

**You are at this level if:**
- Your code could go to production as-is
- You've thought about "what breaks at scale" and handled it
- You'd review another person's code and catch real issues
- You could explain this architecture in a technical interview

---

## What a Failed Milestone Looks Like

If you're exhibiting these signs, you haven't passed Phase 0. Do not move to Phase 1.

### Code Quality Red Flags

- [ ] **Mypy has errors.** Types are incomplete or wrong. `mypy --strict` doesn't pass.
- [ ] **Tests don't run.** Test suite is broken or incomplete. Coverage < 50%.
- [ ] **Global state everywhere.** No dependency injection. Hard to test.
- [ ] **Blocking operations in async code.** Event loop freezes. API hangs.
- [ ] **No error handling.** Unhandled exceptions crawl in logs.
- [ ] **SQL injection possible.** String formatting instead of parameterized queries.
- [ ] **Secrets in code.** API keys in pyproject.toml or hardcoded strings.

### Deployment Red Flags

- [ ] **Only works locally.** Deployment fails.
- [ ] **Deployment is manual.** "Type these 5 commands" instead of automation.
- [ ] **Cannot reproduce environment.** Dockerfile doesn't match pyproject.toml.
- [ ] **No monitoring.** App crashes silently.
- [ ] **No logs.** Nothing appears in journalctl. No way to debug.
- [ ] **Permissions are broken.** systemd service fails with permission denied.

### Database Red Flags

- [ ] **No indexes.** Queries are slow (>1 second for 1k rows).
- [ ] **N+1 queries.** Load 10 users, then query each user's documents = 11 queries.
- [ ] **Migrations don't work.** `alembic upgrade head` fails.
- [ ] **Embeddings not stored efficiently.** Vectors as JSON strings instead of pgvector.
- [ ] **No pagination.** Loading entire datasets into memory.

### Architecture Red Flags

- [ ] **No caching.** Every request hits database.
- [ ] **SynchronousAPI calls.** Each request blocks for 2 seconds waiting for database.
- [ ] **No rate limiting.** Single client can send 1M requests.
- [ ] **Hardcoded configuration.** Change environment = manual code edits.
- [ ] **No auth.** Anyone can call any endpoint.

---

## Common Reasons People Rush Phase 0 (And Why It Backfires)

### Reason 1: "I already know Python"

**The trap:** Python syntax ≠ production Python. You know loops and functions, but you've never written type hints, async functions, or proper OOP.

**Where it breaks in Phase 1:** You build an LLM client that's not async. When you call OpenAI API 10 times, it's sequential (20 seconds). Should be parallel (2 seconds). You blame the library. Actually, you didn't understand async.

**Cost:** 1 week rebuilding the LLM client with proper async.

---

### Reason 2: "I'll learn the database stuff when I need it"

**The trap:** Thinking "I'll use PostgreSQL basics for now, advanced stuff later."

**Where it breaks in Phase 1:** You're storing embeddings. You find yourself with a slow semantic search (full table scan). You add an index, but it doesn't help because you're using text columns instead of pgvector. You rewrite the schema. All embedded vectors are orphaned.

**Cost:** 3 days rewriting. Data loss. Embarrassment in code review.

---

### Reason 3: "Testing slows me down"

**The trap:** "Tests are optional. I'll add them later."

**Where it breaks in Phase 1:** You add a feature. It works locally. Deployed to staging, it breaks. You don't know which change broke it. Spent 6 hours debugging because you have no tests to isolate the issue.

**Cost:** 6 hours debugging. Loss of confidence. Tests would have caught it in 30 seconds.

---

### Reason 4: "I'll deploy later"

**The trap:** "I'll ship locally first, then figure out deployment."

**Where it breaks in Phase 1:** Code works locally. Deployment fails. Secrets are hardcoded. Database connection string is wrong. Service won't start.

**Now you're learning deployment AND Phase 1 content simultaneously. Brain overload.

**Cost:** 2 weeks of frustration.

---

### Reason 5: "Async is complicated, I'll skip it"

**The trap:** Not learning asyncio, using synchronous libraries instead.

**Where it breaks in Phase 1:** Every LLM API is async (OpenAI, Anthropic, Groq). Your code is sync. You can't call them properly. You spend 2 weeks learning async under deadline pressure.

**Cost:** 2 weeks of painful learning at the worst time.

---

## What Happens in Phase 1

Phase 1 is where you learn LLM integration, embeddings, RAG pipelines, fine-tuning, and evaluation.

If you skipped Phase 0 foundations:

- **Weak Python?** → Async code becomes incomprehensible
- **Weak FastAPI?** → Streaming LLM responses hangs the server
- **Weak databases?** → Embeddings don't store/retrieve correctly
- **Weak operations?** → Your service crashes in production and you don't know why

**Result:** You're learning Phase 0 content while building Phase 1 content. You fall 4 weeks behind.

---

## Final Requirements

Before claiming you've completed Phase 0:

### GitHub Repository

- [ ] Code is on GitHub (public or private, link in resume)
- [ ] README explains what the project does
- [ ] README has setup instructions
- [ ] README has example API calls
- [ ] Code is clean (use `make format` before pushing)
- [ ] Commit history is readable (not 100 commits named "fix")

### Documentation

- [ ] API endpoints documented (FastAPI's /docs auto-generates this)
- [ ] Database schema documented (comment in schema.sql or models.py)
- [ ] Architecture documented (README with diagram or text explanation)
- [ ] Deployment documented (how to run on Linux server)

### Deployability

- [ ] Clone repo on fresh machine → `poetry install` → `poetry run pytest` → all tests pass
- [ ] `docker-compose up` → all services start → API responds
- [ ] Deployment docs are clear enough for someone else to follow

### Honesty

- [ ] You wrote most of the code (don't claim copy-paste examples)
- [ ] You can explain every line
- [ ] You didn't use AI to avoid thinking (use AI to learn faster, not to skip learning)

---

## Exit Interview: Ask Yourself These Questions

Before moving to Phase 1, sit down and answer these honestly. If you can't:

### Python

- "Why did I use Pydantic instead of dataclasses?"
- "How would I add a validator that ensures embedding length?"
- "Why is `asyncio.gather` faster than a for loop with await?"

### FastAPI

- "What would I do if I got 1000 requests/second?"
- "How does dependency injection make testing easier?"
- "Why shouldn't I use global variables?"

### Databases

- "Why does this query need an index?"
- "What's the difference between pgvector and storing embeddings as JSON?"
- "How would I migrate a million rows to a new schema?"

### Operations

- "How do I know if a production service is working right now?"
- "What happens if the server runs out of disk space?"
- "How do I update the code without downtime?"

If you can answer all of these with specifics (not vague), you're ready for Phase 1.

---

## You're Ready. Phase 1 Awaits.

You've built something production-grade. You understand the foundations. Your AI backend is real.

Celebrate for 1 day.

Then: Open Phase 1.

→ **[Phase 1: AI Systems (LLMs, RAG, Fine-tuning) →](../phase1/overview.md)**

---

## Before You Go: Phase 1 Preview

Phase 1 builds directly on this foundation. You'll:

1. **Call LLM APIs** (OpenAI, Anthropic, Groq) from your FastAPI service
2. **Build embeddings pipeline** (chunk documents, embed, store in pgvector)
3. **Implement RAG** (retrieve relevant chunks, send to LLM for generation)
4. **Build evaluation framework** (grade your system's outputs)
5. **Fine-tune models** (LoRA, train on custom data)
6. **Handle AI-specific failures** (timeouts, API errors, hallucinations)

All with the foundation you just built.

See you in Phase 1. You've got this.
