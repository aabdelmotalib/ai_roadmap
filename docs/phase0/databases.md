# Databases for AI: PostgreSQL, MongoDB, Redis

Database choice shapes your entire architecture. Pick the wrong database, and you're rewriting months of code later.

---

## 1. Why This Matters in Production

A startup built a document search system. They stored everything in PostgreSQL text columns. After 6 months with 100M documents, searches took 45 seconds. Unacceptable.

Root cause: Their schema wasn't optimized for AI. They needed:
- Vector search (pgvector) for semantic similarity
- Proper indexing for full-text search
- Denormalized data for fast retrieval

They rewrite the entire schema. 2 weeks of downtime. $200k in lost business.

**Production reality:**
- Schema design is permanent. Wrong schema = rewrite costs.
- Indexes make or break performance. Some queries are 100x slower without them.
- AI data is unstructured. PostgreSQL + MongoDB hybrid is common.
- Caching (Redis) is not optional. It's load survival.

---

## 2. Conceptual Explanation: The AI Data Stack

AI systems deal with 3 types of data:

**Structured Data** (PostgreSQL)
- User accounts, sessions, metadata
- Queries with exact matches (user_id = 123)
- ACID transactions required

**Unstructured Data** (MongoDB)
- Raw documents, conversations, logs
- Queries with pattern matching (text contains "market")
- Schema flexibility needed

**Cached Data** (Redis)
- Session state, conversation history
- LLM responses (expensive, cache for 1h)
- Temporary computations

**Vector Data** (PostgreSQL + pgvector)
- Embeddings (1536-dim vectors)
- Semantic search (cosine similarity)
- Hybrid: text search + vector search

**The architecture:**
```
User Request
    ↓
FastAPI (validation)
    ↓ (structured lookup)
PostgreSQL (user metadata)
    ↓ (cache check)
Redis (LLM response cache)
    ↓ (if not cached, compute)
MongoDB (fetch documents)
    ↓ (create embeddings)
OpenAI API / Local Model
    ↓ (store embeddings)
PostgreSQL + pgvector (vector search)
    ↓
Return to user
```

Each database has one job. Mixing them kills performance.

---

## 3. Code Examples

### 3.1 PostgreSQL Schema for AI Documents

```sql
-- Create pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    api_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Documents table (structured metadata)
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(256) NOT NULL,
    content TEXT NOT NULL,
    source_url VARCHAR(2048),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_title CHECK (LENGTH(title) > 0)
);

-- Document chunks (for RAG, search within documents)
CREATE TABLE chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_num INTEGER NOT NULL,  -- Which chunk in the document
    content TEXT NOT NULL,
    start_char INTEGER NOT NULL,  -- Position in original document
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Embeddings (vectors for semantic search)
CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    chunk_id INTEGER NOT NULL REFERENCES chunks(id) ON DELETE CASCADE,
    embedding vector(1536),  -- OpenAI text-embedding-3-small dimension
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Chat sessions
CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_interaction TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Chat messages (conversation history)
CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    session_id INTEGER NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role VARCHAR(10) NOT NULL,  -- 'user' or 'assistant'
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes (critical for performance)
CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_chunks_document_id ON chunks(document_id);
CREATE INDEX idx_embeddings_chunk_id ON embeddings(chunk_id);
CREATE INDEX idx_messages_session_id ON messages(session_id);

-- Vector index for semantic search (HNSW = Hierarchical Navigable Small World)
CREATE INDEX idx_embeddings_vector ON embeddings USING hnsw (embedding vector_cosine_ops);

-- Full-text search index (for text search)
ALTER TABLE documents ADD COLUMN search_tsvector tsvector;
CREATE INDEX idx_documents_fts ON documents USING gin(search_tsvector);

-- Update trigger to keep search vector current
CREATE OR REPLACE FUNCTION documents_update_search()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_tsvector := to_tsvector('english', NEW.title || ' ' || NEW.content);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER documents_search_trigger
BEFORE INSERT OR UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION documents_update_search();
```

### 3.2 SQLAlchemy 2.0 Async ORM

```python
from sqlalchemy import create_engine, select, insert
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base, relationship, sessionmaker
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, TIMESTAMP
from datetime import datetime
from typing import List

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id: int = Column(Integer, primary_key=True)
    email: str = Column(String(255), unique=True, nullable=False)
    api_key: str = Column(String(255), unique=True)
    created_at: datetime = Column(TIMESTAMP, default=datetime.utcnow)
    
    # Relationships
    documents: List["Document"] = relationship(
        "Document",
        back_populates="user",
        cascade="all, delete-orphan"
    )
    sessions: List["Session"] = relationship(
        "Session",
        back_populates="user",
        cascade="all, delete-orphan"
    )

class Document(Base):
    __tablename__ = "documents"
    
    id: int = Column(Integer, primary_key=True)
    user_id: int = Column(Integer, ForeignKey("users.id"), nullable=False)
    title: str = Column(String(256), nullable=False)
    content: str = Column(Text, nullable=False)
    created_at: datetime = Column(TIMESTAMP, default=datetime.utcnow)
    
    # Relationships
    user = relationship("User", back_populates="documents")
    chunks: List["Chunk"] = relationship(
        "Chunk",
        back_populates="document",
        cascade="all, delete-orphan"
    )

class Chunk(Base):
    __tablename__ = "chunks"
    
    id: int = Column(Integer, primary_key=True)
    document_id: int = Column(Integer, ForeignKey("documents.id"), nullable=False)
    chunk_num: int = Column(Integer, nullable=False)
    content: str = Column(Text, nullable=False)
    
    # Relationships
    document = relationship("Document", back_populates="chunks")
    embeddings: List["Embedding"] = relationship(
        "Embedding",
        back_populates="chunk",
        cascade="all, delete-orphan"
    )

class Embedding(Base):
    __tablename__ = "embeddings"
    
    id: int = Column(Integer, primary_key=True)
    chunk_id: int = Column(Integer, ForeignKey("chunks.id"), nullable=False)
    embedding = Column("embedding", None)  # Vector type, specialized
    
    # Relationships
    chunk = relationship("Chunk", back_populates="embeddings")

class Message(Base):
    __tablename__ = "messages"
    
    id: int = Column(Integer, primary_key=True)
    session_id: int = Column(Integer, ForeignKey("sessions.id"), nullable=False)
    role: str = Column(String(10), nullable=False)  # 'user' or 'assistant'
    content: str = Column(Text, nullable=False)
    created_at: datetime = Column(TIMESTAMP, default=datetime.utcnow)

class Session(Base):
    __tablename__ = "sessions"
    
    id: int = Column(Integer, primary_key=True)
    user_id: int = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at: datetime = Column(TIMESTAMP, default=datetime.utcnow)
    
    # Relationships
    user = relationship("User", back_populates="sessions")
    messages: List[Message] = relationship(
        "Message",
        cascade="all, delete-orphan"
    )

# Database setup
DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/aiapp"
engine = create_async_engine(
    DATABASE_URL,
    echo=False,  # Set to True for SQL debugging
    pool_size=20,
    max_overflow=10
)

async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# Dependency for FastAPI
async def get_db() -> AsyncSession:
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

# Usage examples
async def save_document(
    user_id: int,
    title: str,
    content: str,
    db: AsyncSession
) -> Document:
    """Save document (and return it)."""
    doc = Document(
        user_id=user_id,
        title=title,
        content=content
    )
    db.add(doc)
    await db.commit()
    await db.refresh(doc)  # Reload with auto-generated ID
    return doc

async def search_similar_documents(
    user_id: int,
    query_embedding: List[float],
    limit: int = 5,
    db: AsyncSession = None
) -> List[Chunk]:
    """
    Find chunks similar to query embedding using cosine distance.
    Lower distance = more similar.
    """
    from sqlalchemy import func, text
    
    # Vector similarity search using PostgreSQL cosine distance
    stmt = select(Chunk).from_statement(
        text("""
            SELECT c.* FROM chunks c
            JOIN embeddings e ON e.chunk_id = c.id
            WHERE (e.embedding <-> :query) < 0.5  -- Cosine distance < 0.5
            ORDER BY e.embedding <-> :query
            LIMIT :limit
        """)
    ).param(
        query=query_embedding,
        limit=limit
    )
    
    result = await db.execute(stmt)
    return result.scalars().all()

async def bulk_insert_chunks(
    document_id: int,
    chunks: List[str],
    db: AsyncSession
) -> None:
    """Efficiently insert many chunks."""
    chunk_rows = [
        Chunk(
            document_id=document_id,
            chunk_num=i,
            content=chunk_text
        )
        for i, chunk_text in enumerate(chunks)
    ]
    db.add_all(chunk_rows)
    await db.commit()
```

### 3.3 Alembic Migrations

```bash
# Initialize Alembic
alembic init alembic

# Create a migration
alembic revision --autogenerate -m "Create initial schema"

# Apply migrations
alembic upgrade head

# Rollback one migration
alembic downgrade -1
```

**alembic/env.py** (for async):
```python
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import create_async_engine

def run_async_migrations():
    """Run migrations in async context."""
    connectable = AsyncEngine(
        create_async_engine(
            config.get_main_option("sqlalchemy.url"),
            poolclass=pool.NullPool,
        )
    )

    with connectable.begin() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        
        with context.begin_transaction():
            context.run_migrations()
```

### 3.4 MongoDB for Unstructured Data

```python
from motor.motor_asyncio import AsyncClient, AsyncDatabase
from typing import List, Dict, Any

class MongoDBClient:
    def __init__(self, connection_string: str):
        self.client = AsyncClient(connection_string)
        self.db: AsyncDatabase = self.client['aiapp']
    
    async def save_raw_document(
        self,
        user_id: int,
        content: str,
        metadata: Dict[str, Any]
    ) -> str:
        """Save document without schema constraints."""
        doc = {
            'user_id': user_id,
            'content': content,
            'metadata': metadata,
            'created_at': datetime.utcnow()
        }
        result = await self.db['raw_documents'].insert_one(doc)
        return str(result.inserted_id)
    
    async def search_documents(
        self,
        user_id: int,
        query: str
    ) -> List[Dict]:
        """Full-text search across documents."""
        results = await self.db['raw_documents'].find(
            {
                'user_id': user_id,
                '$text': {'$search': query}
            }
        ).to_list(limit=10)
        return results

# Usage
mongo = MongoDBClient("mongodb+srv://user:pass@cluster.mongodb.net/")
doc_id = await mongo.save_raw_document(
    user_id=123,
    content="...",
    metadata={'source': 'api', 'version': 1}
)
```

### 3.5 Redis for Caching

```python
import redis.asyncio as redis
import json
from datetime import timedelta

class RedisCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = None
        self.redis_url = redis_url
    
    async def connect(self):
        self.redis = await redis.from_url(self.redis_url)
    
    async def cache_llm_response(
        self,
        prompt: str,
        response: str,
        ttl_hours: int = 1
    ) -> None:
        """Cache LLM response to avoid recomputation."""
        # Use deterministic hash of prompt as key
        import hashlib
        key = f"llm:{hashlib.md5(prompt.encode()).hexdigest()}"
        
        await self.redis.setex(
            key,
            timedelta(hours=ttl_hours),
            response
        )
    
    async def get_cached_response(self, prompt: str) -> str:
        """Retrieve cached LLM response."""
        import hashlib
        key = f"llm:{hashlib.md5(prompt.encode()).hexdigest()}"
        
        result = await self.redis.get(key)
        return result.decode() if result else None
    
    async def save_session_state(
        self,
        session_id: str,
        conversation: List[Dict]
    ) -> None:
        """Store conversation in Redis."""
        key = f"session:{session_id}"
        await self.redis.setex(
            key,
            timedelta(hours=24),  # Expire after 24 hours
            json.dumps(conversation)
        )
    
    async def close(self):
        if self.redis:
            await self.redis.close()

# Usage
cache = RedisCache()
await cache.connect()

# Cache expensive operation
response = await fetch_from_llm("What is AI?")
await cache.cache_llm_response("What is AI?", response)

# Later, avoid re-fetching
cached = await cache.get_cached_response("What is AI?")
if cached:
    response = cached  # Use cached, don't call API
```

---

## 4. Architecture Diagram: AI Data Flow

```mermaid
graph TB
    A["FastAPI Request"] --> B{"Data Type?"}
    B -->|"Structured"| C["PostgreSQL<br/>User, Sessions"]
    B -->|"Raw Documents"| D["MongoDB<br/>Raw content"]
    B -->|"Embeddings"| E["PostgreSQL + pgvector<br/>Semantic search"]
    B -->|"Cache"| F["Redis<br/>LLM responses, sessions"]
    C --> G["Async Queries<br/>SQLAlchemy ORM"]
    D --> H["Full-text search<br/>Aggregation pipelines"]
    E --> I["Vector similarity<br/>Cosine distance"]
    F --> J["TTL expiration<br/>One-liner caching"]
    G --> K["Return to user"]
    H --> K
    I --> K
    J --> K
```

---

## 5. Tools Section: Database Comparison

| Feature | PostgreSQL | MongoDB | Redis |
|---------|-----------|---------|--------|
| **Data Type** | Structured (rows) | Unstructured (docs) | Key-value |
| **Queries** | SQL (JOINs) | MongoDB Query Lang | Simple get/set |
| **ACID** | Full ACID | Multi-doc ACID | None (cache) |
| **Scaling** | Vertical (bigger server) | Horizontal (sharding) | Replication |
| **Schema** | Rigid | Flexible | None |
| **Vector Search** | pgvector extension | MongoDB Atlas Search | N/A |
| **Full-text Search** | GiST/GIN indexes | Full-text search | N/A |
| **Persistence** | On disk | On disk | Optional (RDB/AOF) |
| **Latency** | 10-100ms | 10-100ms | <1ms |

---

## 6. Decision Framework: When to Use Each

| Scenario | Database | Why |
|----------|----------|-----|
| User authentication, transactions | PostgreSQL | ACID guarantees |
| Vector semantic search | PostgreSQL + pgvector | Native support, indexes |
| Raw documents, logs | MongoDB | Schema flexibility |
| Conversation history (temp) | Redis | Speed, automatic expiry |
| LLM response cache | Redis | Sub-millisecond lookups |
| Analytical queries (aggregation) | MongoDB | Aggregation pipeline |
| Full-text search | PostgreSQL GIN index | Fast phrase search |
| Session state (temporary) | Redis | Distributed sessions |

---

## 7. Step-by-Step: Setting Up AI Database Stack

### Step 1: Install Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: aiengineuser
      POSTGRES_PASSWORD: password
      POSTGRES_DB: aiapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
  
  postgres-init:
    image: postgres:15
    depends_on:
      - postgres
    command: |
      sh -c 'psql -h postgres -U aiengineuser -d aiapp -c "CREATE EXTENSION IF NOT EXISTS vector;"'
    environment:
      PGPASSWORD: password
  
  mongodb:
    image: mongo:6
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodata:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  mongodata:
  redisdata:
```

### Step 2: Start Services

```bash
docker-compose up -d
# Wait 10 seconds for postgres to initialize

# Verify PostgreSQL
psql postgres://aiengineuser:password@localhost:5432/aiapp -c "SELECT version();"

# Verify MongoDB
mongosh mongodb://admin:password@localhost:27017/

# Verify Redis
redis-cli ping  # Should output "PONG"
```

### Step 3: Run Alembic Migrations

```bash
alembic upgrade head
```

### Step 4: Test Connections

```python
import asyncio
from databases import Database

async def test():
    # PostgreSQL
    db = Database("postgresql://aiengineuser:password@localhost/aiapp")
    await db.connect()
    result = await db.fetch_one("SELECT 1;")
    print(f"PostgreSQL: {result}")
    await db.disconnect()
    
    # MongoDB
    from motor.motor_asyncio import AsyncClient
    mongo = AsyncClient("mongodb://admin:password@localhost:27017/")
    print(f"MongoDB: {await mongo.server_info()}")
    mongo.close()
    
    # Redis
    import redis.asyncio as redis
    r = await redis.from_url("redis://localhost:6379")
    await r.set("test", "value")
    print(f"Redis: {await r.get('test')}")
    await r.close()

asyncio.run(test())
```

---

## 8. Practical Project: Complete RAG Database Schema

**Goal:** Build a document storage system ready for RAG (Retrieval-Augmented Generation).

**Full implementation:**

```python
from datetime import datetime
from typing import List, Optional
from sqlalchemy import select, and_
import numpy as np

class DocumentStore:
    """Handle document storage and retrieval for RAG."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create_user(self, email: str, api_key: str) -> User:
        """Create a user."""
        user = User(email=email, api_key=api_key)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
    
    async def store_document_with_embeddings(
        self,
        user_id: int,
        title: str,
        content: str,
        embeddings: List[List[float]]
    ) -> Document:
        """Store document and its chunk embeddings."""
        # Create document
        doc = Document(user_id=user_id, title=title, content=content)
        self.db.add(doc)
        await self.db.flush()  # Get auto-ID without committing yet
        
        # Split content into chunks
        chunk_size = 512
        chunks_list = []
        for i in range(0, len(content), chunk_size):
            chunk = Chunk(
                document_id=doc.id,
                chunk_num=i // chunk_size,
                content=content[i:i+chunk_size],
                start_char=i
            )
            chunks_list.append(chunk)
            self.db.add(chunk)
        
        await self.db.flush()
        
        # Store embeddings
        for chunk, embedding in zip(chunks_list, embeddings):
            emb = Embedding(
                chunk_id=chunk.id,
                embedding=embedding
            )
            self.db.add(emb)
        
        await self.db.commit()
        return doc
    
    async def semantic_search(
        self,
        user_id: int,
        query_embedding: List[float],
        limit: int = 5
    ) -> List[Chunk]:
        """
        Find most similar chunks using vector distance.
        """
        from sqlalchemy import func, text
        
        # Use PostgreSQL cosine similarity
        stmt = select(Chunk).join(Embedding).where(
            Chunk.document.has(Document.user_id == user_id)
        ).order_by(
            func.l2_distance(Embedding.embedding, query_embedding)
        ).limit(limit)
        
        result = await self.db.execute(stmt)
        return result.scalars().all()
    
    async def hybrid_search(
        self,
        user_id: int,
        keyword_query: str,
        query_embedding: List[float],
        limit: int = 5
    ) -> List[Chunk]:
        """
        Combine keyword search + semantic search.
        """
        # Keyword match
        keyword_stmt = select(Chunk).join(Document).where(
            and_(
                Document.user_id == user_id,
                Chunk.content.ilike(f"%{keyword_query}%")
            )
        )
        keyword_results = await self.db.execute(keyword_stmt)
        keyword_chunks = set(keyword_results.scalars().all())
        
        # Semantic match
        semantic_chunks = await self.semantic_search(
            user_id,
            query_embedding,
            limit=limit*2
        )
        
        # Union, then rank by relevance
        combined = list(keyword_chunks | set(semantic_chunks))[:limit]
        return combined
```

---

## 9. Debugging Playbook: 10 Database Errors

### Error 1: "ERROR: extension "vector" does not exist"

**Symptom:** CREATE TABLE with `vector` column fails.

**Explanation:** pgvector extension not installed.

**Fix:**
```sql
CREATE EXTENSION vector;
-- Then create table
```

**Prevention:** Use database initialization script that installs extension first.

---

### Error 2: "FATAL: remaining connection slots are reserved"

**Symptom:** Can't connect to PostgreSQL.

**Explanation:** All 100 default connections exhausted.

**Fix:**
```python
# Increase pool size
engine = create_async_engine(
    DATABASE_URL,
    pool_size=50,  # Larger pool
    max_overflow=20  # Allow extra under load
)

# Also configure PostgreSQL max_connections
ALTER SYSTEM SET max_connections = 200;
```

**Prevention:** Monitor connection count. Adjust pool size based on concurrent users.

---

### Error 3: "relation does not exist"

**Symptom:** Query fails: "ERROR: relation 'documents' does not exist"

**Explanation:** Table not created (migration not run).

**Fix:**
```bash
alembic upgrade head
```

**Prevention:** Run migrations in deployment script before starting app.

---

### Error 4: "duplicate key value violates unique constraint"

**Symptom:** Insert fails on unique field.

**Explanation:** Value already exists (e.g., duplicate email).

**Fix:**
```python
# Check before insert
existing = await db.execute(
    select(User).where(User.email == email)
)
if existing.scalars().first():
    raise ValueError("Email already exists")

# Or use upsert
stmt = insert(User).values(email=email).on_conflict_do_update(
    index_elements=['email'],
    set_=dict(email=email)
)
```

**Prevention:** Validate unique constraints in application logic.

---

### Error 5: "AsyncSession.execute() requires an awaitable"

**Symptom:** Forgot `await` on database call.

**Explanation:** AsyncSession methods return coroutines, need `await`.

**Fix:**
```python
# Wrong
result = db.execute(select(...))

# Right
result = await db.execute(select(...))
```

**Prevention:** Type hints catch this (mypy warns).

---

### Error 6: "No module named 'asyncpg'"

**Symptom:** `ImportError: No module named 'asyncpg'`

**Explanation:** Missing async PostgreSQL driver.

**Fix:**
```bash
pip install asyncpg
```

**Prevention:** Install full dependencies: `pip install -e ".[dev]"`

---

### Error 7: "Vector indexes are not being used"

**Symptom:** Vector search is slow (full table scan).

**Explanation:** Index not created or query doesn't match index.

**Fix:**
```sql
-- Create HNSW index (better than default)
CREATE INDEX idx_embeddings_hnsw ON embeddings 
USING hnsw (embedding vector_cosine_ops);

-- For exact nearest neighbor (slower but accurate)
CREATE INDEX idx_embeddings_ivfflat ON embeddings 
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

**Prevention:** Monitor EXPLAIN ANALYZE output to see if index is used.

---

### Error 8: "Connection timeout"

**Symptom:** Queries hang, then timeout after 30 seconds.

**Explanation:** Database server unreachable or overloaded.

**Fix:**
```python
# Add timeout to connection
engine = create_async_engine(
    DATABASE_URL,
    connect_args={"timeout": 10}  # Timeout after 10 seconds
)

# Check database is running
docker ps | grep postgres
```

**Prevention:** Implement health checks. Monitor database responsiveness.

---

### Error 9: "NO_SUCH_TABLE"

**Symptom:** MongoDB query returns no documents from non-existent collection.

**Explanation:** MongoDB creates collections on first write, but query assumes it exists.

**Fix:**
```python
# Check collection exists before querying
collections = await db.list_collection_names()
if 'documents' not in collections:
    # Create with validation schema
    await db.create_collection('documents')
```

**Prevention:** Initialize collections on app startup.

---

### Error 10: "Redis connection error"

**Symptom:** `redis.ConnectionError: Connection refused`

**Explanation:** Redis not running or wrong URL.

**Fix:**
```bash
# Check Redis is running
redis-cli ping  # Should return PONG

# Check URL format
# Should be: redis://localhost:6379 (not redis://localhost:6379:6379)
```

**Prevention:** Use environment variables for connection strings with defaults.

---

## 10. Common Mistakes: Database Design

### Mistake 1: No Indexes (Everything is Slow)

```python
# Wrong: query without index
SELECT * FROM documents WHERE user_id = 123;  # Scans entire table

# Right: create index
CREATE INDEX idx_documents_user_id ON documents(user_id);
```

**Impact:** Queries 100x slower without indexes.

---

### Mistake 2: Storing Embeddings as JSON

```python
# Wrong: embeddings as JSON string
embedding: str = "[0.1, 0.2, 0.3]"  # Stored as text

# Right: native vector type
embedding = Column(Vector(1536))  # pgvector extension
```

**Impact:** Vector search impossible. Can't use indexes.

---

### Mistake 3: Not Using Connection Pooling

```python
# Wrong: create new connection per query
engine = create_async_engine(DATABASE_URL, pool_size=1)

# Right: pool connections
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10
)
```

**Impact:** 10x slower under concurrent load.

---

### Mistake 4: Synchronous Database Calls in Async Code

```python
# Wrong: blocks event loop
async def get_docs():
    with db.connect() as conn:  # Synchronous!
        result = conn.execute("SELECT ...")

# Right: async throughout
async def get_docs():
    async with AsyncSession() as session:
        result = await session.execute(select(...))
```

**Impact:** One slow query blocks all other requests.

---

### Mistake 5: Loading Entire Collections Before Filtering

```python
# Wrong: load all, filter in Python
chunks = await db.execute(select(Chunk))
relevant = [c for c in chunks if len(c.content) > 100]

# Right: filter in database
stmt = select(Chunk).where(func.length(Chunk.content) > 100)
relevant = await db.execute(stmt)
```

**Impact:** Tons of data transfer. Out-of-memory errors.

---

## 11. Production Realism: Schema Evolution

### Tutorial Schema

```python
class Document(Base):
    __tablename__ = "documents"
    
    id = Column(Integer, primary_key=True)
    content = Column(Text)
    # No metadata, timestamps, or versioning
```

### Production Schema

```python
class Document(Base):
    __tablename__ = "documents"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    title = Column(String(256), nullable=False)
    content = Column(Text, nullable=False)
    
    # Metadata
    source_url = Column(String(2048))
    source_type = Column(String(50))  # 'api', 'upload', 'web'
    
    # Versioning
    version = Column(Integer, default=1)
    # previous_version_id = Column(Integer)  # Track changes
    
    # Audit trail
    created_at = Column(TIMESTAMP, server_default=func.now())
    updated_at = Column(TIMESTAMP, server_default=func.now(), onupdate=func.now())
    created_by = Column(Integer, ForeignKey("users.id"))
    updated_by = Column(Integer, ForeignKey("users.id"))
    
    # Soft deletes (don't actually delete, mark as deleted)
    is_deleted = Column(Boolean, default=False)
    deleted_at = Column(TIMESTAMP)
```

**Differences:**
- Audit columns (who, when)
- Soft deletes (compliance, recovery)
- Metadata field
- Versioning support
- Foreign key for user (authorization)

---

## 12. Cost & Performance: Scaling Considerations

### Query Performance

```python
# BAD: O(N) full table scan
SELECT * FROM documents WHERE content ILIKE '%keyword%';

# GOOD: O(1) with full-text index
CREATE INDEX idx_documents_fts ON documents USING gin(search_tsvector);
SELECT * FROM documents WHERE search_tsvector @@ to_tsquery('keyword');

# BEST: For billions of rows, use Elasticsearch or Apache Solr
```

### Storage Costs

```python
# Vector storage is expensive (1M vectors × 1536 dim × 4 bytes = 6GB)
embedding_size = 1536
num_vectors = 1_000_000
storage_mb = (num_vectors * embedding_size * 4) / (1024**2)
print(f"Storage needed: {storage_mb:.0f} MB (~${storage_mb * 0.02 / 1000})")  # ~$0.12/month
```

### Connection Pool Sizing

```python
# Rule of thumb: pool_size = (workers × connections_per_worker) + buffer
# If 8 CPU workers, each using 5 connections: 8 × 5 + 20 = 60
pool_size = 60
max_overflow = 20  # Allow temporary spikes
```

---

## 13. Security Considerations

### SQL Injection (Prevented by SQLAlchemy)

```python
# Wrong: string formatting (vulnerable)
query = f"SELECT * FROM users WHERE email = '{email}'"

# Right: parameterized (safe)
stmt = select(User).where(User.email == email)
# SQLAlchemy handles escaping
```

### Encryption at Rest

```python
# Sensitive fields should be encrypted
from sqlalchemy import Text
from cryptography.fernet import Fernet

class User(Base):
    # Encrypt API keys before storing
    api_key_encrypted = Column(Text)

# On save
cipher = Fernet(key)
encrypted = cipher.encrypt(api_key.encode())
user.api_key_encrypted = encrypted

# On retrieve
decrypted = cipher.decrypt(user.api_key_encrypted).decode()
```

### Row-Level Security (PostgreSQL)

```sql
-- Users can only see their own documents
CREATE POLICY user_documents ON documents
    USING (user_id = current_user_id);

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```

---

## 14. Case Study: Notion's Database Design

**Why Notion matters:** Stores billions of documents flexibly.

**Their patterns:**
- MongoDB for document flexibility
- PostgreSQL for transactions (user billing)
- Redis for real-time sync
- Elasticsearch for search

**Lesson:** Hybrid database approach. Each DB does one thing well.

---

## 15. Interview Cheat Sheet: Databases

### Concept 1: Indexing

**Q:** Why is indexing important?

**A:** Indexes make lookups O(log N) instead of O(N). For 1M rows, difference between 1ms and 1 second.

---

### Concept 2: ACID Properties

**Q:** What are ACID properties?

**A:**
- **Atomicity:** Transaction all-or-nothing
- **Consistency:** Data valid before/after
- **Isolation:** Concurrent txns don't interfere
- **Durability:** Committed data survives failures

---

### Concept 3: Vector Distance Metrics

**Q:** What's cosine distance in embeddings?

**A:** Measures angle between two vectors. 0 = identical, 1 = orthogonal. Use for semantic search.

---

### Concept 4: Connection Pooling

**Q:** Why use connection pooling?

**A:** Creating DB connections is expensive. Reuse them. Pool holds N ready-to-use connections.

---

### Concept 5: N+1 Query Problem

**Q:** What's the N+1 problem?

**A:** Loading 100 users, then looping to load each user's documents = 101 queries. Use JOINs or eager loading.

---

## 16. Review Questions

1. Design a PostgreSQL schema for storing documents and embeddings. Include indexes.
2. What's the difference between `pool_size` and `max_overflow` in SQLAlchemy?
3. Write an async function that searches documents by embedding similarity.
4. Explain why MongoDB is better than PostgreSQL for unstructured data.
5. How do you prevent SQL injection in SQLAlchemy?
6. What's a full-text search index? When would you use it?
7. Write a migration that adds a `vector` column to an embeddings table.
8. Explain row-level security in PostgreSQL.
9. What's the N+1 query problem? How do you fix it?
10. Design a Redis caching strategy for an LLM API.

---

## 17. Flashcards

### Card 1
**Front:** What should `pool_size` be for a production app?

**Back:** `pool_size = (number_of_workers × connections_per_worker) + buffer`. Example: 8 workers × 5 connections = 40 + 20 buffer = 60.

---

### Card 2
**Front:** When should you use MongoDB over PostgreSQL?

**Back:** When data is unstructured and schema changes frequently. Documents, logs, messy JSON. PostgreSQL for transactions and relationships.

---

### Card 3
**Front:** What is cosine similarity in embeddings?

**Back:** Measure of angle between vectors. 0 = identical direction, 1 = perpendicular. Used for semantic search.

---

### Card 4
**Front:** How do you prevent N+1 queries in SQLAlchemy?

**Back:** Use `selectinload` or `joinedload` to eager-load relationships. Or write a single JOIN query instead of looping.

---

### Card 5
**Front:** Why is Full-Text Search (GIN index) better than ILIKE?

**Back:** ILIKE is O(N) full table scan. FTS with GIN is O(log N) + phrase matching, stemming, etc. Much faster.

---

### Card 6
**Front:** What does `await db.refresh(obj)` do?

**Back:** Reloads object from database. Needed to get auto-generated IDs after insert.

---

### Card 7
**Front:** When should Redis cache expire?

**Back:** Depends on data. LLM responses: 1 hour. Session state: 24 hours. Temporary state: 5 minutes.

---

### Card 8
**Front:** Why use Redis instead of application memory cache?

**Back:** Redis is distributed. Multiple app instances share cache. App memory cache is lost on restart.

---

### Card 9
**Front:** What's an HNSW index in postgresql?

**Back:** Hierarchical Navigable Small World. Fast approximate nearest neighbor search for vectors. Better than brute force.

---

### Card 10
**Front:** How do you handle connection timeouts?

**Back:** Use `connect_args={"timeout": 10}`. If DB doesn't respond in 10s, raise exception. Prevents hanging requests.

---

## 18. Teach-It-Back Prompt

Explain to a junior engineer why a vector database is necessary for RAG systems.

**Model Answer:**

"RAG retrieves relevant documents for context. If you query 'What is AI?' you need documents semantically similar to that phrase, not just exact word matches. Vectors represent meaning in a mathematical space. Similar meaning = close vectors. Vector databases like pgvector let you find nearest neighbors efficiently. Without it, you'd search every document (slow) or miss relevant docs (bad answers)."

---

## 19. 1-Week Review Checklist

- [ ] Set up PostgreSQL + pgvector locally
- [ ] Create a schema with users, documents, chunks, embeddings
- [ ] Write async SQLAlchemy queries with relationships
- [ ] Create a Alembic migration
- [ ] Store and search embeddings using cosine similarity

---

## 20. Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [SQLAlchemy 2.0 Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic Migrations](https://alembic.sqlalchemy.org/)
- [MongoDB Async Motor](https://motor.readthedocs.io/)
- [Redis Async](https://redis-py.readthedocs.io/)
- [Database Optimization](https://use-the-index-luke.com/)

---

## 21. Next: Linux & Operations

Databases run on servers. Time to learn how to manage them.

→ **[Linux & Networking (next) →](linux.md)**
