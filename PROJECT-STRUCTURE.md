# Multi-Agent Chatbot V2 — Project Structure

## Root Directory

```
ai-chatbot-v2/
|
+-- frontend/                              # React/Next.js frontend
+-- backend/                               # FastAPI backend (Single Agent)
+-- infrastructure/                        # Docker, deployment configs
+-- docs/                                  # Documentation
+-- scripts/                               # Utility scripts
+-- data/                                  # Local development data
+-- tests/                                 # Comprehensive test suite
+-- .github/                               # CI/CD workflows
|
+-- docker-compose.yml                     # Main compose file
+-- docker-compose.dev.yml                 # Dev overrides
+-- docker-compose.prod.yml                # Prod overrides
+-- Makefile                               # Common commands
+-- .env.example                           # Environment template
+-- .gitignore
+-- README.md
+-- CONTRIBUTING.md
+-- CHANGELOG.md
+-- LICENSE
```

---

## Backend Structure (backend/)

```
backend/
|
+-- app/
|   +-- main.py                            # FastAPI entry point
|   |
|   +-- config/
|   |   +-- __init__.py
|   |   +-- settings.py                    # Pydantic BaseSettings
|   |   +-- constants.py                   # App constants
|   |   +-- env_loader.py                  # Environment helper
|   |
|   +-- api/
|   |   +-- __init__.py
|   |   +-- v2/
|   |   |   +-- __init__.py
|   |   |   +-- endpoints/
|   |   |   |   +-- __init__.py
|   |   |   |   +-- chat.py               # POST /api/v2/chat
|   |   |   |   +-- documents.py          # POST /api/v2/documents/upload
|   |   |   |   +-- jobs.py               # GET  /api/v2/jobs
|   |   |   |   +-- scheduler.py          # POST /api/v2/scheduler/*
|   |   |   |   +-- otp.py                # POST /api/v2/otp/send, /verify
|   |   |   |   +-- health.py             # GET  /api/v2/health
|   |   |   |   +-- admin.py              # Admin operations
|   |   |   +-- router.py                 # API router aggregator
|   |   +-- dependencies.py               # Dependency injection
|   |
|   +-- agent/                             # Single LangGraph Agent
|   |   +-- __init__.py
|   |   +-- agent.py                       # Main agent class
|   |   +-- graph_builder.py               # LangGraph graph definition
|   |   +-- state_schema.py               # AgentState TypedDict
|   |   |
|   |   +-- nodes/                         # Graph nodes
|   |   |   +-- __init__.py
|   |   |   +-- session_node.py            # Session init / load
|   |   |   +-- intent_node.py             # Intent classification via LLM
|   |   |   +-- confidence_gate.py         # Confidence > 0.85 check
|   |   |   +-- tool_selection_node.py     # Pick tool based on intent
|   |   |   +-- re_evaluation_node.py      # Quality check
|   |   |   +-- response_node.py           # Final response formatting
|   |   |   +-- clarification_node.py      # Ask user to clarify
|   |   |
|   |   +-- tools/                         # Tool implementations
|   |   |   +-- __init__.py
|   |   |   +-- schedule_meeting.py        # Meeting scheduler flow
|   |   |   +-- search_jobs.py             # Job search + cache
|   |   |   +-- query_knowledge_base.py    # RAG pipeline Q&A
|   |   |   +-- ingest_document.py         # Document processing pipeline
|   |   |   +-- send_otp.py                # OTP generation & sending
|   |   |   +-- verify_otp.py              # OTP verification
|   |   |   +-- send_email.py              # Email to HR
|   |   |
|   |   +-- services/                      # Shared services used by tools
|   |       +-- __init__.py
|   |       +-- otp_service.py             # OTP generation & verification
|   |       +-- email_service.py           # SMTP / SendGrid
|   |       +-- job_cache_service.py       # Job JSON cache manager
|   |       +-- form_validator.py          # Scheduler form validation
|   |       +-- intent_detector.py         # Intent classification logic
|   |
|   +-- rag/                               # RAG Pipeline
|   |   +-- __init__.py
|   |   +-- retriever.py                   # Semantic search in Qdrant
|   |   +-- context_builder.py             # Context formatting
|   |   +-- response_generator.py          # LLM response generation
|   |
|   +-- ingestion/                         # Document Ingestion
|   |   +-- __init__.py
|   |   +-- document_loader.py             # Multi-format loader
|   |   +-- text_cleaner.py               # Text normalization
|   |   +-- chunker.py                     # Configurable chunking
|   |   +-- metadata_extractor.py          # Document metadata
|   |   +-- embedding_service.py           # OpenAI embeddings
|   |   +-- vector_store.py               # Qdrant operations
|   |   +-- pipeline.py                    # Full ingestion pipeline
|   |
|   +-- memory/
|   |   +-- __init__.py
|   |   +-- conversation_buffer.py         # ConversationBufferMemory
|   |   +-- session_store.py              # Session persistence (Redis/memory)
|   |   +-- history_manager.py            # Chat history utilities
|   |
|   +-- models/
|   |   +-- __init__.py
|   |   +-- chat.py                        # ChatRequest, ChatResponse
|   |   +-- document.py                    # Document models
|   |   +-- scheduler.py                   # SchedulerRequest, OTPRequest
|   |   +-- job.py                         # JobListing, JobSearchResult
|   |   +-- state.py                       # Agent state models
|   |
|   +-- schemas/
|   |   +-- __init__.py
|   |   +-- chat.py
|   |   +-- document.py
|   |   +-- scheduler.py
|   |   +-- otp.py
|   |   +-- common.py
|   |
|   +-- middleware/
|   |   +-- __init__.py
|   |   +-- cors.py
|   |   +-- rate_limit.py
|   |   +-- error_handler.py
|   |   +-- request_id.py
|   |   +-- input_sanitizer.py
|   |
|   +-- services/
|   |   +-- __init__.py
|   |   +-- llm/
|   |   |   +-- __init__.py
|   |   |   +-- openai_client.py           # OpenAI API wrapper
|   |   |   +-- prompt_templates.py        # All prompt templates
|   |   |
|   |   +-- vector_db/
|   |       +-- __init__.py
|   |       +-- qdrant_client.py           # Qdrant wrapper
|   |       +-- operations.py             # CRUD operations
|   |
|   +-- resilience/
|   |   +-- __init__.py
|   |   +-- circuit_breaker.py
|   |   +-- retry_handler.py
|   |   +-- dead_letter_queue.py
|   |   +-- fallback_strategies.py
|   |
|   +-- observability/
|   |   +-- __init__.py
|   |   +-- logger.py
|   |   +-- metrics.py
|   |   +-- tracing.py
|   |
|   +-- utils/
|       +-- __init__.py
|       +-- text_processing.py
|       +-- file_handling.py
|       +-- validators.py
|       +-- helpers.py
|       +-- exceptions.py
|
+-- tests/
|   +-- __init__.py
|   +-- conftest.py
|   +-- unit/
|   |   +-- test_agent.py
|   |   +-- test_intent_node.py
|   |   +-- test_confidence_gate.py
|   |   +-- test_schedule_meeting_tool.py
|   |   +-- test_search_jobs_tool.py
|   |   +-- test_query_kb_tool.py
|   |   +-- test_ingest_document_tool.py
|   |   +-- test_otp_service.py
|   |   +-- test_email_service.py
|   |   +-- test_circuit_breaker.py
|   |   +-- test_memory.py
|   |
|   +-- integration/
|   |   +-- test_full_scheduler_flow.py
|   |   +-- test_full_job_search_flow.py
|   |   +-- test_full_qa_flow.py
|   |   +-- test_api_endpoints.py
|   |   +-- test_vector_db.py
|   |   +-- test_graph_execution.py
|   |
|   +-- e2e/
|       +-- test_chat_flow_e2e.py
|       +-- test_scheduler_e2e.py
|       +-- test_resilience_e2e.py
|
+-- .env
+-- .env.example
+-- .gitignore
+-- requirements.txt
+-- requirements-dev.txt
+-- Dockerfile
+-- pytest.ini
+-- README.md
```

---

## Frontend Structure (frontend/)

```
frontend/
|
+-- public/
|   +-- images/
|   +-- icons/
|   +-- favicon.ico
|
+-- src/
|   +-- components/
|   |   +-- common/
|   |   |   +-- Button/
|   |   |   +-- Card/
|   |   |   +-- Modal/
|   |   |   +-- Input/
|   |   |   +-- Loader/
|   |   |   +-- Toast/
|   |   |
|   |   +-- layout/
|   |   |   +-- Header/
|   |   |   +-- Footer/
|   |   |   +-- Navbar/
|   |   |   +-- Sidebar/
|   |   |
|   |   +-- chatbot/
|   |       +-- ChatWindow/
|   |       +-- ChatMessage/
|   |       +-- ChatInput/
|   |       +-- ChatBubble/
|   |       +-- TypingIndicator/
|   |       +-- SuggestedQuestions/
|   |       +-- OTPVerification/
|   |       +-- SchedulerForm/
|   |       +-- JobListings/
|   |       +-- ErrorMessage/
|   |
|   +-- pages/
|   +-- hooks/
|   +-- services/
|   +-- store/
|   +-- utils/
|   +-- styles/
|   +-- types/
|   +-- config/
|
+-- package.json
+-- tsconfig.json
+-- next.config.js
+-- Dockerfile
+-- README.md
```

---

## Infrastructure (infrastructure/)

```
infrastructure/
|
+-- docker/
|   +-- docker-compose.yml
|   +-- docker-compose.dev.yml
|   +-- docker-compose.prod.yml
|   +-- frontend/
|   +-- backend/
|   +-- qdrant/
|   +-- redis/
|
+-- scripts/
|   +-- deploy.sh
|   +-- backup.sh
|   +-- restore.sh
|   +-- health_check.sh
|
+-- azure/
|   +-- vm-setup.sh
|   +-- firewall-rules.json
|   +-- storage-config.json
|
+-- monitoring/
    +-- prometheus.yml
    +-- grafana/
    |   +-- dashboards/
    |       +-- agent-performance.json
    |       +-- intent-distribution.json
    |       +-- system-health.json
    +-- loki-config.yml
```

---

## Key Design Principles

| Principle | How It Is Applied |
|-----------|-----------------|
| **Single Agent** | One LangGraph agent with specialized tools, no inter-agent overhead |
| **Separation of Concerns** | Each tool handles one workflow, services are shared |
| **Single Responsibility** | Each file/class has one job |
| **Dependency Injection** | FastAPI Depends() for loose coupling |
| **No Circular Imports** | Agent > Tools > Services, never reverse |
| **Testability** | Every tool and node independently testable |
| **Configuration-Driven** | All tuning via env vars / settings.py |
| **Confidence Gate** | Intent routing only proceeds above 85/100 threshold |
