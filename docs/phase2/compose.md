# Docker Compose for the Full AI Stack

Docker Compose lets you run your entire system locally with one command.

Instead of starting FastAPI, PostgreSQL, Redis, and Qdrant separately, you declare them once and `docker compose up` starts everything.

---

## Why This Matters

**Without Compose:** New team member gets source code. Has to manually:
- Install PostgreSQL
- Start Redis in background
- Start Qdrant server
- Guess connection strings
- Troubleshoot when nothing connects

**With Compose:** `git clone && docker compose up`. Entire system running in 2 minutes.

Plus: Matches production setup. If it works here, it works on AWS.

---

## Conceptual Explanation

**Analogy: Orchestrating a Band**

Without Compose: Musician comes to practice. They have to tune their own instrument, set their own tempo, synchronize with others manually.

With Compose: Conductor has score. Says "start at measure 1." Everyone plays together, same tempo, same key.

docker-compose.yml is the conductor. Services are musicians.

---

## Core Concepts

### Services
Named containers in your compose file (api, postgres, redis, qdrant).

### Volumes
Persistent storage. PostgreSQL data survives container restart.

### Networks
How services talk to each other. `api` can reach `postgres:5432` by hostname.

### Environment
Config passed into containers (database URL, API keys, etc).

### Dependencies
`depends_on`: Wait for another service to be healthy before starting this one.

---

## Complete Docker Compose File

```yaml
version: '3.9'

services:
  # FastAPI Application
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ai-api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://aiuser:aipass@postgres:5432/aidb
      REDIS_URL: redis://redis:6379/0
      QDRANT_URL: http://qdrant:6333
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      ENVIRONMENT: development
    volumes:
      - .:/app  # Hot reload: code changes restart app
      - ./data:/app/data  # Uploaded documents
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      qdrant:
        condition: service_started
    networks:
      - ai-network
    restart: unless-stopped

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: ai-postgres
    environment:
      POSTGRES_USER: aiuser
      POSTGRES_PASSWORD: aipass
      POSTGRES_DB: aidb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aiuser"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - ai-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: ai-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ai-network

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:latest
    container_name: ai-qdrant
    ports:
      - "6333:6333"  # HTTP API
      - "6334:6334"  # gRPC API
    volumes:
      - qdrant_storage:/qdrant/storage
    environment:
      QDRANT_API_KEY: ${QDRANT_API_KEY:-insecure-key}  # Change in production
    networks:
      - ai-network

volumes:
  postgres_data:
  redis_data:
  qdrant_storage:

networks:
  ai-network:
    driver: bridge
```

---

## Service Dependencies Explained

```yaml
depends_on:
  postgres:
    condition: service_healthy  # Wait for health check to pass
  redis:
    condition: service_healthy
  qdrant:
    condition: service_started   # Just needs to start (no health check)
```

**Why:** FastAPI starts and immediately tries to connect to DB. If DB isn't ready, connection fails.

`condition: service_healthy` means:
1. Postgres starts
2. Docker checks `pg_isready -U aiuser` every 10 seconds
3. Once it returns success, API starts
4. No race condition

---

## Environment Variables: .env File

```bash
# .env (never commit this to git)
OPENAI_API_KEY=sk-proj-xyz123
POSTGRES_PASSWORD=securepass123
QDRANT_API_KEY=my-secret-key
ENVIRONMENT=development
```

Compose reads this automatically:

```yaml
environment:
  OPENAI_API_KEY: ${OPENAI_API_KEY}  # From .env
```

---

## Volumes: Persistent Data

```yaml
volumes:
  postgres_data:/var/lib/postgresql/data  # Named volume
  redis_data:/data

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

**Named volume:** Docker manages location. Data survives `docker compose down`.

**Bind mount:** Directory on your machine.

```yaml
volumes:
  - ./data:/app/data  # Map Host ./data → Container /app/data
```

**Difference:**
- Named volume: Docker manages, portable
- Bind mount: You control path, can hot-reload code

---

## Code Example 1: Hot Reload for Development

```yaml
services:
  api:
    build: .
    volumes:
      - .:/app  # Mount entire directory
    command: uvicorn main:app --host 0.0.0.0 --reload
    environment:
      DATABASE_URL: postgresql://aiuser:aipass@postgres:5432/aidb
```

**Effect:** Change main.py → Fastapi auto-reloads instantly. No rebuild needed.

---

## Code Example 2: Override File for Production

```yaml
# docker-compose.yml (base configuration)
services:
  api:
    restart: unless-stopped
    volumes:
      - .:/app

# docker-compose.prod.yml (overrides for production)
services:
  api:
    restart: always
    volumes: []  # No hot reload in production
    environment:
      DEBUG: "false"
      LOG_LEVEL: "info"
```

**Usage:**
```bash
# Development (includes both files)
docker compose up

# Production (apply prod overrides)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Code Example 3: Health Checks

```yaml
services:
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aiuser"]
      interval: 10s       # Check every 10 seconds
      timeout: 5s         # Fail if no response in 5s
      retries: 5          # Mark unhealthy after 5 failures
      start_period: 10s   # Grace period before checks start

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s   # API needs time to boot
```

---

## Code Example 4: Network Configuration

```yaml
services:
  api:
    networks:
      - ai-network
  postgres:
    networks:
      - ai-network

networks:
  ai-network:
    driver: bridge
```

**Result:** Services can reach each other by hostname.

```python
# In FastAPI
import os
db_url = os.getenv("DATABASE_URL")
# postgresql://aiuser:aipass@postgres:5432/aidb
# Note: "postgres" is service name, Docker DNS resolves it
```

---

## Code Example 5: Building Custom Images

```yaml
services:
  api:
    build:
      context: .                # Build from current directory
      dockerfile: Dockerfile    # Which Dockerfile
      args:                     # Pass ARG to Dockerfile
        ENVIRONMENT: development

  # Alternative: pull pre-built image
  postgres:
    image: postgres:16-alpine   # Pre-built from registry
```

---

## Common Commands

```bash
# Start services in background
docker compose up -d

# View logs
docker compose logs
docker compose logs -f api           # Follow api logs only
docker compose logs --tail 50 redis  # Last 50 lines

# Execute command in running container
docker compose exec api bash
docker compose exec postgres psql -U aiuser -d aidb

# View running services
docker compose ps

# Stop services (data persists in volumes)
docker compose down

# Stop and remove volumes (DATA LOST)
docker compose down -v

# Rebuild after code changes
docker compose build
docker compose up -d

# View resource usage
docker compose stats
```

---

## Troubleshooting: Service Won't Start

```bash
# Check logs for errors
docker compose logs api

# Output might show: "Error: pg_isready failed"

# Solution: Postgres isn't healthy yet
# Check postgres status
docker compose logs postgres

# If postgres is fine: might be the depends_on condition
# Change to: condition: service_started (less stringent)
```

---

## Architecture: Docker Compose Network

```mermaid
graph TB
    subgraph compose["Docker Compose Network: ai-network"]
        API["FastAPI<br/>:8000"]
        DB["PostgreSQL<br/>:5432"]
        CACHE["Redis<br/>:6379"]
        VECTOR["Qdrant<br/>:6333"]
    end
    
    HOST["Host Machine"]
    
    HOST -->|localhost:8000| API
    HOST -->|localhost:5432| DB
    HOST -->|localhost:6379| CACHE
    HOST -->|localhost:6333| VECTOR
    
    API -->|@postgres:5432| DB
    API -->|@redis:6379| CACHE
    API -->|@qdrant:6333| VECTOR
```

---

## Tools Comparison

| Tool | Purpose | Pros | Cons | Best For |
|------|---------|------|------|----------|
| **Docker Compose** | Local multi-container | Simple, included with Docker, production-like | Single machine only | Local development |
| **Kubernetes** | Orchestration at scale | Multi-machine, auto-scaling, self-healing | Complex, overkill for dev | Production, 100+ containers |
| **Podman Compose** | Rootless containers | More secure | Less ecosystem | Kubernetes prep |

---

## Decision Framework

```
Running locally?
  ├─ YES: Docker Compose
  └─ NO: Go to AWS

Single machine needed?
  ├─ YES: Docker Compose
  └─ NO: Kubernetes

Testing production config?
  ├─ YES: Docker Compose (matches AWS setup)
  └─ NO: Minimal compose OK
```

---

## Step-by-Step: Set Up Local Stack

### 1. Create docker-compose.yml

```yaml
version: '3.9'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://aiuser:aipass@postgres:5432/aidb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: aipass
      POSTGRES_USER: aiuser
      POSTGRES_DB: aidb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aiuser"]
  
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  postgres_data:
  redis_data:
```

### 2. Create .env

```
OPENAI_API_KEY=sk-...
```

### 3. Start

```bash
docker compose up -d
```

### 4. Test

```bash
curl http://localhost:8000/health
psql postgresql://aiuser:aipass@localhost:5432/aidb
redis-cli -h localhost
```

---

## Practical Project: Production-Like Local Stack

**Build:** Docker Compose file for Phase 1 capstone.

Requirements:
- FastAPI service with hot reload
- PostgreSQL with health check and init script
- Redis with persistence
- Qdrant with volume mount
- All environment variables in .env
- Service dependencies with conditions
- Override file for production

**Deliverables:**
- docker-compose.yml (fully working)
- docker-compose.prod.yml (overrides)
- .env.example (template)
- scripts/init.sql (create tables)
- Documented commands

---

## Debugging: 10 Common Compose Errors

### 1. Service Won't Start: "depends_on condition never met"

**Symptom:** API won't start, logs show "Postgres not ready"

**Fix:**
```yaml
depends_on:
  postgres:
    condition: service_healthy  # Strict check
    # Change to:
    condition: service_started   # Loose check
```

### 2. Can't Connect to Service by Hostname

**Symptom:** `psql: could not translate host name "postgres" to address`

**Fix:** Services must be on same network:
```yaml
services:
  api:
    networks:
      - ai-network
  postgres:
    networks:
      - ai-network
```

### 3. Data Lost After `docker compose down`

**Symptom:** Run compose, add data, down, up again → data gone

**Fix:** Use volumes:
```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 4. Port Already in Use

**Symptom:** `ERROR: for postgres Cannot start service postgres: Bind for 0.0.0.0:5432`

**Fix:**
```yaml
ports:
  - "5433:5432"  # Use different port on host
```

### 5. Volume Mount Permission Denied

**Symptom:** `permission denied: /data/file.txt`

**Cause:** Container non-root user can't write to host directory.

**Fix:**
```bash
# Give directory to user
sudo chown 1000:1000 ./data

# Or in Dockerfile
RUN useradd -u 1000 appuser
```

### 6. Env Variables Not Loading

**Symptom:** Container environment empty, .env file ignored

**Fix:**
```bash
# docker compose doesn't auto-load .env in some versions
docker compose --env-file .env up

# Or use env_file:
services:
  api:
    env_file: .env
```

### 7. Hot Reload Not Working

**Symptom:** Change code, nothing happens

**Fix:**
```yaml
services:
  api:
    volumes:
      - .:/app  # Must mount entire directory
    command: uvicorn main:app --host 0.0.0.0 --reload
```

### 8. Health Check Always Failing

**Symptom:** `docker compose ps` shows "unhealthy"

**Fix:** Test command manually:
```bash
docker compose exec postgres pg_isready -U aiuser
# Should return "accepting connections"
```

### 9. Services Can't See Each Other

**Symptom:** From api, `redis-cli -h redis` fails

**Fix:** Missing network or wrong network:
```yaml
services:
  api:
    networks: [ai-network]
  redis:
    networks: [ai-network]

networks:
  ai-network:
```

### 10. Build Fails With Secrets

**Symptom:** `COPY . .` fails because .env has secrets

**Fix:** Add to .dockerignore:
```
.env
.env.local
```

---

## Case Study 1: Airbnb Docker Compose

**Challenge:** 50 engineers, each running different services locally.

**Solution:**
- Multi-file compose setup (api, db, cache, vector db separated)
- Compose override for performance (disable hot-reload in prod)
- Health checks on every service
- Volume strategy: dev mounts /src, prod is RO

**Result:** Onboarding time: 2 hours → 10 minutes

---

## Case Study 2: GitHub Copilot Local AI Stack

**Challenge:** AI developers need PostgreSQL + Redis + custom service.

**Solution:**
- Single docker-compose.yml with 4 services
- Health checks prevent race conditions
- Named volumes for persistence
- Override file for CI (no volumes, faster shutdown)

**Result:** "Clone repo and `docker compose up`" → works every time

---

## 1-Week Checklist

- [ ] Day 1: Write docker-compose.yml with all services
- [ ] Day 2: Add health checks to every service
- [ ] Day 3: Create .env and .env.example
- [ ] Day 4: Test docker compose up + dependencies
- [ ] Day 5: Create docker-compose.prod.yml overrides
- [ ] Day 6: Write init.sql for database setup
- [ ] Day 7: Document all commands, test full workflow

---

## Resources

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Compose Health Checks](https://docs.docker.com/compose/compose-file/compose-file-v3/#healthcheck)
- [Docker Networking](https://docs.docker.com/network/)

→ **[AWS Deployment →](aws.md)**
