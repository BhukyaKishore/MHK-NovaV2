# Multi-Agent Chatbot V2 — System Architecture

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Agent and Tool Descriptions](#2-agent-and-tool-descriptions)
3. [Detailed Flowcharts](#3-detailed-flowcharts)
4. [State Machine Design](#4-state-machine-design)
5. [Fault Tolerance and Resilience](#5-fault-tolerance-and-resilience)
6. [Scalability Patterns](#6-scalability-patterns)
7. [Security and Compliance](#7-security-and-compliance)

---

## 1. Architecture Overview

### V1 to V2 Evolution

| Aspect | V1 (Basic) | V2 (Single Agent + Tools) |
|--------|------------|---------------------------|
| Agent Model | Single LangGraph agent, no tools | Single LangGraph agent with specialized tools |
| Ingestion | Script-based batch | Ingestion tool with live pipeline |
| Query Handling | Single RAG pipeline | Intent-routed tool selection |
| Meeting Scheduler | Not supported | OTP-verified email + form flow (tool) |
| Job Search | Not supported | JSON-cached + API-backed live search (tool) |
| Memory | Basic buffer | Conversation Buffer Memory + session persistence |
| Fault Tolerance | Basic try/catch | Circuit breakers, retries, dead-letter queues |
| Observability | Logging only | Structured logs + metrics + tracing |

### High-Level System Architecture

```
+------------------------------------------------------------------------------+
|                              FRONTEND (React/Next.js)                        |
|  +----------+ +----------+ +-----------+ +----------+ +---------------+     |
|  |ChatWindow| |OTP Form  | |Scheduler  | |Job       | |Typing         |     |
|  |          | |          | |Form       | |Listings  | |Indicator      |     |
|  +----------+ +----------+ +-----------+ +----------+ +---------------+     |
+-------------------------------------+----------------------------------------+
                                       | WebSocket / REST API
                                       v
+------------------------------------------------------------------------------+
|                           API GATEWAY (FastAPI)                               |
|  +--------------+ +---------------+ +--------------+ +-------------------+   |
|  | Rate Limiter | | Auth Middleware| | CORS Handler | | Request Validator |   |
|  +--------------+ +---------------+ +--------------+ +-------------------+   |
+-------------------------------------+----------------------------------------+
                                       |
                                       v
+------------------------------------------------------------------------------+
|                     LANGGRAPH AGENT (Single Agent)                            |
|  +-----------------------------------------------------------------------+   |
|  |  Session Node > Intent Node > Confidence Gate > Tool Selection Node   |   |
|  +-----------------------------------------------------------------------+   |
|           |            |            |            |            |               |
|           v            v            v            v            v               |
|  +-----------+ +------------+ +----------+ +----------+ +-----------+        |
|  | schedule  | | search     | | query    | | ingest   | | send/     |        |
|  | _meeting  | | _jobs      | | _kb      | | _document| | verify    |        |
|  | tool      | | tool       | | tool     | | tool     | | _otp tool |        |
|  +-----------+ +------------+ +----------+ +----------+ +-----------+        |
+------------------------------------------------------------------------------+
                                       |
              +------------------------+------------------------+
              v                        v                        v
     +------------+           +------------+           +------------+
     |   Qdrant   |           |   OpenAI   |           |  Email/SMTP|
     |  Vector DB |           |   GPT-4    |           |   Service  |
     +------------+           +------------+           +------------+
```

---

## 2. Agent and Tool Descriptions

### 2.1 LangGraph Agent

**Purpose**: The single entry point for all user interactions. Uses LangGraph to manage conversation state, detect intent, enforce the confidence gate, select the appropriate tool, and return responses.

**Responsibilities**:
- Create and manage conversation sessions
- Detect intent from user messages via LLM
- Enforce the confidence gate (> 0.85) before tool selection
- Select and execute the correct tool based on detected intent
- Track multi-step workflows (e.g., meeting scheduler: OTP > Form > Email)
- Handle tool failures with fallback strategies
- Manage conversation state and memory across turns

**State Schema**:
```python
class AgentState(TypedDict):
    session_id: str
    user_id: Optional[str]
    messages: List[BaseMessage]            # Full conversation history
    current_intent: Optional[str]          # "meeting_scheduler" | "job_search" | "general_qa"
    workflow_stage: Optional[str]          # "otp_pending" | "form_pending" | "completed"
    verified_email: Optional[str]          # After OTP verification
    collected_form_data: Optional[dict]    # Meeting scheduler form data
    job_cache: Optional[dict]             # Cached job data
    retry_count: int                       # For fault tolerance
    error_log: List[str]                   # Error tracking
    confidence_score: float               # Intent detection confidence (must be > 0.85)
    loop_count: int                        # Prevent infinite loops
    tool_trace: List[dict]                # Observability trace
    selected_tool: Optional[str]          # Currently selected tool
    tool_result: Optional[dict]           # Result from last tool execution
```

**Confidence Gate**: After intent classification, the agent only proceeds to tool selection if the confidence score exceeds **85/100 (0.85)**. If confidence is below this threshold, the agent asks the user a clarification question to refine the intent before selecting any tool.

---

### 2.2 Tool Catalog

The agent has access to the following tools. Each tool is a self-contained function that handles one specific workflow.

#### Tool: `schedule_meeting`

A multi-step, stateful tool for meeting scheduling:

```
Step 1: Detect "meeting_scheduler" intent (confidence > 0.85)
    |
    v
Step 2: Request user's email
    |
    v
Step 3: Send OTP to email (via SMTP/SendGrid)
    |
    v
Step 4: Verify OTP entered by user
    |
    +-- [FAIL] Invalid OTP > Allow 3 retries > Lock after 3 failures
    |
    v  [PASS] OTP Verified
Step 5: Show meeting scheduler form in chat UI
         Fields: Name, Contact No, Email (pre-filled), Subject, Message
    |
    v
Step 6: Validate form data
    |
    +-- [FAIL] Invalid > Show errors, ask to fix
    |
    v  [PASS] Valid
Step 7: Send email to HR (company-hr@company.com)
    |
    v
Step 8: Confirm to user: "Meeting request sent. Our team will get back to you."
```

#### Tool: `search_jobs`

```
Step 1: Detect "job_search" intent (confidence > 0.85)
    |
    v
Step 2: Check if job_cache exists and is fresh (< 1 hour old)
    |
    +-- [HIT] Cache hit > Use cached JSON
    |
    v  [MISS] Cache miss
Step 3: Fetch from API / Database
    |
    +-- [FAIL] API fails > Use stale cache if available > Else error msg
    |
    v  [PASS] Success
Step 4: Update job_cache with timestamp
    |
    v
Step 5: Parse user's specific question against job data
         (e.g., "any React jobs?", "remote positions?", "salary range?")
    |
    v
Step 6: Generate contextual answer from job data using LLM
    |
    v
Step 7: Return formatted job listings + answer
```

#### Tool: `query_knowledge_base`

```
Step 1: Detect "general_qa" intent (confidence > 0.85)
    |
    v
Step 2: Load conversation history from Conversation Buffer Memory
    |
    v
Step 3: Build context from chat history (for follow-ups)
    |
    v
Step 4: Embed query > Semantic search in Qdrant
    |
    v
Step 5: Retrieve top-K relevant document chunks
    |
    +-- No results > "I don't have information about that. Would you like to schedule a meeting?"
    |
    v
Step 6: Build prompt with context + chat history
    |
    v
Step 7: Generate response via GPT-4
    |
    v
Step 8: Append to conversation memory > Return response
```

#### Tool: `ingest_document`

Handles all document processing — loading, cleaning, chunking, embedding, and storing into Qdrant.

**Supported formats**: PDF, DOCX, TXT, MD, CSV

**Processing Pipeline**:
```
Document Upload
    |
    v
+----------------+
| Format Detector | --- Unsupported? > Reject with error
+--------+-------+
         v
+----------------+
| Document Loader | --- Corrupted? > Dead-letter queue
+--------+-------+
         v
+----------------+
|  Text Cleaner  | --- Empty after clean? > Skip with warning
+--------+-------+
         v
+----------------+
|  Text Chunker  | --- Configurable: chunk_size=1000, overlap=200
+--------+-------+
         v
+----------------+
| Embedding Gen  | --- Rate limited? > Exponential backoff
+--------+-------+
         v
+----------------+
| Qdrant Upsert  | --- Connection fail? > Retry with circuit breaker
+--------+-------+
         v
    Success Response
```

#### Tool: `send_otp` / `verify_otp`

- Generates 6-digit cryptographically random OTP
- Sends via SMTP/SendGrid
- Hashes before storage
- Enforces 5-minute expiry, 3 attempt limit, 15-minute cooldown

#### Tool: `send_email`

- Sends meeting scheduler form data to HR
- Retry 3x with exponential backoff on failure
- Queues for later delivery if all retries fail

---

## 3. Detailed Flowcharts

### 3.1 Master Orchestration Flow

```
                    +------------------+
                    |  User sends      |
                    |   message        |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    |  LangGraph       |
                    |    Agent         |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    |  Intent          |
                    |  Detection       |
                    |  Node (LLM)      |
                    +--------+---------+
                             |
                    +--------+--------+
                    | Confidence       |
                    | Score > 0.85?    |
                    +--------+---------+
                             |
                    +--------+--------+
                    | NO               | YES
                    v                  v
              +-----------+   +-------+--------+
              | Ask user  |   | Tool Selection |
              | to clarify|   | Node           |
              |           |   |                |
              +-----+-----+   +-------+--------+
                    |                  |
                    v                  |
              Intent Detection  +------+-----+------+
              Node (retry)      |            |      |
                                v            v      v
                          +----------+ +----------+ +----------+
                          | schedule | | search   | | query    |
                          | _meeting | | _jobs    | | _kb      |
                          | tool     | | tool     | | tool     |
                          +----+-----+ +----+-----+ +----+-----+
                               |            |            |
                               v            v            v
                          +----------+ +----------+ +----------+
                          | OTP Flow | | Cache/API| |   RAG    |
                          | > Form   | | > Answer | | Pipeline |
                          | > Email  | |          | | + Memory |
                          +----+-----+ +----+-----+ +----+-----+
                               |            |            |
                               +------------+------------+
                                            |
                                            v
                                     +------------------+
                                     |  Response        |
                                     |  Node            |
                                     +--------+---------+
                                              |
                                              v
                                     +------------------+
                                     |  Update          |
                                     |  Memory &        |
                                     |  State           |
                                     +--------+---------+
                                              |
                                              v
                                     +------------------+
                                     |  Send to         |
                                     |  User            |
                                     +------------------+
```

### 3.2 LangGraph State Machine

```
                +------------------+
                |   START (Entry)  |
                +--------+---------+
                         |
                         v
                +------------------+
                |  Session Node     | ---- New? > Create session
                |  (Load State)     | ---- Existing? > Load state
                +--------+---------+
                         |
                         v
                +------------------+
                |  Intent Detection |
                |  Node (LLM)       |
                +--------+---------+
                         |
                +--------+---------+
                | Confidence > 0.85?|
                +--------+---------+
                         |
            +------------+----------- NO --> Clarification Node
            | YES                              |
            |                                  v
            |                           Ask user to clarify
            |                                  |
            |                                  v
            |                           Intent Detection Node (retry)
            |
            +----> Tool Selection Node
                         |
            +------------+----------------+
            |            |                |
            v            v                v
    +------------+ +------------+  +------------+
    | schedule   | | search     |  | query      |
    | _meeting   | | _jobs      |  | _kb        |
    | tool       | | tool       |  | tool       |
    +-----+------+ +-----+------+  +------+-----+
          |             |               |
          v             v               v
    +------------+ +------------+  +------------+
    | Stateful   | | Cache >    |  |  RAG       |
    | Sub-Flow   | | API >      |  |  Pipeline  |
    | (OTP>Form  | | Filter >   |  |  + Memory  |
    |  >Email)   | | Answer     |  |            |
    +-----+------+ +-----+------+  +------+-----+
          |             |               |
          +-------------+---------------+
                        |
                        v
                +------------------+
                |  Re-evaluation    | ---- Quality insufficient?
                |     Node          |      > Retry (max 3)
                +--------+---------+
                         |
                         v
                +------------------+
                |  Response Node    |
                |  (Format & Send)  |
                +--------+---------+
                         |
                         v
                +------------------+
                |      END          |
                +------------------+
```

### 3.3 Meeting Scheduler Sub-Flow (Detailed)

```
          +-----------------------+
          | scheduler_entry_node  |
          +----------+------------+
                     |
                     v
          +-----------------------+
          | Check workflow_stage   |
          +----------+------------+
                     |
        +------------+------------------------+
        |            |                        |
        v            v                        v
  stage=null    stage=otp_sent          stage=form_pending
        |            |                        |
        v            v                        v
  +----------+  +----------+          +---------------+
  | Ask Email|  |Verify OTP|          | Validate Form |
  +----+-----+  +----+-----+          +-------+-------+
       |              |                        |
       v              +-- [FAIL]               +-- [FAIL] invalid
  +----------+        |   retry<3?             |
  | send_otp |        |   > ask again          v
  | tool     |        |   retry>=3?      +---------------+
  +----+-----+        |   > abort        |Return errors  |
       |              |                  |ask to fix     |
       v              v [PASS]           +---------------+
  Set stage=      Set stage=                   |
  "otp_sent"      "form_pending"               v
       |              |              +-----------------+
       v              v              |  [PASS] Valid   |
  +----------+  +---------------+   +--------+--------+
  | Ask user |  | Show form in  |            |
  | for OTP  |  | chat UI       |            v
  +----------+  | (pre-fill     |   +-----------------+
                |  email)        |   | send_email      |
                +---------------+   | tool (to HR)    |
                                    +--------+--------+
                                             |
                                    +--------+--------+
                                    |  [FAIL] Email?  |
                                    |  > Retry 3x     |
                                    |  > Queue for    |
                                    |    later send   |
                                    +--------+--------+
                                             | [PASS]
                                             v
                                    +-----------------+
                                    | "Meeting request|
                                    |  sent. Our team |
                                    |  will get back  |
                                    |  to you."       |
                                    +-----------------+
```

---

## 4. State Machine Design

### 4.1 Conversation States

```python
class ConversationStage(Enum):
    IDLE = "idle"                          # No active workflow
    INTENT_DETECTION = "intent_detection"  # Analyzing user message
    CLARIFICATION = "clarification"        # Confidence < 0.85, asking user to clarify
    TOOL_SELECTION = "tool_selection"      # Selecting appropriate tool

    # Meeting Scheduler states
    SCHEDULER_EMAIL_REQUESTED = "scheduler_email_requested"
    SCHEDULER_OTP_SENT = "scheduler_otp_sent"
    SCHEDULER_OTP_VERIFIED = "scheduler_otp_verified"
    SCHEDULER_FORM_DISPLAYED = "scheduler_form_displayed"
    SCHEDULER_FORM_SUBMITTED = "scheduler_form_submitted"
    SCHEDULER_EMAIL_SENT = "scheduler_email_sent"
    SCHEDULER_COMPLETED = "scheduler_completed"
    SCHEDULER_FAILED = "scheduler_failed"

    # Job Search states
    JOB_CACHE_CHECK = "job_cache_check"
    JOB_API_FETCH = "job_api_fetch"
    JOB_ANSWERING = "job_answering"
    JOB_COMPLETED = "job_completed"

    # General Q&A states
    QA_RETRIEVING = "qa_retrieving"
    QA_GENERATING = "qa_generating"
    QA_COMPLETED = "qa_completed"

    # Error states
    ERROR_RECOVERABLE = "error_recoverable"
    ERROR_FATAL = "error_fatal"
```

### 4.2 State Transitions

```
IDLE --> INTENT_DETECTION --> confidence >= 0.85? --> TOOL_SELECTION --> [schedule_meeting | search_jobs | query_kb]
                          --> confidence < 0.85?  --> CLARIFICATION --> INTENT_DETECTION (retry)

schedule_meeting tool:
  EMAIL_REQUESTED > OTP_SENT > OTP_VERIFIED > FORM_DISPLAYED
  > FORM_SUBMITTED > EMAIL_SENT > COMPLETED

search_jobs tool:
  CACHE_CHECK > (API_FETCH?) > ANSWERING > COMPLETED

query_kb tool:
  RETRIEVING > GENERATING > COMPLETED

Any state > ERROR_RECOVERABLE > (retry) > previous state
Any state > ERROR_FATAL > IDLE (with error message)
```

---

## 5. Fault Tolerance and Resilience

### 5.1 Circuit Breaker Pattern

```python
class CircuitBreaker:
    """
    Prevents cascading failures by stopping calls to a failing service.

    States:
    - CLOSED:    Normal operation, requests pass through
    - OPEN:      Service failing, requests are rejected immediately
    - HALF_OPEN: Testing if service recovered
    """
    FAILURE_THRESHOLD = 5          # Open circuit after 5 failures
    RECOVERY_TIMEOUT = 30          # Try again after 30 seconds
    SUCCESS_THRESHOLD = 3          # Close circuit after 3 successes
```

**Applied to all tools calling external services**:
- OpenAI API calls (embedding + completion)
- Qdrant vector DB operations
- SMTP/Email service
- External job API

### 5.2 Retry Strategy

| Service | Max Retries | Backoff | Timeout |
|---------|-------------|---------|---------|
| OpenAI Embedding | 3 | Exponential (1s, 2s, 4s) | 30s |
| OpenAI Completion | 3 | Exponential (2s, 4s, 8s) | 60s |
| Qdrant Operations | 3 | Linear (1s, 2s, 3s) | 15s |
| Email (SMTP) | 3 | Exponential (5s, 10s, 20s) | 30s |
| Job API | 2 | Fixed (3s) | 10s |

### 5.3 Dead Letter Queue (DLQ)

Failed tool operations that exhaust all retries are pushed to a DLQ:

```python
class DeadLetterQueue:
    """
    Stores failed operations for later processing or manual review.

    Stored in: Redis or in-memory deque (configurable)
    Max size: 1000 entries
    TTL: 24 hours
    """
    async def push(self, operation: str, payload: dict, error: str):
        ...

    async def retry_all(self):
        ...

    async def get_failed(self, limit: int = 50):
        ...
```

### 5.4 Graceful Degradation

| Tool Failure | Degradation Strategy |
|---------|---------------------|
| OpenAI API down | Return cached response or "Service temporarily unavailable" |
| Qdrant unavailable | Attempt fallback to keyword search if configured |
| Email service down | Queue email, notify user "Request saved, email pending" |
| Job API timeout | Use stale cache if available |
| OTP service fail | Allow manual verification path |
| Memory store fail | Continue without history (warn: "Context may be limited") |

### 5.5 Health Monitoring

```python
class HealthChecker:
    """
    Periodic health checks for all dependencies used by tools.
    Endpoint: GET /api/v2/health
    """
    checks = [
        "openai_api",
        "qdrant_connection",
        "email_service",
        "job_api",
        "redis_connection",   # If using Redis for sessions
        "memory_usage",
        "cpu_usage",
    ]
```

---

## 6. Scalability Patterns

### 6.1 Horizontal Scaling

```
                    +--------------+
                    |  Nginx/LB    |
                    |  (Reverse    |
                    |   Proxy)     |
                    +------+-------+
                           |
              +------------+------------+
              v            v            v
        +----------+ +----------+ +----------+
        | FastAPI  | | FastAPI  | | FastAPI  |
        | Instance | | Instance | | Instance |
        |    #1    | |    #2    | |    #3    |
        +----------+ +----------+ +----------+
              |            |            |
              +------------+------------+
                           |
                           v
                    +--------------+
                    |    Qdrant    |
                    |   Cluster   |
                    +--------------+
```

### 6.2 Caching Strategy

| Cache Layer | Technology | TTL | Purpose |
|-------------|-----------|-----|---------|
| Query Embedding Cache | In-memory (LRU) | 1 hour | Avoid re-embedding identical queries |
| Job Data Cache | Redis / In-memory | 1 hour | Reduce external API calls |
| Session State | Redis | 24 hours | Persist conversation across restarts |
| Document Chunk Cache | Redis | 6 hours | Fast retrieval of hot documents |

### 6.3 Async Processing

```python
# Ingestion runs as background tasks to not block the API
@router.post("/documents/upload")
async def upload_document(file: UploadFile, background_tasks: BackgroundTasks):
    task_id = str(uuid4())
    background_tasks.add_task(
        ingest_document_tool,
        file=file,
        task_id=task_id
    )
    return {"task_id": task_id, "status": "processing"}

@router.get("/documents/status/{task_id}")
async def get_ingestion_status(task_id: str):
    return await get_tool_status(task_id)
```

---

## 7. Security and Compliance

### 7.1 Input Sanitization

```python
class InputSanitizer:
    """
    Sanitize all user inputs before the agent processes them.
    - Strip HTML/JS injection attempts
    - Block prompt injection patterns
    - Validate email formats
    - Limit message length (max 2000 chars)
    """
    BLOCKED_PATTERNS = [
        r"ignore previous instructions",
        r"you are now",
        r"act as",
        r"forget everything",
    ]
```

### 7.2 OTP Security

- OTP expires after **5 minutes**
- Max **3 attempts** per OTP
- After 3 failed attempts, **15-minute cooldown**
- OTP is **6 digits**, cryptographically random
- Rate limit: **3 OTP requests per email per hour**
- OTPs are **hashed** before storage (never stored in plaintext)

### 7.3 Data Privacy

- No PII stored in vector database
- Conversation logs auto-purge after **30 days** (configurable)
- Email addresses used for meeting scheduling are **not** retained after email is sent
- All API keys stored in environment variables / secrets manager
