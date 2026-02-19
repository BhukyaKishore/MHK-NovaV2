# MHK Enterprise Chatbot V2 — Single Agent Architecture

> Production-ready, fault-tolerant chatbot powered by a single LangGraph agent with specialized tools — meeting scheduling, job search, and RAG-powered Q&A — built on FastAPI + Qdrant.

---

## What is New in V2

| Feature | V1 | V2 |
|---------|----|---|
| Architecture | Single-agent RAG chatbot | Single LangGraph agent with specialized tools |
| Meeting Scheduler | Not available | OTP verification > form > HR email (schedule_meeting tool) |
| Job Search | Not available | Cached JSON + API fallback (search_jobs tool) |
| Conversation Memory | Basic | ConversationBufferMemory with follow-up support |
| Confidence Gate | Not available | Intent routing only above 85/100 threshold |
| Observability | Logging only | Structured logging + metrics + tracing |

---

## Architecture Overview

```
+-------------------------------------------------------------+
|                    FRONTEND (React/Next.js)                   |
|  ChatWindow | OTP Form | Scheduler Form | Job Listings       |
+---------------------------------+---------------------------+
                                  | REST / WebSocket
                                  v
+-------------------------------------------------------------+
|                   API GATEWAY (FastAPI)                       |
|  Rate Limiter | Auth | CORS | Input Sanitizer                |
+---------------------------------+---------------------------+
                                  |
+---------------------------------v---------------------------+
|               LANGGRAPH AGENT (Single Agent)                 |
|                                                              |
|   Session Node > Intent Node > Confidence Gate (>0.85)       |
|   > Tool Selection Node > Response Node                      |
|                                                              |
|  +-------------+  +--------------+  +--------------------+   |
|  | TOOLS       |  | SERVICES     |  | RESILIENCE         |   |
|  | schedule_   |  | OTP, Email,  |  |    LAYER           |   |
|  | meeting     |  | Job Cache,   |  | Circuit Breaker    |   |
|  | search_jobs |  | RAG Pipeline |  | Retry + Backoff    |   |
|  | query_kb    |  |              |  | Dead Letter Queue  |   |
|  | ingest_doc  |  |              |  |                    |   |
|  +------+------+  +------+------+  +--------------------+   |
+---------+----------------+-----------------------------------+
          |                |
    +-----+-----+    +----+----+    +----------+
    |  Qdrant   |    | OpenAI  |    |  Email   |
    | Vector DB |    |  GPT-4  |    |  (SMTP)  |
    +-----------+    +---------+    +----------+
```

### Key Components

| Component | Role | Key Features |
|-----------|------|-------------|
| **LangGraph Agent** | Session management, intent detection, tool selection | State machine, confidence gate (>0.85), conditional edges, re-evaluation loops |
| **Tools** | Execute specific workflows | schedule_meeting, search_jobs, query_kb, ingest_document, send_otp, verify_otp, send_email |
| **Services** | Shared business logic used by tools | OTP service, email service, job cache, RAG pipeline |

---

## How It Works

### Master Flow

```
User Message
    |
    v
LangGraph Agent
    |
    +-- Intent Detection Node (LLM)
         |
         +-- Confidence > 0.85? --> YES --> Tool Selection Node
         |                     --> NO  --> Clarification Node --> Retry
         |
         +-- "meeting_scheduler" --> schedule_meeting tool
         |                           +-- Ask email --> Send OTP --> Verify
         |                           +-- Show form (name, phone, email, subject, msg)
         |                           +-- Send HR email --> Confirm to user
         |
         +-- "job_search" --> search_jobs tool
         |                    +-- Check cache (< 1hr?)
         |                    +-- Fetch from API if stale
         |                    +-- Filter & answer from job data
         |
         +-- "general_qa" --> query_kb tool
                               +-- Load conversation history
                               +-- Embed query --> Search Qdrant
                               +-- Build context + history
                               +-- Generate response (GPT-4)

Document Upload (independent pipeline):
    POST /api/v2/documents/upload --> ingest_document tool --> Process & Store
```

### Meeting Scheduler Flow (Detailed)

```
"I want to schedule a demo"
    |
    v
Ask for email --> "user@email.com"
    |
    v
Send OTP (6-digit, expires in 5 min)
    |
    v
User enters OTP --> Verify (max 3 attempts)
    |
    v  [PASS] Verified
Show form in chat:
  +----------------------------+
  | Name:     [__________]     |
  | Phone:    [__________]     |
  | Email:    user@email.com   | (pre-filled)
  | Subject:  [__________]     |
  | Message:  [__________]     |
  |                            |
  |     [Submit Request]       |
  +----------------------------+
    |
    v
Validate --> Send email to HR --> "Meeting request sent. Our team will get back to you."
```

---

## Confidence Gate

After intent classification, the agent enforces a **confidence gate at 85/100 (0.85)**:

- **Confidence >= 0.85**: Tool Selection Node proceeds to the detected intent's tool.
- **Confidence < 0.85**: Clarification Node asks the user to refine their message. Re-classifies after clarification. Repeats until threshold is met.

This prevents incorrect tool selection and ensures the system only acts when it has high certainty about user intent.

---

## Fault Tolerance

| Pattern | Applied To | Behavior |
|---------|-----------|----------|
| **Circuit Breaker** | OpenAI, Qdrant, SMTP | Opens after 5 failures, auto-recovers after 30s |
| **Retry + Backoff** | All external calls | Exponential backoff (1s>2s>4s), max 3 retries |
| **Dead Letter Queue** | Failed operations | Stored for retry/manual review, TTL 24hrs |
| **Graceful Degradation** | Service outages | Stale cache, keyword fallback, queued emails |
| **Health Monitoring** | All dependencies | `/api/v2/health` with per-service status |

---

## Project Structure

```
ai-chatbot-v2/
+-- backend/
|   +-- app/
|       +-- agent/                 # Single LangGraph Agent
|       |   +-- nodes/             # Graph nodes (session, intent, confidence gate, etc.)
|       |   +-- tools/             # Tools (schedule_meeting, search_jobs, query_kb, etc.)
|       |   +-- services/          # Shared services (OTP, email, cache, validator)
|       +-- rag/                   # RAG pipeline (retriever, context, generator)
|       +-- ingestion/             # Document processing pipeline
|       +-- memory/                # Conversation buffer, session store
|       +-- resilience/            # Circuit breaker, retry, DLQ
|       +-- observability/         # Logging, metrics, tracing
|       +-- services/              # Shared LLM & vector DB clients
|       +-- models/ & schemas/     # Data models
|       +-- api/v2/                # REST endpoints
|       +-- middleware/            # Rate limit, CORS, sanitizer
+-- frontend/
|   +-- src/
|       +-- components/chatbot/
|           +-- OTPVerification/
|           +-- SchedulerForm/
|           +-- JobListings/
+-- infrastructure/                # Docker, deployment, monitoring
+-- tests/                         # Unit, integration, E2E
```

> Full structure: [PROJECT-STRUCTURE.md](./PROJECT-STRUCTURE.md)

---

## Team Division (3 Developers)

| Developer | Ownership | Key Deliverables |
|-----------|-----------|-----------------:|
| **Dev 1** | Agent Core + Graph + Infra | LangGraph setup, AgentState, all graph nodes, confidence gate (0.85), memory, circuit breakers, observability |
| **Dev 2** | Tools + Workflows + Frontend | schedule_meeting tool (OTP+form+email), search_jobs tool (cache+API), query_kb tool (RAG+memory), frontend components |
| **Dev 3** | Ingestion + Infrastructure | ingest_document tool, document pipeline, Docker setup, CI/CD, shared services, deployment |

> Full breakdown: [TEAM-DIVISION.md](./TEAM-DIVISION.md)

### Timeline

```
Week 1-2:  Foundation  -->  FastAPI + LangGraph nodes + confidence gate + RAG pipeline + Docker
Week 3-4:  Features    -->  Tool selection + scheduler tool + jobs tool + ingestion
Week 5-6:  Production  -->  Resilience + monitoring + deployment + E2E tests
```

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18+, Next.js 14+, TypeScript, Tailwind CSS, Zustand |
| **Backend** | Python 3.10+, FastAPI, Pydantic v2, LangGraph, LangChain |
| **AI/ML** | OpenAI GPT-4, text-embedding-3-small |
| **Vector DB** | Qdrant (self-hosted) |
| **Cache/Session** | Redis (or in-memory fallback) |
| **Email** | SMTP / SendGrid |
| **Infrastructure** | Docker, Docker Compose, Nginx, Azure VM |
| **Monitoring** | Prometheus, Grafana, structured JSON logging |
| **CI/CD** | GitHub Actions |

---

## Quick Start

### Prerequisites
- Docker and Docker Compose
- Node.js 18+
- Python 3.10+
- OpenAI API Key
- SMTP credentials (for email)

### Setup

```bash
# Clone
git clone <repository-url>
cd ai-chatbot-v2

# Configure
cp .env.example .env
# Edit .env with your API keys

# Start everything
make dev-start
```

### Services

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| API Docs | http://localhost:8000/docs |
| Qdrant Dashboard | http://localhost:6333/dashboard |
| Grafana (optional) | http://localhost:3001 |

---

## API Endpoints (V2)

### Chat
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v2/chat` | Send message (agent selects tool if confidence > 0.85) |
| GET | `/api/v2/chat/history/{session_id}` | Get conversation history |

### Meeting Scheduler
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v2/otp/send` | Send OTP to email |
| POST | `/api/v2/otp/verify` | Verify OTP code |
| POST | `/api/v2/scheduler/submit` | Submit meeting scheduler form |

### Jobs
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v2/jobs` | List all job openings |
| GET | `/api/v2/jobs/search?q={query}` | Search jobs |

### Documents
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v2/documents/upload` | Upload document for ingestion |
| GET | `/api/v2/documents` | List all documents |
| GET | `/api/v2/documents/status/{task_id}` | Check ingestion status |
| DELETE | `/api/v2/documents/{id}` | Delete document |

### System
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v2/health` | System health with dependency status |
| GET | `/api/v2/metrics` | Prometheus metrics |

---

## Testing

```bash
# Run all tests
make test

# Specific suites
make test-unit              # Unit tests only
make test-integration       # Integration tests
make test-e2e               # End-to-end tests

# With coverage
cd backend && pytest --cov=app --cov-report=html
```

> Full test case document: [TEST-CASES.md](./TEST-CASES.md) (100+ test scenarios)

---

## Environment Variables

```env
# API
API_HOST=0.0.0.0
API_PORT=8000
API_ENV=development

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4
OPENAI_EMBEDDING_MODEL=text-embedding-3-small

# Qdrant
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_COLLECTION_NAME=company_docs_v2

# Redis (optional)
REDIS_HOST=redis
REDIS_PORT=6379

# Email / OTP
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=chatbot@company.com
SMTP_PASSWORD=app-password
HR_EMAIL=hr@company.com
OTP_EXPIRY_SECONDS=300
OTP_MAX_ATTEMPTS=3

# Security
SECRET_KEY=your-secret-key
CORS_ORIGINS=http://localhost:3000
RATE_LIMIT=20/minute

# Confidence Gate
INTENT_CONFIDENCE_THRESHOLD=0.85

# Resilience
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_RECOVERY_TIMEOUT=30
MAX_RETRIES=3

# Cache
JOB_CACHE_TTL_SECONDS=3600
EMBEDDING_CACHE_SIZE=100
```

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Detailed system architecture with flowcharts, state machines, and design patterns |
| [PROJECT-STRUCTURE.md](./PROJECT-STRUCTURE.md) | Complete folder structure with all files and their purposes |
| [TEST-CASES.md](./TEST-CASES.md) | 100+ test scenarios across 12 categories |
| [TEAM-DIVISION.md](./TEAM-DIVISION.md) | 3-developer task breakdown with timeline and dependencies |
| [FLOWCHART.md](./FLOWCHART.md) | Visual Mermaid diagrams of all workflows |

---

## Roadmap

### V2.0 (Current)
- [x] Single LangGraph agent with specialized tools
- [x] Meeting scheduler with OTP verification
- [x] Job search with caching
- [x] Conversation memory
- [x] Confidence gate (85/100 threshold)
- [x] Fault tolerance


---

## Support

For questions or issues:
- Open a GitHub Issue
- Contact the development team
- Check the documentation index above

---

