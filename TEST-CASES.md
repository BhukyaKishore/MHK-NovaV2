# Multi-Agent Chatbot V2 — Test Cases

## Table of Contents

1. [Agent and Graph Tests](#1-agent-and-graph-tests)
2. [Ingestion Tool Tests](#2-ingestion-tool-tests)
3. [schedule_meeting Tool Tests](#3-schedule_meeting-tool-tests)
4. [search_jobs Tool Tests](#4-search_jobs-tool-tests)
5. [query_kb Tool Tests](#5-query_kb-tool-tests)
6. [Intent Detection and Confidence Gate Tests](#6-intent-detection-and-confidence-gate-tests)
7. [Conversation Memory Tests](#7-conversation-memory-tests)
8. [Fault Tolerance Tests](#8-fault-tolerance-tests)
9. [Security Tests](#9-security-tests)
10. [Performance Tests](#10-performance-tests)
11. [End-to-End Tests](#11-end-to-end-tests)
12. [Edge Case Tests](#12-edge-case-tests)

---

## 1. Agent and Graph Tests

| # | Test Case | Input | Precondition | Expected Outcome | Priority |
|---|-----------|-------|--------------|------------------|----------|
| A-01 | New session creation | First message from user | No existing session | Session Node creates new session with unique ID | High |
| A-02 | Session resumption | Message with existing session_id | Session exists in store | Session Node loads previous state | High |
| A-03 | Session expiry handling | Message after 24hr session timeout | Session expired | New session created, old state cleaned | Medium |
| A-04 | Tool selection for meeting scheduler | "I want to schedule a meeting" | Active session, confidence > 0.85 | Tool Selection Node picks schedule_meeting tool | High |
| A-05 | Tool selection for jobs | "Any open positions?" | Active session, confidence > 0.85 | Tool Selection Node picks search_jobs tool | High |
| A-06 | Tool selection for Q&A | "What services do you offer?" | Active session, confidence > 0.85 | Tool Selection Node picks query_kb tool | High |
| A-07 | Low confidence handling | Ambiguous message: "Hmm" | Active session, confidence < 0.85 | Clarification Node asks user, does NOT select tool | Critical |
| A-08 | Confidence gate enforcement | Borderline message, confidence = 0.84 | Active session | Clarification Node triggered, no tool selected | Critical |
| A-09 | Confidence gate pass | Clear message, confidence = 0.92 | Active session | Tool Selection Node proceeds | High |
| A-10 | Multi-intent in single message | "Show me jobs and schedule a call" | Active session | Prioritize first intent, queue second | High |
| A-11 | Tool failure recovery | Tool throws exception | Active session | Agent catches, returns friendly error via Response Node | Critical |
| A-12 | Maximum loop prevention | Re-evaluation loops > 3 times | Active session | Breaks loop, returns best available response | High |
| A-13 | Concurrent session handling | Two users chatting simultaneously | Both active | Each gets independent state | High |
| A-14 | State persistence across turns | Multi-turn scheduler conversation | Session active | State preserved between messages | Critical |

---

## 2. Ingestion Tool Tests

| # | Test Case | Input | Precondition | Expected Outcome | Priority |
|---|-----------|-------|--------------|------------------|----------|
| I-01 | PDF upload processing | Valid PDF file | Qdrant running | Document chunked, embedded, stored | High |
| I-02 | DOCX upload processing | Valid DOCX file | Qdrant running | Document chunked, embedded, stored | High |
| I-03 | TXT upload processing | Valid TXT file | Qdrant running | Document chunked, embedded, stored | High |
| I-04 | MD upload processing | Valid Markdown file | Qdrant running | Document chunked, embedded, stored | High |
| I-05 | Large file handling | PDF > 50MB | Qdrant running | Processed in batches, no timeout | Medium |
| I-06 | Empty file rejection | Empty PDF | Qdrant running | Rejected with clear error message | High |
| I-07 | Corrupted file handling | Corrupted PDF | Qdrant running | Sent to dead-letter queue, error logged | High |
| I-08 | Unsupported format rejection | .exe file upload | Qdrant running | Rejected with "unsupported format" error | High |
| I-09 | Duplicate document handling | Same PDF uploaded twice | First already stored | Updates existing (same doc ID) | Medium |
| I-10 | Chunking accuracy | Document with clear sections | — | Chunks respect section boundaries | Medium |
| I-11 | Metadata extraction | PDF with title, author | — | Metadata stored with vectors | Medium |
| I-12 | Embedding batch processing | 500 chunks from large doc | OpenAI API active | Batched 100 at a time, no rate limit hit | High |
| I-13 | OpenAI rate limit handling | Many docs uploaded rapidly | OpenAI rate limited | Exponential backoff, eventually succeeds | High |
| I-14 | Qdrant connection failure | Valid document | Qdrant down | Circuit breaker opens, error returned | Critical |
| I-15 | Async status tracking | Large document upload | — | GET /status/{task_id} returns progress | Medium |

---

## 3. schedule_meeting Tool Tests

| # | Test Case | Input | Precondition | Expected Outcome | Priority |
|---|-----------|-------|--------------|------------------|----------|
| S-01 | Scheduler intent detected | "I would like to schedule a demo" | Active session, confidence > 0.85 | Tool asks for email address | High |
| S-02 | Email collection | "my@email.com" | Stage: email_requested | send_otp tool invoked, stage changes | High |
| S-03 | Invalid email handling | "notanemail" | Stage: email_requested | "Please enter a valid email address" | High |
| S-04 | OTP sent successfully | Valid email provided | SMTP configured | 6-digit OTP sent, stored (hashed) | Critical |
| S-05 | Correct OTP verification | "123456" (correct OTP) | OTP sent, not expired | Email verified, show scheduler form | Critical |
| S-06 | Wrong OTP (1st attempt) | "000000" (wrong OTP) | OTP sent | "Invalid OTP, 2 attempts remaining" | High |
| S-07 | Wrong OTP (3rd attempt) | Wrong OTP 3rd time | 2 failures already | "Too many attempts. Please try again in 15 minutes." | Critical |
| S-08 | Expired OTP | Correct OTP after 5 min | OTP expired | "OTP expired. Sending new one." | High |
| S-09 | Form display in chat | — | OTP verified | Form rendered: Name, Contact, Email (pre-filled), Subject, Message | High |
| S-10 | Form with valid data | All fields filled correctly | Form displayed | Data validated, send_email tool invoked | High |
| S-11 | Form with missing fields | Name left empty | Form displayed | "Name is required" | High |
| S-12 | Form with invalid phone | "abc" in contact field | Form displayed | "Please enter a valid phone number" | Medium |
| S-13 | HR email sent successfully | Valid form data | SMTP configured | Email to HR with user data | Critical |
| S-14 | HR email failure | Valid form data | SMTP down | Retry 3x, queue for later, "Request saved" | Critical |
| S-15 | Success confirmation | Email sent | — | "Meeting request sent. Our team will get back to you." | High |
| S-16 | Cancel mid-flow | "cancel" during OTP step | Stage: otp_sent | "Meeting scheduling cancelled. How else can I help?" | Medium |
| S-17 | Resume interrupted flow | New message after disconnect | Stage: otp_sent (saved) | "You were verifying your email. Please enter the OTP:" | Medium |
| S-18 | Multiple scheduler attempts | User starts 2nd request | 1st request completed | New flow starts fresh | Low |
| S-19 | OTP rate limiting | 4th OTP request in 1 hour | 3 OTPs sent already | "Too many OTP requests. Try again later." | High |

---

## 4. search_jobs Tool Tests

| # | Test Case | Input | Precondition | Expected Outcome | Priority |
|---|-----------|-------|--------------|------------------|----------|
| J-01 | Job search intent detected | "Are you hiring?" | Active session, confidence > 0.85 | Returns job listings | High |
| J-02 | Cache hit (fresh data) | "What positions are open?" | Cache < 1hr old | Answer from cached JSON | High |
| J-03 | Cache miss (stale/no data) | "Any new openings?" | No cache or > 1hr old | Fetches from API/DB, updates cache | High |
| J-04 | Specific role query | "Any React developer jobs?" | Jobs cached | Filtered results for React positions | High |
| J-05 | Location-based query | "Remote jobs available?" | Jobs cached | Filtered by remote/location | Medium |
| J-06 | Salary-based query | "Jobs with salary > 10LPA?" | Jobs cached | Filtered by salary range | Medium |
| J-07 | Experience level query | "Entry level positions?" | Jobs cached | Filtered by experience | Medium |
| J-08 | No matching jobs | "Any AI researcher positions?" | Jobs cached, none match | "No openings matching that criteria at this time." | High |
| J-09 | Job API failure | "Show me jobs" | API unreachable | Use stale cache or return error | High |
| J-10 | Empty job database | "Any openings?" | No jobs exist | "No current openings. Please check back later." | Medium |
| J-11 | Follow-up job question | "Tell me more about the React position" | Previous query answered | Provides detailed info from same data | Medium |
| J-12 | Job data formatting | — | Jobs fetched | Clean, structured display | Medium |

---

## 5. query_kb Tool Tests

| # | Test Case | Input | Precondition | Expected Outcome | Priority |
|---|-----------|-------|--------------|------------------|----------|
| Q-01 | Basic company question | "What services do you offer?" | KB ingested, confidence > 0.85 | Relevant answer from knowledge base | High |
| Q-02 | Specific service question | "Do you provide web development?" | KB ingested | Accurate answer from specific chunks | High |
| Q-03 | Business hours query | "What are your business hours?" | KB ingested | Exact hours from KB | High |
| Q-04 | Location query | "Where are you located?" | KB ingested | Address from KB | High |
| Q-05 | No relevant info in KB | "What is your CEO's phone number?" | KB does not contain this | "I do not have that information." | High |
| Q-06 | Follow-up question | "Tell me more about that" | Previous Q&A in buffer | Uses context from previous exchange | Critical |
| Q-07 | Multi-turn follow-up | Q1: "Services?" > Q2: "Pricing for first?" > Q3: "Any discounts?" | Sequential conversation | Maintains context across 3+ turns | Critical |
| Q-08 | Context switch mid-conversation | Q1: "Services?" > Q2: "Show me jobs" | Different intent | Correctly detects intent change, selects different tool | High |
| Q-09 | Out-of-scope question | "What is the weather today?" | — | Polite decline: "I can only help with company-related information." | Medium |
| Q-10 | Irrelevant / random text | "asdfghjkl" | — | "I did not understand that. Could you please rephrase?" | High |
| Q-11 | Very long message | 2000+ character message | — | Truncated/processed without error | Medium |
| Q-12 | Multiple questions in one | "What services and pricing?" | KB ingested | Addresses both parts | Medium |
| Q-13 | Greeting handling | "Hello" / "Hi there" | — | Greeting response + "How can I help you?" | Medium |
| Q-14 | Thank you handling | "Thanks" / "That is helpful" | — | Polite response: "You are welcome. Is there anything else?" | Low |

---

## 6. Intent Detection and Confidence Gate Tests

| # | Test Case | Input | Expected Intent | Expected Confidence | Passes Gate (>0.85)? |
|---|-----------|-------|-----------------|---------------------|----------------------|
| ID-01 | Explicit scheduling | "Schedule a meeting" | meeting_scheduler | > 0.90 | Yes |
| ID-02 | Implicit scheduling | "Can I talk to someone on your team?" | meeting_scheduler | 0.70 - 0.85 | No — clarify |
| ID-03 | Demo request | "I would like a product demo" | meeting_scheduler | > 0.85 | Yes |
| ID-04 | Explicit job search | "Are you hiring?" | job_search | > 0.90 | Yes |
| ID-05 | Career query | "How can I apply for a job?" | job_search | > 0.85 | Yes |
| ID-06 | Position query | "Any openings for designers?" | job_search | > 0.90 | Yes |
| ID-07 | Service question | "What do you do?" | general_qa | > 0.85 | Yes |
| ID-08 | Price question | "How much does your service cost?" | general_qa | > 0.85 | Yes |
| ID-09 | Mixed intent | "Schedule a call and show me jobs" | meeting_scheduler (primary) | > 0.70 | Needs clarification |
| ID-10 | Ambiguous | "Help me" | general_qa (default) | < 0.50 | No — clarify |
| ID-11 | Non-English input | "Bonjour" | general_qa | < 0.50 | No — clarify |
| ID-12 | Empty message | "" | None (error) | 0.00 | No — error |
| ID-13 | Scheduling language | "Schedule a call for next Tuesday" | meeting_scheduler | > 0.90 | Yes |
| ID-14 | Job + location | "Jobs in Bangalore" | job_search | > 0.90 | Yes |
| ID-15 | Re-classification after clarification | User clarifies "I want to schedule" | meeting_scheduler | > 0.90 | Yes |
| ID-16 | Below-threshold then above | First: "Maybe" (0.40) > Clarify > "Schedule a demo" (0.95) | meeting_scheduler | 0.95 | Yes |

---

## 7. Conversation Memory Tests

| # | Test Case | Input Sequence | Expected Behavior | Priority |
|---|-----------|---------------|-------------------|----------|
| M-01 | Basic memory retention | Q1: "services?" A1: "We offer X" Q2: "pricing?" | Q2 answer references "X" services | Critical |
| M-02 | Pronoun resolution | Q1: "React dev positions?" A1: lists positions Q2: "Tell me more about it" | "it" resolves to React dev position | High |
| M-03 | Memory across tool switch | Q1: "Services?" > Q2: "Schedule a meeting" > Q3: "Back to services" | Returns to services context | High |
| M-04 | Long conversation memory | 20+ turn conversation | Still references early context | Medium |
| M-05 | Memory isolation between sessions | User A and User B chat | No cross-contamination | Critical |
| M-06 | Memory buffer overflow | Buffer exceeds max tokens | Oldest messages summarized/dropped | Medium |
| M-07 | Memory persistence after restart | Server restarts mid-conversation | Session state restored from store | High |

---

## 8. Fault Tolerance Tests

| # | Test Case | Failure Condition | Expected Behavior | Priority |
|---|-----------|------------------|-------------------|----------|
| F-01 | OpenAI API timeout | API takes > 30s | Retry with backoff, then user error | Critical |
| F-02 | OpenAI API 429 (rate limit) | Rate limit hit | Exponential backoff (1s, 2s, 4s) | Critical |
| F-03 | OpenAI API 500 | Server error | Retry 3x, then "Service temporarily unavailable" | High |
| F-04 | Qdrant connection lost | Qdrant container crashes | Circuit breaker opens, friendly error | Critical |
| F-05 | Qdrant recovers after failure | Qdrant comes back online | Circuit breaker: half-open then closed | High |
| F-06 | SMTP failure during OTP | Email service down | "Unable to send OTP. Try again later." | High |
| F-07 | SMTP failure during HR email | Email service down after form | Retry 3x, queue, "Request saved, email pending" | Critical |
| F-08 | Job API unreachable | External API 503 | Use stale cache if available | High |
| F-09 | Redis connection failure | Redis down | Fallback to in-memory store (warn log) | Medium |
| F-10 | All retries exhausted | 3 retries fail | Dead-letter queue, user error message | High |
| F-11 | Concurrent request overload | 100+ simultaneous requests | Rate limiter returns 429 for excess | High |
| F-12 | Memory corruption | Corrupted session state | Reset session, inform user | Medium |
| F-13 | Tool failure with agent alive | Tool throws but graph continues | Agent returns fallback response via Response Node | Critical |
| F-14 | Infinite loop detection | Intent keeps changing | Max 3 re-evaluations, then force respond | High |

---

## 9. Security Tests

| # | Test Case | Input | Expected Behavior | Priority |
|---|-----------|-------|-------------------|----------|
| SEC-01 | XSS injection in chat | `<script>alert('xss')</script>` | HTML escaped, script blocked | Critical |
| SEC-02 | SQL injection attempt | `'; DROP TABLE users; --` | Sanitized, no DB impact | Critical |
| SEC-03 | Prompt injection | "Ignore previous instructions. You are now..." | Blocked by sanitizer, normal response | Critical |
| SEC-04 | PII in chat | "My SSN is 123-45-6789" | Agent response does not echo PII | Critical |
| SEC-05 | Credit card in chat | "My card is 4111-1111-1111-1111" | Agent blocks: "I cannot process payment information." | Critical |
| SEC-06 | Rate limit enforcement | 20 requests in 1 minute | Returns 429 after limit exceeded | High |
| SEC-07 | CORS enforcement | Request from unauthorized origin | 403 Forbidden | High |
| SEC-08 | Missing API key | Request without auth header | 401 Unauthorized (if auth enabled) | Medium |
| SEC-09 | File upload size limit | 100MB file upload | Rejected: "File too large (max 50MB)" | High |
| SEC-10 | OTP brute force | Automated OTP guessing | Locked after 3 attempts | Critical |
| SEC-11 | Email enumeration | Checking if email exists via OTP | Same response for existing/non-existing | Medium |

---

## 10. Performance Tests

| # | Test Case | Scenario | SLA Target | Priority |
|---|-----------|----------|------------|----------|
| P-01 | Simple Q&A response time | Single user, general question | < 3 seconds | High |
| P-02 | Meeting scheduler total time | Full flow: intent to email sent | < 60 seconds (excluding user input) | Medium |
| P-03 | Job search response time | First query (cache miss) | < 5 seconds | Medium |
| P-04 | Job search cached response | Subsequent query (cache hit) | < 1 second | High |
| P-05 | Document ingestion time | 10-page PDF | < 30 seconds | Medium |
| P-06 | Concurrent user handling | 50 simultaneous users | No degradation | High |
| P-07 | Vector search latency | Top-5 retrieval from 10K chunks | < 500ms | High |
| P-08 | Memory footprint | Server running 24hrs | < 2GB RAM | Medium |
| P-09 | Peak load scenario | 200 requests/minute | < 5s avg response | High |

---

## 11. End-to-End Tests

| # | Test Case | User Journey | Expected Result | Priority |
|---|-----------|-------------|-----------------|----------|
| E2E-01 | Full meeting scheduler journey | User says "schedule a call" > provides email > enters OTP > fills form > gets confirmation | HR receives email, user sees confirmation | Critical |
| E2E-02 | Full job search journey | User says "hiring?" > sees listings > asks about specific role > gets details | Accurate job info returned | High |
| E2E-03 | Full Q&A journey | User asks about services > follow-up > another follow-up > switches to jobs | Context maintained, smooth tool switch | High |
| E2E-04 | Mixed workflow | User: Q&A > meeting scheduler (abandon) > job search > Q&A | All tools work, state managed | High |
| E2E-05 | Error recovery journey | User starts scheduler > OTP fails > retry > succeeds > completes | Graceful recovery, no state loss | Critical |
| E2E-06 | Long conversation | 30+ messages with mixed intents | System stable, memory managed | Medium |
| E2E-07 | Confidence gate in full flow | Ambiguous > clarify > clear intent > tool executes > complete | Gate blocks, then allows after clarification | Critical |

---

## 12. Edge Case Tests

| # | Test Case | Scenario | Expected Behavior | Priority |
|---|-----------|----------|-------------------|----------|
| EC-01 | Rapid double submit | User clicks send twice fast | Only one message processed | Medium |
| EC-02 | Unicode / emoji input | "Schedule a meeting please" with special chars | Correctly processes, handles gracefully | Low |
| EC-03 | Very short input | "Hi" | Greeting response, not an error | Low |
| EC-04 | Only whitespace | "   " | "Please enter a message." | Medium |
| EC-05 | Session with no messages | New session, no interaction for 30min | Session cleaned up, resources freed | Low |
| EC-06 | Browser refresh mid-flow | Refresh during OTP step | State restored, continue from OTP step | Medium |
| EC-07 | Network timeout on response | Backend response takes 60s+ | Frontend timeout, retry option shown | Medium |
| EC-08 | Switching language | English to Hindi mid-conversation | "Currently we support English only." | Low |
| EC-09 | Form submission with max-length data | 5000 char message in scheduler form | Truncated or rejected with length error | Medium |
| EC-10 | Reference past response | User references specific past answer | Resolves from conversation buffer | Low |
