# Multi-Agent Chatbot V2 — Team Division (3 People)

---

## Overview

Work is divided based on **ownership of tools, features, and sub-systems**, with clear boundaries to prevent merge conflicts and allow parallel development.

```
+------------------+  +------------------+  +------------------+
|   DEVELOPER 1    |  |   DEVELOPER 2    |  |   DEVELOPER 3    |
|                  |  |                  |  |                  |
|  Agent Core      |  |  Tools &         |  |  Ingestion &     |
|  + Graph + Infra |  |  Workflows + FE  |  |  Infrastructure  |
+------------------+  +------------------+  +------------------+
```

---

## Developer 1 — Agent Core, Graph, and Infrastructure

### Primary Ownership

| Component | Files |
|-----------|-------|
| Agent Core | `agent/agent.py` |
| State Schema | `agent/state_schema.py` |
| Graph Builder | `agent/graph_builder.py` |
| Session Node | `agent/nodes/session_node.py` |
| Intent Node | `agent/nodes/intent_node.py` |
| Confidence Gate Node | `agent/nodes/confidence_gate.py` |
| Tool Selection Node | `agent/nodes/tool_selection_node.py` |
| Clarification Node | `agent/nodes/clarification_node.py` |
| Re-evaluation Node | `agent/nodes/re_evaluation_node.py` |
| Response Node | `agent/nodes/response_node.py` |
| Memory System | `memory/` (all files) |
| Resilience Layer | `resilience/` (circuit_breaker, retry, DLQ, fallback) |
| Core Config | `config/` (settings, constants, env_loader) |
| Middleware | `middleware/` (all files) |
| Observability | `observability/` (logger, metrics, tracing) |

### Tasks (Phased)

#### Phase 1 — Foundation
- [ ] Set up FastAPI project structure with config management
- [ ] Define `AgentState` TypedDict with all fields
- [ ] Build LangGraph graph: session > intent > confidence gate > tool selection > response nodes
- [ ] Implement confidence gate node (threshold: 0.85)
- [ ] Implement clarification node for confidence < 0.85
- [ ] Implement session node (create/load sessions)
- [ ] Set up structured logging with request IDs
- [ ] Create middleware (CORS, rate limiting, error handling)
- [ ] Set up dependency injection framework

#### Phase 2 — Graph Coordination 
- [ ] Implement intent detection node (LLM-based classification)
- [ ] Build conditional edges with confidence gate enforcement
- [ ] Implement tool selection node (maps intent to tool)
- [ ] Implement re-evaluation node (quality threshold check)
- [ ] Build response node (format and send)
- [ ] Implement conversation memory (ConversationBufferMemory)
- [ ] Add session persistence (Redis / in-memory fallback)

#### Phase 3 — Resilience and Production 
- [ ] Implement circuit breaker for all external services
- [ ] Add retry handlers with configurable backoff
- [ ] Build dead-letter queue for failed operations
- [ ] Implement graceful degradation strategies
- [ ] Add Prometheus metrics collection
- [ ] Add health check endpoint with dependency status
- [ ] Performance optimization (caching, connection pooling)

### Tests Owned

- `test_agent.py`
- `test_intent_node.py`
- `test_confidence_gate.py`
- `test_memory.py`
- `test_circuit_breaker.py`
- `test_graph_execution.py` (integration)
- `test_resilience_e2e.py`

---

## Developer 2 — Tools, Workflows, and Frontend

### Primary Ownership

| Component | Files |
|-----------|-------|
| Meeting Scheduler Tool | `agent/tools/schedule_meeting.py` |
| Job Search Tool | `agent/tools/search_jobs.py` |
| Q&A Tool | `agent/tools/query_knowledge_base.py` |
| OTP Tools | `agent/tools/send_otp.py`, `agent/tools/verify_otp.py` |
| Email Tool | `agent/tools/send_email.py` |
| OTP Service | `agent/services/otp_service.py` |
| Email Service | `agent/services/email_service.py` |
| Job Cache Service | `agent/services/job_cache_service.py` |
| Form Validator | `agent/services/form_validator.py` |
| RAG Pipeline | `rag/` (retriever, context_builder, response_generator) |
| Prompt Templates | `services/llm/prompt_templates.py` |
| API Endpoints | `api/v2/endpoints/` (chat, scheduler, otp, jobs) |
| Frontend Components | `frontend/src/components/chatbot/` (OTP, SchedulerForm, JobListings) |
| Frontend Hooks | `frontend/src/hooks/` (useOTP, useScheduler, useJobSearch) |
| Frontend Services | `frontend/src/services/` (otpService, schedulerService, jobService) |

### Tasks (Phased)

#### Phase 1 — General Q&A Tool 
- [ ] Implement RAG retriever (semantic search in Qdrant)
- [ ] Build context builder (format retrieved chunks)
- [ ] Implement response generator (LLM with prompt template)
- [ ] Create query_knowledge_base tool (query > retrieve > generate)
- [ ] Integrate conversation buffer memory for follow-ups
- [ ] Handle follow-up questions (pronoun resolution via history)
- [ ] Handle edge cases: no results, irrelevant queries, greetings

#### Phase 2 — Meeting Scheduler Tool 
- [ ] Build OTP service (generate, send via SMTP, verify, expire)
- [ ] Hash OTPs before storage
- [ ] Implement rate limiting for OTP requests
- [ ] Build email service (SMTP/SendGrid wrapper)
- [ ] Create form validator (name, phone, email, subject, message)
- [ ] Implement schedule_meeting tool with multi-step state machine
- [ ] Build frontend OTPVerification component
- [ ] Build frontend SchedulerForm component (renders in chat)
- [ ] Handle flow interruptions (cancel, timeout, retry)
- [ ] Create HR email template

#### Phase 3 — Job Search Tool 
- [ ] Implement job cache service (in-memory/Redis with TTL)
- [ ] Build job API/DB connector
- [ ] Create search_jobs tool (cache check > fetch > filter > answer)
- [ ] Implement natural language job filtering (role, location, salary, experience)
- [ ] Build frontend JobListings/JobCard components
- [ ] Handle edge cases: no jobs, stale cache, API failure

#### Phase 4 — Integration 
- [ ] Register all tools with the LangGraph agent
- [ ] Implement intent switching mid-conversation
- [ ] Add API endpoints for all workflows
- [ ] Frontend integration: chat window handles all message types
- [ ] End-to-end testing of all 3 tools

### Tests Owned

- `test_schedule_meeting_tool.py`
- `test_search_jobs_tool.py`
- `test_query_kb_tool.py`
- `test_otp_service.py`
- `test_email_service.py`
- `test_full_scheduler_flow.py` (integration)
- `test_full_job_search_flow.py` (integration)
- `test_full_qa_flow.py` (integration)
- `test_scheduler_e2e.py`
- `test_chat_flow_e2e.py`

---

## Developer 3 — Ingestion and Infrastructure

### Primary Ownership

| Component | Files |
|-----------|-------|
| Ingestion Tool | `agent/tools/ingest_document.py` |
| Document Loader | `ingestion/document_loader.py` |
| Text Cleaner | `ingestion/text_cleaner.py` |
| Chunker | `ingestion/chunker.py` |
| Metadata Extractor | `ingestion/metadata_extractor.py` |
| Embedding Service | `ingestion/embedding_service.py` |
| Vector Store | `ingestion/vector_store.py` |
| Ingestion Pipeline | `ingestion/pipeline.py` |
| Shared LLM Client | `services/llm/openai_client.py` |
| Shared Vector DB Client | `services/vector_db/` |
| Data Models | `models/` (all files) |
| Pydantic Schemas | `schemas/` (all files) |
| Docker Infrastructure | `infrastructure/` (all files) |
| CI/CD | `.github/workflows/` |
| API - Documents | `api/v2/endpoints/documents.py` |
| API - Health | `api/v2/endpoints/health.py` |

### Tasks (Phased)

#### Phase 1 — Infrastructure Setup 
- [ ] Create `docker-compose.yml` (FastAPI + Qdrant + Redis)
- [ ] Create Dockerfiles for frontend and backend
- [ ] Set up GitHub repository with branch protection
- [ ] Create PR templates and issue templates
- [ ] Set up GitHub Actions CI (lint + test)
- [ ] Define all Pydantic schemas and data models
- [ ] Set up environment variable management

#### Phase 2 — Ingestion Pipeline 
- [ ] Implement multi-format document loader (PDF, DOCX, TXT, MD)
- [ ] Build text cleaner (remove boilerplate, normalize)
- [ ] Implement configurable text chunker (fixed-size, overlap-based)
- [ ] Build metadata extractor (title, author, date, source)
- [ ] Create OpenAI embedding service with batch processing
- [ ] Implement Qdrant vector store wrapper (upsert, search, delete)
- [ ] Build full ingestion pipeline (loader > cleaner > chunker > embed > store)
- [ ] Create ingest_document tool wrapping the pipeline
- [ ] Add async ingestion with status tracking

#### Phase 3 — Shared Services 
- [ ] Build shared OpenAI client wrapper (used by agent and tools)
- [ ] Implement connection pooling for Qdrant
- [ ] Create document management endpoints (upload, list, delete, status)
- [ ] Create health check endpoint
- [ ] Build embedding caching layer
- [ ] Handle rate limits with exponential backoff

#### Phase 4 — DevOps and Deployment 
- [ ] Set up production Docker Compose
- [ ] Configure Nginx reverse proxy
- [ ] Set up Redis for caching/sessions
- [ ] Configure Azure VM deployment scripts
- [ ] Set up Prometheus + Grafana monitoring dashboards
- [ ] Create backup and restore scripts
- [ ] Create automated deployment pipeline

### Tests Owned

- `test_ingest_document_tool.py`
- `test_vector_db.py` (integration)
- `test_api_endpoints.py`

---

## Shared Responsibilities (All 3 Developers)

| Activity | Cadence |
|----------|---------|
| Code reviews (each PR needs 1 approval) | Every PR |
| Daily standup (15 min sync) | Daily |
| Integration testing | End of each phase |
| Documentation updates | Ongoing |
| Sprint planning | Weekly |
| Bug triage | As needed |

---

## Development Timeline

```
WEEK 1:  Foundation
           +-- Dev 1: FastAPI setup, AgentState, LangGraph nodes, confidence gate, logging
           +-- Dev 2: RAG pipeline, query_kb tool
           +-- Dev 3: Docker setup, GitHub, ingestion pipeline (start)

WEEK 1.5:  Core Features
           +-- Dev 1: Intent node, tool selection, graph edges, memory
           +-- Dev 2: schedule_meeting tool (OTP + form), search_jobs tool
           +-- Dev 3: Ingestion pipeline (complete), shared services

WEEK 2:  Resilience and Production
           +-- Dev 1: Circuit breakers, retries, monitoring
           +-- Dev 2: Frontend components, tool integration, E2E tests
           +-- Dev 3: Deployment, CI/CD, performance optimization
```

---

## Git Branching Strategy

```
main <------------- Production-ready code
  |
  +-- dev <--------- Integration branch
       |
       +-- feature/agent-core               (Dev 1)
       +-- feature/graph-nodes              (Dev 1)
       +-- feature/confidence-gate          (Dev 1)
       +-- feature/resilience               (Dev 1)
       |
       +-- feature/qa-tool                  (Dev 2)
       +-- feature/scheduler-tool           (Dev 2)
       +-- feature/job-search-tool          (Dev 2)
       +-- feature/frontend-components      (Dev 2)
       |
       +-- feature/infra-docker             (Dev 3)
       +-- feature/ingestion-pipeline       (Dev 3)
       +-- feature/shared-services          (Dev 3)
       +-- feature/ci-cd                    (Dev 3)
```

### Rules
1. Never push directly to `main` or `dev`.
2. Feature branches branch from `dev`.
3. Every PR must have 1 approval before merge.
4. Squash merge into `dev` for clean history.
5. Merge commit from `dev` to `main` (preserve merge points).
6. Delete branches after merge.

---

## Inter-Developer Dependencies

```
Dev 3 (Infrastructure) ------> Dev 1 (Agent Core) ------> Dev 2 (Tools)
   Docker, Schemas,               AgentState,                Uses agent state,
   LLM Client, Qdrant             Graph Builder,             registers tools
   client are needed               Confidence Gate,           with the graph
   by both Dev 1 & 2              Node interfaces
                                   needed by Dev 2
```