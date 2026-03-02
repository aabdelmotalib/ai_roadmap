# FastAPI Deep Dive: Building AI-Ready APIs

FastAPI is the production standard for AI backends. Not just because it's fast. Because it does the foundational work (validation, documentation, dependency injection) automatically, leaving you to focus on AI logic.

---

## 1. Why This Matters in Production

A startup built an LLM chatbot API without proper request validation. Users could send payloads of any size. One attacker wrote a script sending 1GB requests. The server crashed trying to parse them.

Root cause: No request size limit. No validation. No rate limiting. They could have fixed it in FastAPI with 3 lines of code.

**Production reality:**
- Your API is a public attack surface. Validate everything.
- API responses fail in production (network timeouts, errors). Handle them gracefully.
- Tens of concurrent users. Your code must handle parallelism.
- Monitoring matters. Logs must be structured and queryable.

---

## 2. Conceptual Explanation: Routing & Dependency Injection

FastAPI is fundamentally different from Flask. It shifts from "magical global state" to "explicit dependency injection."

```python
# Flask (implicit, harder to test)
from flask import Flask, request

app = Flask(__name__)
db = Database()  # Global!

@app.route("/docs")
def get_docs():
    return db.fetch()  # Where does db come from? Global.
```

```python
# FastAPI (explicit, testable)
from fastapi import FastAPI, Depends

app = FastAPI()

def get_db():
    return Database()

@app.get("/docs")
async def get_docs(db: Database = Depends(get_db)):
    return db.fetch()  # db injected, easy to test
```

**Dependency Injection** means: "Here's a function that creates what I need. Call it, pass the result to me."

For AI systems: LLM clients, embedding models, database sessions are all "dependencies." Inject them properly, and your code becomes:
- **Testable** — Mock dependencies in tests
- **Reusable** — Same dependency for multiple endpoints
- **Clear** — Everyone knows where objects come from

---

## 3. Code Examples: FastAPI Patterns

### 3.1 Basic Routing with Request/Response Models

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List

app = FastAPI(title="AI Document API", version="1.0.0")

class Document(BaseModel):
    id: str
    title: str = Field(..., min_length=1)
    content: str
    tags: List[str] = Field(default_factory=list)

class DocumentID(BaseModel):
    id: str
    message: str

@app.post("/documents", response_model=DocumentID, status_code=201)
async def create_document(doc: Document) -> DocumentID:
    """
    Create a new document.
    FastAPI validates input against Document schema.
    Returns DocumentID with status 201 (created).
    """
    return DocumentID(id="doc123", message="Created")

@app.get("/documents/{doc_id}", response_model=Document)
async def get_document(doc_id: str) -> Document:
    """Get a document by ID."""
    if doc_id not in ["doc123"]:
        raise HTTPException(status_code=404, detail="Not found")
    
    return Document(
        id=doc_id,
        title="Example",
        content="...",
        tags=["ai"]
    )
```

### 3.2 Dependency Injection: Shared Database Session

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from contextlib import asynccontextmanager

class DatabaseManager:
    def __init__(self):
        self.session = None
    
    async def __aenter__(self):
        self.session = AsyncSession()
        return self.session
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.session.close()

# Dependency function: creates and yields a DB session
async def get_db() -> AsyncSession:
    """
    Provides a database session to endpoints.
    Automatically closes after the endpoint returns.
    """
    async with DatabaseManager() as session:
        yield session

@app.post("/documents")
async def create_document(
    doc: Document,
    db: AsyncSession = Depends(get_db)
) -> DocumentID:
    """
    The db parameter is automatically injected.
    FastAPI calls get_db(), passes the session here.
    """
    # Use db to save document
    return DocumentID(id="doc123", message="Saved")
```

### 3.3 Streaming Responses: LLM Token Streaming

```python
from fastapi.responses import StreamingResponse
import asyncio

@app.post("/chat/stream")
async def chat_stream(message: str) -> StreamingResponse:
    """
    Stream LLM response tokens as they arrive.
    Client sees text appearing in real-time.
    """
    async def generate():
        prompt = f"User: {message}\nAssistant: "
        
        # Simulate streaming (from LLM API in reality)
        response = "This is a streaming response from the LLM."
        
        for token in response.split():
            # Yield each token immediately
            yield f"data: {token}\n\n"
            await asyncio.sleep(0.1)  # Simulate latency
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )
```

### 3.4 Background Tasks

```python
from fastapi import FastAPI, BackgroundTasks

@app.post("/documents/upload")
async def upload_document(
    file: UploadFile,
    background_tasks: BackgroundTasks
) -> dict:
    """
    Accept document, save it, queue embedding computation.
    Returns immediately. Embedding happens in background.
    """
    # Save file
    filename = file.filename
    
    # Queue background task
    async def generate_embeddings(fname: str):
        # This runs AFTER response is sent
        embeddings = await create_embeddings(fname)
        await save_embeddings(fname, embeddings)
    
    background_tasks.add_task(generate_embeddings, filename)
    
    return {
        "status": "Received",
        "filename": filename,
        "message": "Embeddings will be computed shortly"
    }
```

### 3.5 Authentication: API Key

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

async def verify_api_key(x_api_key: Optional[str] = Header(None)):
    """
    Verify X-API-Key header.
    If missing or wrong, raise 403 Forbidden.
    """
    if x_api_key is None:
        raise HTTPException(status_code=403, detail="Missing API key")
    
    # In production: check against database/cache
    if x_api_key != "valid-key-12345":
        raise HTTPException(status_code=403, detail="Invalid API key")
    
    return x_api_key

@app.post("/documents", dependencies=[Depends(verify_api_key)])
async def create_document(doc: Document):
    """Only accessible with valid X-API-Key header."""
    return {"status": "created"}
```

### 3.6 Rate Limiting with slowapi

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import FastAPI

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()

@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request, exc):
    return JSONResponse(
        status_code=429,
        content={"error": "Rate limit exceeded"}
    )

@app.get("/chat")
@limiter.limit("10/minute")  # Max 10 requests per minute per IP
async def chat(request: Request, message: str):
    """Rate-limited endpoint."""
    return {"response": "Hello"}
```

---

## 4. Architecture Diagram: FastAPI Request Flow

```mermaid
graph LR
    A["Client<br/>HTTP Request"] --> B["Route Matching<br/>@app.get, @app.post"]
    B --> C["Request Validation<br/>Pydantic models"]
    C --> D["Dependency Injection<br/>Get DB, LLM client, etc"]
    D --> E["Endpoint Logic<br/>Your code"]
    E --> F["Response Serialization<br/>JSON from Pydantic"]
    F --> G["HTTP Response<br/>JSON, Streaming, Files"]
    
    H["Error Handling<br/>Custom exceptions"] -.-> E
    I["Middleware<br/>CORS, Logging"] -.-> B
    J["Rate Limiting<br/>Per-IP limits"] -.-> B
    
    style E fill:#FFE082
    style C fill:#C5CAE9
    style D fill:#A5D6A7
```

---

## 5. Tools Section: FastAPI Ecosystem

| Tool | Purpose | Example |
|------|---------|---------|
| **FastAPI** | Web framework | `@app.post("/docs")` |
| **Pydantic** | Request validation | `doc: Document` parameter |
| **SQLAlchemy** | ORM for databases | `db: Session = Depends(get_db)` |
| **slowapi** | Rate limiting | `@limiter.limit("10/minute")` |
| **python-jose** | JWT tokens | JWT auth |
| **httpx** | Async HTTP client | Call LLM APIs |
| **pytest** | Testing | `TestClient(app)` |
| **uvicorn** | ASGI server | `uvicorn main:app --reload` |

---

## 6. Decision Framework: When to Use FastAPI Patterns

| Scenario | Pattern | Why |
|----------|---------|-----|
| Multiple endpoints sharing DB | `Depends(get_db)` | Clean, testable |
| LLM tokens appear in real-time | `StreamingResponse` | Better UX |
| Indexing documents after upload | `BackgroundTasks` | Don't block user |
| Require authentication | JWT in `Depends()` | Secure, stateless |
| Prevent API abuse | `slowapi` limiter | Protect infrastructure |
| Document API automatically | Pydantic models | OpenAPI schema auto-generated |

---

## 7. Step-by-Step: Building Your First AI API

### Step 1: Project Structure

```
src/
├── myapp/
│   ├── __init__.py
│   ├── main.py              # FastAPI app
│   ├── schemas.py           # Pydantic models
│   ├── dependencies.py      # Dependency functions
│   ├── database.py          # DB setup
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── documents.py     # Document endpoints
│   │   └── chat.py          # Chat endpoints
│   └── services/
│       ├── __init__.py
│       ├── embedding.py     # Embedding service
│       └── llm.py           # LLM service
```

### Step 2: Create Schemas (src/myapp/schemas.py)

```python
from pydantic import BaseModel, Field
from typing import List, Optional

class DocumentCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=256)
    content: str = Field(..., min_length=1)
    tags: List[str] = Field(default_factory=list)

class DocumentResponse(DocumentCreate):
    id: str
    created_at: str
```

### Step 3: Set Up Database (src/myapp/database.py)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"
engine = create_async_engine(DATABASE_URL, echo=False)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with async_session() as session:
        yield session
```

### Step 4: Create Dependencies (src/myapp/dependencies.py)

```python
from fastapi import Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from .database import get_db

async def verify_token(token: str):
    """Verify JWT token."""
    if token != "my-secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return token
```

### Step 5: Create Routes (src/myapp/routes/documents.py)

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from ..schemas import DocumentCreate, DocumentResponse
from ..database import get_db

router = APIRouter(prefix="/documents", tags=["documents"])

@router.post("", response_model=DocumentResponse, status_code=201)
async def create_document(
    doc: DocumentCreate,
    db: AsyncSession = Depends(get_db)
) -> DocumentResponse:
    """Create and save a document."""
    # Save to database
    return DocumentResponse(
        id="doc123",
        title=doc.title,
        content=doc.content,
        tags=doc.tags,
        created_at="2024-03-01T10:00:00Z"
    )
```

### Step 6: Assemble in Main App (src/myapp/main.py)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .routes import documents

app = FastAPI(
    title="Document AI API",
    description="Process documents with embeddings",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routes
app.include_router(documents.router)

@app.get("/health")
async def health_check():
    """Liveness check for Kubernetes."""
    return {"status": "healthy"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Step 7: Run It

```bash
poetry add fastapi uvicorn[standard] sqlalchemy pydantic

uvicorn src.myapp.main:app --reload
# Server running at http://localhost:8000
# Docs at http://localhost:8000/docs
```

---

## 8. Practical Project: Production Chat API

**Goal:** Build a chatbot API with streaming, authentication, database persistence, and rate limiting.

**Endpoints:**
- `POST /chat/stream` — Stream chat responses
- `GET /history/{session_id}` — Get conversation history
- `DELETE /history/{session_id}` — Clear history
- `GET /health` — Liveness check

**Full code (src/myapp/main.py):**

```python
from fastapi import FastAPI, Depends, HTTPException, BackgroundTasks
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from slowapi import Limiter
from slowapi.util import get_remote_address
from typing import Optional
import asyncio
import json

app = FastAPI(title="Chat API")
limiter = Limiter(key_func=get_remote_address)

# In-memory storage (replace with DB in production)
sessions: dict = {}

class ChatRequest(BaseModel):
    message: str
    session_id: str

@app.post("/chat/stream")
@limiter.limit("10/minute")
async def chat_stream(request_body: ChatRequest, request=None):
    """Stream chat response."""
    session_id = request_body.session_id
    message = request_body.message
    
    # Initialize session if needed
    if session_id not in sessions:
        sessions[session_id] = []
    
    # Add message to history
    sessions[session_id].append({"role": "user", "content": message})
    
    async def generate():
        # Simulate LLM streaming (in production: call OpenAI API)
        response = f"Echo: {message}"
        for word in response.split():
            yield f"data: {json.dumps({'token': word})}\n\n"
            await asyncio.sleep(0.05)
    
    return StreamingResponse(generate(), media_type="text/event-stream")

@app.get("/history/{session_id}")
async def get_history(session_id: str):
    """Get conversation history."""
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail="Session not found")
    return {"messages": sessions[session_id]}

@app.delete("/history/{session_id}")
async def clear_history(session_id: str):
    """Clear conversation history."""
    if session_id in sessions:
        del sessions[session_id]
    return {"status": "cleared"}

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

---

## 9. Debugging Playbook: 10 FastAPI Errors

### Error 1: 422 Unprocessable Entity (Validation Error)

**Symptom:** POST request returns 422 with validation error.

**Explanation:** Request body doesn't match Pydantic schema.

**Fix:**
```python
# Wrong request (missing required field)
POST /documents {"title": "Test"}  # content is required

# Right request
POST /documents {"title": "Test", "content": "..."}
```

**Prevention:** Check request schema in OpenAPI docs at `/docs`.

---

### Error 2: 500 Internal Server Error (Database Connection Failed)

**Symptom:** Every request crashes with 500.

**Explanation:** Database isn't running or credentials are wrong.

**Fix:**
```bash
# Check database is running
docker ps | grep postgres

# Test connection manually
psql postgresql://user:pass@localhost:5432/dbname
```

**Prevention:** Set up health check endpoint that tests DB connection.

---

### Error 3: RuntimeError: Event loop is closed

**Symptom:** Happens when reloading dev server.

**Explanation:** Previous event loop wasn't cleaned up before starting new one.

**Fix:**
```bash
# Use uvicorn with proper reload
uvicorn src.myapp.main:app --reload

# Add this to pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

**Prevention:** Use `--reload` flag. Don't manually restart event loop.

---

### Error 4: 429 Too Many Requests (Rate Limit)

**Symptom:** After making several requests rapidly, get 429.

**Explanation:** Exceeded rate limit (slowapi limiter).

**Fix:**
```python
# Increase limit
@limiter.limit("100/minute")  # More lenient
```

**Prevention:** Right-size limits for your use case. Test with load testing.

---

### Error 5: CORS Errors: No 'Access-Control-Allow-Origin'

**Symptom:** Frontend can't reach API: `Access to XMLHttpRequest has been blocked by CORS policy`.

**Explanation:** CORS middleware not configured.

**Fix:**
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Specific origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Prevention:** Configure CORS for specific domains, not `["*"]` in production.

---

### Error 6: 401 Unauthorized (Auth Failed)

**Symptom:** Valid requests get 401.

**Explanation:** Token is missing, malformed, or expired.

**Fix:**
```python
# Check header is present
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8000/protected

# Verify token format and expiry in dependency
async def get_current_user(token: str = Depends(oauth2_scheme)):
    if not token or not is_valid_token(token):
        raise HTTPException(status_code=401, detail="Invalid token")
    return decode_token(token)
```

**Prevention:** Log token validation failures. Use structured error messages.

---

### Error 7: Request body is too large (413 Payload Too Large)

**Symptom:** Large file upload rejected.

**Explanation:** FastAPI has a default max upload size.

**Fix:**
```python
from fastapi import FastAPI

app = FastAPI()

# Set max upload size to 100MB
app.middleware("http")(limit_upload_size(100 * 1024 * 1024))
```

**Prevention:** Validate file sizes before upload. Stream large files.

---

### Error 8: Dependency injection returns None

**Symptom:** Dependency function works, but injected parameter is None.

**Explanation:** Dependency function didn't return anything.

**Fix:**
```python
# Wrong (doesn't return)
async def get_db():
    session = AsyncSession()
    yield session
    # Forgot to close!

# Right
async def get_db():
    session = AsyncSession()
    try:
        yield session
    finally:
        await session.close()
```

**Prevention:** Always yield in dependency. Use try/finally for cleanup.

---

### Error 9: StreamingResponse doesn't send final data

**Symptom:** Stream starts but doesn't finish.

**Explanation:** Generator didn't yield last item or crashed silently.

**Fix:**
```python
async def generate():
    try:
        for item in items:
            yield f"data: {item}\n\n"
    except Exception as e:
        # Always log errors in streaming responses
        print(f"Stream error: {e}")
        yield f"data: {json.dumps({'error': str(e)})}\n\n"
```

**Prevention:** Add logging. Test stream with partial data.

---

### Error 10: Circular import between routes and main

**Symptom:** `ModuleNotFoundError: No module named 'myapp'`

**Explanation:** routes.py imports from main.py which imports routes.py.

**Fix:**
```python
# In main.py: don't import routes directly
# Instead, import and register at the end
from src.myapp.routes import documents

app = FastAPI()
app.include_router(documents.router)
```

**Prevention:** Separate route files from main. Use `include_router`. Keep imports at top of file.

---

## 10. Common Mistakes: FastAPI Pitfalls

### Mistake 1: Blocking Operations in Async Endpoints

```python
# Wrong: synchronous file I/O blocks entire server
@app.post("/documents")
async def upload(file: UploadFile):
    with open(f"data/{file.filename}", "wb") as f:
        f.write(await file.read())  # f.write is blocking!

# Right: use async file I/O
import aiofiles

@app.post("/documents")
async def upload(file: UploadFile):
    async with aiofiles.open(f"data/{file.filename}", "wb") as f:
        await f.write(await file.read())
```

---

### Mistake 2: Creating New Dependencies Every Request

```python
# Wrong: new LLM client every request
@app.post("/chat")
async def chat(message: str):
    client = OpenAI()  # Creates new client, wasteful
    return await client.chat(message)

# Right: reuse client via dependency
from functools import lru_cache

@lru_cache(maxsize=1)
def get_llm_client():
    return OpenAI()

@app.post("/chat")
async def chat(message: str, client = Depends(get_llm_client)):
    return await client.chat(message)
```

---

### Mistake 3: Not Handling Missing Dependencies

```python
# Wrong: accessing injected param without checking
@app.get("/data")
async def get_data(db: AsyncSession = Depends(get_db)):
    # If get_db returns None, this crashes
    result = await db.execute(...)

# Right: ensure dependency is never None
async def get_db():
    session = AsyncSession()
    if not session:
        raise HTTPException(status_code=500, detail="DB unavailable")
    yield session
```

---

### Mistake 4: Returning Raw Exceptions

```python
# Wrong: leaks internal info
@app.get("/data")
async def get_data():
    try:
        return expensive_operation()
    except Exception as e:
        return {"error": str(e)}  # User sees internals!

# Right: handle with HTTPException
from fastapi import HTTPException

@app.get("/data")
async def get_data():
    try:
        return expensive_operation()
    except Exception as e:
        # Log for debugging, return generic message
        logger.error(f"Error in get_data: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

### Mistake 5: Not Setting Response Status Codes

```python
# Wrong: POST succeeds but returns default 200
@app.post("/documents")
async def create(doc: Document):
    save(doc)
    return doc  # Status 200, but should be 201

# Right: explicit status code
@app.post("/documents", status_code=201)
async def create(doc: Document):
    save(doc)
    return doc  # Status 201 (created)
```

---

## 11. Production Realism: Tutorial vs Production

### Tutorial Code

```python
# Simple, works locally, fails at scale
app = FastAPI()

documents = {}  # In-memory storage

@app.post("/documents")
async def create(doc: Document):
    doc_id = str(len(documents))
    documents[doc_id] = doc
    return doc
```

### Production Code

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from slowapi import Limiter
from slowapi.util import get_remote_address
import logging

logger = logging.getLogger(__name__)
limiter = Limiter(key_func=get_remote_address)

app = FastAPI(
    title="Document API",
    description="Production-ready document service",
    version="1.0.0"
)

# Rate limiting on all endpoints
app.state.limiter = limiter

@app.post("/documents", status_code=201, dependencies=[Depends(limiter.limit("100/minute"))])
async def create(
    doc: Document,
    db: AsyncSession = Depends(get_db),
    token: str = Depends(verify_api_key)
):
    """
    Create a document with:
    - Rate limiting (100/minute)
    - Authentication (API key)
    - Database persistence
    - Logging
    - Error handling
    """
    try:
        # Use database
        db_doc = await db.execute(
            insert(DocumentModel).values(
                title=doc.title,
                content=doc.content
            )
        )
        await db.commit()
        
        logger.info(f"Document created: {db_doc.id}")
        return {"id": db_doc.id, "status": "created"}
    
    except Exception as e:
        await db.rollback()
        logger.error(f"Failed to create document: {e}")
        raise HTTPException(status_code=500, detail="Failed to save document")
```

**Differences:**
- Rate limiting
- Authentication verification
- Database transactions (commit/rollback)
- Structured logging
- Error handling with meaningful messages
- Status codes

---

## 12. Cost & Performance Considerations

### Concurrency & Resource Usage

```python
# Naive: sequential, blocks on each DB query
@app.get("/documents/{doc_ids}")
async def get_many(doc_ids: List[str], db: AsyncSession = Depends(get_db)):
    results = []
    for doc_id in doc_ids:
        result = await db.execute(...)  # 10 queries × 100ms = 1000ms
        results.append(result)
    return results

# Optimized: batch query, single DB round-trip
import asyncio

@app.get("/documents/{doc_ids}")
async def get_many(doc_ids: List[str], db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Document).where(Document.id.in_(doc_ids))  # Single query
    )
    return result.scalars().all()  # Returns in 100ms
```

**Performance gain:** 10x faster (1000ms → 100ms).

### Connection Pooling

```python
from sqlalchemy.ext.asyncio import create_async_engine

# Without pooling: creates new connection per request
engine = create_async_engine(DATABASE_URL)

# With pooling: reuses connections
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,  # Keep 20 connections ready
    max_overflow=10,  # Allow 10 extra during spikes
    pool_pre_ping=True,  # Verify connections before using
)
```

**Cost:** Reusing connections reduces connection overhead.

---

## 13. Security Considerations

### Input Validation

```python
from pydantic import BaseModel, Field, HttpUrl

class DocumentRequest(BaseModel):
    url: HttpUrl  # Validates URL format
    title: str = Field(..., min_length=1, max_length=256)
    content: str = Field(..., max_length=100000)  # Prevent DOS

# Pydantic rejects malformed data before your code runs
```

### Secret Management

```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class Settings(BaseSettings):
    database_password: SecretStr  # Masked in logs
    api_key: SecretStr
    
    class Config:
        env_file = ".env"

settings = Settings()
# settings.database_password prints as "***"
```

### SQL Injection Prevention

```python
# Wrong: string formatting (vulnerable)
query = f"SELECT * FROM documents WHERE id = {doc_id}"

# Right: parameterized (safe)
query = select(Document).where(Document.id == doc_id)
```

SQLAlchemy uses parameterized queries by default. SQL injection prevented.

---

## 14. Case Study: Stripe's API Design

**Why Stripe matters:** Processes billions of dollars. API is a security-critical surface.

**Their patterns:**
- Strict request validation (Pydantic-like)
- Rate limiting per API key
- Webhook signature verification
- Detailed error messages
- Extensive logging

**Lesson for you:** Treat your AI API like Stripe treats payment APIs. Validation first, then logic.

---

## 15. Interview Cheat Sheet: FastAPI

### Concept 1: Dependency Injection

**Q:** Why use `Depends()` instead of global variables?

**A:** `Depends()` makes code testable and explicit.

```python
# Testable
@app.get("/data")
async def get_data(db: Session = Depends(get_db)):
    return db.query(...)

# In tests
def test_get_data():
    mock_db = MockSession()
    result = get_data(db=mock_db)
```

---

### Concept 2: Streaming vs Regular Responses

**Q:** When should you stream vs return normally?

**A:** Stream when data is slow to generate (LLM) or very large. Regular response for small, fast data.

```python
# Stream: LLM response (slow, appears token-by-token)
@app.post("/chat")
async def chat() -> StreamingResponse:
    return StreamingResponse(generate_tokens())

# Regular: metadata lookup (fast, all at once)
@app.get("/doc/{id}")
async def get_doc(id: str) -> DocumentResponse:
    return db.get(id)
```

---

### Concept 3: Middleware vs Dependencies

**Q:** Should you use middleware or dependencies?

**A:** Middleware runs on every request globally. Dependencies are per-endpoint.

```python
# Middleware: CORS, logging (affects everything)
app.add_middleware(CORSMiddleware, ...)

# Dependencies: auth, DB session (per-endpoint)
@app.get("/protected", dependencies=[Depends(verify_token)])
```

---

### Concept 4: Background Tasks vs Cron Jobs

**Q:** When use BackgroundTasks vs scheduled cron?

**A:** BackgroundTasks for after-request work. Cron for periodic work.

```python
# BackgroundTasks: runs after response is sent
@app.post("/docs")
async def upload(file, background_tasks: BackgroundTasks):
    background_tasks.add_task(generate_embeddings, file)
    return {"status": "uploading"}

# Cron: runs on schedule (separate service)
# Use APScheduler or Celery for scheduled tasks
```

---

### Concept 5: Status Codes

**Q:** What status codes should you use?

**A:**
- 200: Success (GET, PUT)
- 201: Created (POST)
- 204: Deleted (DELETE)
- 400: Bad request (malformed input)
- 401: Unauthorized (bad token)
- 403: Forbidden (valid token, no permission)
- 404: Not found
- 429: Too many requests (rate limit)
- 500: Server error

---

## 16. Review Questions

1. What's the difference between `Depends()` and a global variable? Why use Depends?
2. Write a FastAPI endpoint that requires X-API-Key header authentication.
3. Write streaming response code that sends LLM tokens as they arrive.
4. How do you test a FastAPI endpoint that depends on a database session?
5. What status code should `POST /documents` return on success? Why?
6. Write a background task that embeds documents after upload.
7. How do you enable CORS for a specific domain (not `["*"]`)?
8. Why is `time.sleep()` wrong in a FastAPI endpoint? What's the fix?
9. Write a rate limiter that allows max 100 requests per minute.
10. Explain dependency injection with a concrete example (DB session).

---

## 17. Flashcards

### Card 1
**Front:** When should you use `StreamingResponse`?

**Back:** When data is slow to generate (LLM tokens) or very large (files). Allows client to see partial results immediately instead of waiting for all data.

---

### Card 2
**Front:** What's the difference between status codes 401 vs 403?

**Back:** 401 Unauthorized means "you're not who you claim" (bad token). 403 Forbidden means "you're authenticated but don't have permission."

---

### Card 3
**Front:** Why use `@lru_cache` on dependencies?

**Back:** To reuse expensive resources (LLM client, DB connection) across requests instead of recreating them each time.

---

### Card 4
**Front:** What does `response_model` do in FastAPI?

**Back:** Validates that your endpoint returns data matching the Pydantic model. If your code returns wrong type, FastAPI catches it.

---

### Card 5
**Front:** How do you handle errors in StreamingResponse?

**Back:** Catch exceptions and yield error message. Always log errors. Stream contract allows you to send error after stream starts.

---

### Card 6
**Front:** What's the purpose of `async with AsyncSession`?

**Back:** Ensures session is properly closed even if exception occurs. Prevents connection leaks.

---

### Card 7
**Front:** Why is rate limiting important for AI APIs?

**Back:** LLM APIs are expensive. Without rate limiting, users can spam requests and rack up huge bills.

---

### Card 8
**Front:** What does `depends=[Depends(...)]` do in endpoint decorator?

**Back:** Runs dependency before endpoint, but doesn't pass result. Use when you want dependency to run (for side effects) but don't need the value.

---

### Card 9
**Front:** How do middlewares execute vs dependencies?

**Back:** Middleware runs once per request. Dependencies run based on where they're used (per-endpoint if in endpoint, globally if in app).

---

### Card 10
**Front:** When should you use `background_tasks` vs Celery?

**Back:** BackgroundTasks for quick work (few seconds). Celery for long-running work (minutes+) that should survive app restart.

---

## 18. Teach-It-Back Prompt

Explain to a junior engineer why dependency injection matters for testing.

**Model Answer:**

"With dependencies, you can mock things in tests. Example: if your endpoint calls `Depends(get_db)`, in tests you pass `mock_db` instead of the real database. This means tests run instantly (no DB) and can't accidentally delete production data. Without dependencies, your endpoint would create the database internally, making it hard to test."

---

## 19. 1-Week Review Checklist

- [ ] Deploy a FastAPI app on localhost with at least 3 endpoints
- [ ] Add authentication (API key or JWT) to a protected endpoint
- [ ] Write a test that mocks the database dependency
- [ ] Implement streaming response for something slow
- [ ] Add rate limiting and test that it blocks after limit

---

## 20. Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic v2 Documentation](https://docs.pydantic.dev/)
- [SQLAlchemy Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [slowapi Rate Limiting](https://github.com/laurenceisla/slowapi)
- [OWASP: API Security](https://owasp.org/www-project-api-security/)
- [Real Python: FastAPI](https://realpython.com/fastapi-python-web-apis/)

---

## 21. Next: Databases

You now have an API. It needs data.

→ **[Databases: PostgreSQL, MongoDB, Redis →](databases.md)**
