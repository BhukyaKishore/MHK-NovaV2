# Multi-Agent Chatbot V2 — Flowcharts

## 1. Master Orchestration Flow

```mermaid
flowchart TD
    A["User Message"] --> B["LangGraph Agent"]
    B --> E["Intent Detection Node"]
    E --> CONF{"Confidence > 85%?"}

    CONF -->|No| CL["Clarification Node"]
    CL -->|User responds| E

    CONF -->|Yes| F["Tool Selection Node"]

    F -->|meeting_scheduler| G["schedule_meeting tool"]
    F -->|job_search| H["search_jobs tool"]
    F -->|general_qa| I["query_kb tool"]

    G --> K["Response Node"]
    H --> K
    I --> K

    K --> L["Update Memory and State"]
    L --> M["Send Response to User"]
```

---

## 2. Meeting Scheduler Sub-Flow

```mermaid
flowchart TD
    A["schedule_meeting tool invoked"] --> B{"Workflow Stage?"}

    B -->|null| C["Ask for Email"]
    C --> D["User provides email"]
    D --> E{"Valid email?"}
    E -->|No| C
    E -->|Yes| F["send_otp tool"]
    F --> G{"OTP sent?"}
    G -->|Fail| H["Retry 3x with backoff"]
    H -->|All fail| I["FAIL: Unable to send OTP"]
    G -->|Success| J["Set stage = otp_sent"]

    B -->|otp_sent| K["User enters OTP"]
    K --> L{"OTP correct?"}
    L -->|Wrong| M{"Attempts < 3?"}
    M -->|Yes| N["Invalid OTP. N attempts remaining."]
    N --> K
    M -->|No| O["LOCKED: 15-minute cooldown"]
    L -->|Expired| P["Resend new OTP"]
    P --> K
    L -->|Correct| Q["Email Verified"]

    B -->|otp_verified| Q
    Q --> R["Show Meeting Scheduler Form in Chat"]
    R --> S["Fields: Name, Phone, Email (verified), Subject, Message"]
    S --> T["User submits form"]
    T --> U{"All fields valid?"}
    U -->|No| V["Show validation errors"]
    V --> S
    U -->|Yes| W["send_email tool (to HR)"]
    W --> X{"Email sent?"}
    X -->|Fail| Y["Retry 3x"]
    Y -->|All fail| Z["Queue for later + notify user"]
    X -->|Success| AA["Meeting request sent. Our team will get back to you."]
```

---

## 3. Job Search Sub-Flow

```mermaid
flowchart TD
    A["search_jobs tool invoked"] --> B{"Job cache exists?"}

    B -->|Yes| C{"Cache age < 1 hour?"}
    C -->|Yes| D["Use cached JSON"]
    C -->|No| E["Fetch from API/DB"]
    B -->|No| E

    E --> F{"API Success?"}
    F -->|Yes| G["Update cache with timestamp"]
    F -->|No| H{"Stale cache available?"}
    H -->|Yes| I["Use stale data + disclaimer"]
    H -->|No| J["FAIL: Unable to fetch job data"]

    D --> K["Parse user query"]
    G --> K
    I --> K

    K --> L{"Filter type?"}
    L -->|Role| M["Filter by role name"]
    L -->|Location| N["Filter by location/remote"]
    L -->|Salary| O["Filter by salary range"]
    L -->|Experience| P["Filter by level"]
    L -->|General| Q["Return all openings"]

    M --> R{"Matches found?"}
    N --> R
    O --> R
    P --> R
    Q --> R

    R -->|Yes| S["Format and display job listings"]
    R -->|No| T["No matching positions at this time."]
```

---

## 4. General Q&A Sub-Flow

```mermaid
flowchart TD
    A["query_kb tool invoked"] --> B["Load Conversation History"]
    B --> C["Check if follow-up question"]

    C -->|Yes| D["Build context from history"]
    C -->|No| E["Use query as-is"]

    D --> F["Embed query via OpenAI"]
    E --> F

    F --> G["Semantic Search in Qdrant"]
    G --> H{"Relevant chunks found?"}

    H -->|No| I{"Offer alternative?"}
    I -->|Yes| J["I do not have that information. Would you like to schedule a meeting?"]
    I -->|No| K["I do not have that information in my knowledge base."]

    H -->|Yes| L["Build prompt: system + context + history + query"]
    L --> M["Generate via GPT-4"]
    M --> N["Append to conversation buffer"]
    N --> O["Return answer with sources"]
```

---

## 5. LangGraph State Machine

```mermaid
stateDiagram-v2
    [*] --> SessionNode
    SessionNode --> IntentDetectionNode : Session loaded

    IntentDetectionNode --> ConfidenceGate : Intent classified
    ConfidenceGate --> ClarificationNode : confidence < 0.85
    ConfidenceGate --> ToolSelectionNode : confidence >= 0.85

    ClarificationNode --> IntentDetectionNode : User clarifies

    ToolSelectionNode --> ScheduleMeetingTool : meeting_scheduler
    ToolSelectionNode --> SearchJobsTool : job_search
    ToolSelectionNode --> QueryKBTool : general_qa

    ScheduleMeetingTool --> OTPSubFlow : Need verification
    OTPSubFlow --> FormSubFlow : OTP verified
    FormSubFlow --> EmailSubFlow : Form valid
    EmailSubFlow --> ResponseNode : Email sent
    OTPSubFlow --> ScheduleMeetingTool : OTP failed (retry)

    SearchJobsTool --> CacheCheck : Check cache
    CacheCheck --> JobAPIFetch : Cache miss/stale
    JobAPIFetch --> JobFilter : Data fetched
    CacheCheck --> JobFilter : Cache hit
    JobFilter --> ResponseNode : Answer ready

    QueryKBTool --> RAGRetrieval : Query embedded
    RAGRetrieval --> ResponseGeneration : Chunks found
    ResponseGeneration --> ResponseNode : Answer generated

    ResponseNode --> ReEvaluationNode : Need quality check
    ReEvaluationNode --> IntentDetectionNode : Low quality, retry
    ReEvaluationNode --> [*] : Quality OK, done
    ResponseNode --> [*] : Direct response
```

---


## 6. Ingestion Pipeline

```mermaid
flowchart LR
    A["Document Upload API"] --> B["ingest_document tool"]
    B --> C["Format Detection"]
    C -->|Unsupported| D["Reject"]
    C -->|Supported| E["Document Loader"]
    E -->|Corrupted| F["Dead Letter Queue"]
    E -->|OK| G["Text Cleaner"]
    G -->|Empty| H["Skip with warning"]
    G -->|OK| I["Text Chunker"]
    I --> J["Batch Embedding"]
    J -->|Rate Limited| K["Exponential Backoff"]
    K --> J
    J -->|OK| L["Qdrant Upsert"]
    L -->|Connection Fail| M["Circuit Breaker"]
    M --> L
    L -->|OK| N["Success"]
```

---

## 7. Session Lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant API as FastAPI
    participant AG as LangGraph Agent
    participant VDB as Qdrant
    participant LLM as OpenAI

    U->>FE: Types message
    FE->>API: POST /api/v2/chat
    API->>AG: Process message
    AG->>AG: Session Node (load/create)
    AG->>AG: Intent Detection Node
    AG->>AG: Confidence Gate (>= 0.85?)

    alt Confidence < 0.85
        AG-->>API: Clarification question
        API-->>FE: Clarification response
        FE-->>U: Display clarification request
    else General Q&A
        AG->>AG: Tool Selection Node
        AG->>LLM: query_kb tool: embed query
        LLM-->>AG: Query embedding
        AG->>VDB: Semantic search
        VDB-->>AG: Top-K chunks
        AG->>LLM: Generate answer
        LLM-->>AG: Response text
    else Meeting Scheduler
        AG->>AG: Tool Selection Node
        AG->>AG: schedule_meeting tool: ask for email
    else Job Search
        AG->>AG: Tool Selection Node
        AG->>AG: search_jobs tool: fetch/cache jobs
    end

    AG->>AG: Update state and memory
    AG-->>API: Response payload
    API-->>FE: JSON response
    FE-->>U: Display message
```
