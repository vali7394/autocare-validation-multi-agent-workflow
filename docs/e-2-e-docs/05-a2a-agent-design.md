# ACES Fitment Validation — A2A Agent Design (JobQueryAgent)

> **Document Purpose**: Design for externalizing the JobQueryAgent as a Google A2A (Agent-to-Agent) protocol-compliant service  
> **Parent Document**: [E2E System Design](./00-e2e-design.md)  
> **Version**: 1.0  
> **Last Updated**: June 2026  
> **Implementation Timeline**: Post-MVP (Phase 10)

---

## 1. Overview

### 1.1 What is A2A?

The **Agent-to-Agent (A2A) protocol** is an open standard introduced by Google for enabling communication between AI agents across organizational and platform boundaries. It provides:

- **Agent Cards** — JSON metadata describing an agent's capabilities, skills, and endpoint
- **Task Lifecycle** — A standardized request/response model with states: `submitted → working → completed / failed`
- **Streaming** — Server-Sent Events (SSE) for long-running tasks
- **Discovery** — Agents publish Agent Cards at a well-known URL (`/.well-known/agent.json`) for programmatic discovery

### 1.2 Why Externalize an Agent?

| Benefit | Description |
|---------|-------------|
| **Federation** | External systems (supplier agents, partner validation tools) can discover and query validation results without custom integration |
| **Decoupled Deployment** | The query agent scales independently from the validation pipeline |
| **Standardized Interface** | Any A2A-compliant client can interact — no proprietary API to learn |
| **Future Ecosystem** | As A2A adoption grows, the agent becomes composable with other agents in the automotive data ecosystem |

### 1.3 Which Agent?

The **JobQueryAgent** (Conversational Engine) is selected for A2A externalization:

| Criterion | JobQueryAgent | Pipeline Nodes (Validation, Comparison, etc.) |
|-----------|--------------|-----------------------------------------------|
| Architectural independence | ✅ Standalone, not part of DAG | ❌ Tightly coupled to LangGraph state |
| Statefulness | ✅ Stateless per request (just needs `job_id`) | ❌ Depends on shared pipeline state |
| Safety for external access | ✅ Read-only Snowflake tools | ❌ Write operations on staging tables |
| Clear capability boundary | ✅ "Answer questions about a validation job" | ❌ Too granular / internal |
| Real external use case | ✅ Suppliers query their job results | ❌ No external consumer |

---

## 2. A2A Agent Card

The Agent Card is published at `https://<host>/.well-known/agent.json` and describes the agent's capabilities to external consumers.

```json
{
  "name": "ACES Validation Query Agent",
  "description": "An AI agent that answers questions about ACES fitment validation job results, including validation errors, uplift statistics, failure root causes, and brand/part performance metrics.",
  "url": "https://<host>/a2a",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "job-summary",
      "name": "Job Summary",
      "description": "Provides high-level validation and uplift statistics for a completed job, including total/valid/invalid record counts and ADD/UPDATE/DELETE/UNCHANGED classification breakdown.",
      "tags": ["aces", "validation", "summary", "fitment"],
      "examples": [
        "Give me the summary for job job_01h9ybc93k018s96abc491pq",
        "How many records passed validation?",
        "What's the uplift breakdown?"
      ]
    },
    {
      "id": "failure-analysis",
      "name": "Failure Analysis",
      "description": "Analyzes validation failures by error type (VCDB, PCDB, QDB, PRODUCT), identifies root causes, and provides specific failed record examples with error details.",
      "tags": ["aces", "validation", "errors", "debugging"],
      "examples": [
        "Why did part BP-2120 fail?",
        "Show me the top VCDB failures",
        "What are the most common error causes?"
      ]
    },
    {
      "id": "uplift-details",
      "name": "Uplift Details",
      "description": "Provides details about catalog changes — new additions, updates, and deletions — with specific fitment records and change descriptions.",
      "tags": ["aces", "catalog", "changes", "uplift"],
      "examples": [
        "Show me the new fitments being added",
        "Which fitments were deleted?",
        "What changed in the update records?"
      ]
    },
    {
      "id": "performance-metrics",
      "name": "Brand & Part Performance",
      "description": "Summarizes error rates grouped by part number or line code to identify systemic data quality issues.",
      "tags": ["aces", "quality", "brand", "part"],
      "examples": [
        "Which parts have the highest failure rate?",
        "Show me performance by line code"
      ]
    }
  ],
  "authentication": {
    "schemes": ["bearer"]
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"]
}
```

---

## 3. Architecture

### 3.1 Deployment Model

```
                        External Agent / Client
                                │
                                │ A2A Protocol (HTTPS + JSON-RPC)
                                ▼
┌─────────────────────────────────────────────────┐
│           A2A Server (Standalone Service)        │
│                                                  │
│   ┌──────────────────────────────────────────┐   │
│   │  A2A Protocol Handler                    │   │
│   │  - Agent Card endpoint (GET /.well-known)│   │
│   │  - tasks/send (POST)                     │   │
│   │  - tasks/sendSubscribe (POST + SSE)      │   │
│   │  - tasks/get (GET)                       │   │
│   └──────────────┬───────────────────────────┘   │
│                  │                               │
│   ┌──────────────▼───────────────────────────┐   │
│   │  JobQueryAgent (LLM + Tools)             │   │
│   │  - System prompt with job_id scope       │   │
│   │  - 5 read-only Snowflake tools           │   │
│   │  - Streaming token generation            │   │
│   └──────────────┬───────────────────────────┘   │
│                  │                               │
└──────────────────┼───────────────────────────────┘
                   │ SQL queries (read-only)
                   ▼
          ┌─────────────────┐
          │    Snowflake     │
          │  STAGING + REF   │
          └─────────────────┘
```

### 3.2 Relationship to MVP Architecture

In the MVP, the JobQueryAgent runs **inside the BFF process** and is invoked via REST API (`POST /api/jobs/{job_id}/chat`). The A2A design wraps the **same agent logic** with an A2A-compliant server:

```
MVP (Internal):
  Browser → BFF REST API → JobQueryAgent → Snowflake

Post-MVP (A2A, External):
  External Agent → A2A Server → JobQueryAgent → Snowflake

Post-MVP (A2A, Internal — optional migration):
  Browser → BFF REST API → A2A Client → A2A Server → JobQueryAgent → Snowflake
```

The HITL UI can continue using the existing REST endpoint, or the BFF can be updated to act as an internal A2A client — both approaches work.

---

## 4. A2A Task Lifecycle

### 4.1 Request Flow

```
External Agent                    A2A Server                    JobQueryAgent
     │                                │                              │
     │── 1. POST /a2a ───────────────►│                              │
     │   { method: "tasks/send",      │                              │
     │     params: {                   │                              │
     │       id: "task-123",           │                              │
     │       message: {                │                              │
     │         role: "user",           │                              │
     │         parts: [{               │                              │
     │           type: "text",         │                              │
     │           text: "Summarize      │                              │
     │            job XYZ failures"    │                              │
     │         }]                      │                              │
     │       }                         │                              │
     │     }                           │                              │
     │   }                             │                              │
     │                                │── 2. Extract job_id ────────►│
     │                                │   + user message              │
     │                                │                              │
     │                                │       3. Tool calls ─────────┼──► Snowflake
     │                                │          (read-only)          │
     │                                │                              │
     │                                │◄── 4. Generated response ───│
     │◄── 5. A2A Response ───────────│                              │
     │   { result: {                  │                              │
     │     id: "task-123",            │                              │
     │     status: { state: "completed" },                           │
     │     artifacts: [{              │                              │
     │       parts: [{                │                              │
     │         type: "text",          │                              │
     │         text: "Job XYZ had..." │                              │
     │       }]                       │                              │
     │     }]                         │                              │
     │   }}                           │                              │
```

### 4.2 Streaming Flow (tasks/sendSubscribe)

For longer analytical queries, the A2A server supports SSE streaming:

```
External Agent                    A2A Server
     │                                │
     │── POST /a2a ──────────────────►│  (tasks/sendSubscribe)
     │                                │
     │◄── SSE: status=working ────────│
     │◄── SSE: artifact chunk 1 ──────│  ("Job XYZ processed 102,400...")
     │◄── SSE: artifact chunk 2 ──────│  ("Top failure: VCDB base vehicle...")
     │◄── SSE: artifact chunk 3 ──────│  ("| Part | Error Count | ...")
     │◄── SSE: status=completed ──────│
```

### 4.3 Task States

| State | When |
|-------|------|
| `submitted` | A2A server received the request |
| `working` | JobQueryAgent is querying Snowflake tools |
| `completed` | Response generated successfully |
| `failed` | Tool error, invalid job_id, or agent error |

---

## 5. Job ID Resolution

External agents must provide a `job_id` to scope their queries. Two approaches:

### Option A: Explicit in Message (Simple)

The external agent includes the `job_id` in the natural language message:
```
"Summarize the validation results for job job_01h9ybc93k018s96abc491pq"
```
The JobQueryAgent extracts it from the message using its system prompt instructions.

### Option B: Metadata Field (Structured)

Use the A2A message metadata to pass `job_id` as a structured parameter:
```json
{
  "message": {
    "role": "user",
    "parts": [{ "type": "text", "text": "What are the top failures?" }],
    "metadata": {
      "job_id": "job_01h9ybc93k018s96abc491pq"
    }
  }
}
```
The A2A server extracts `job_id` from metadata and injects it into the agent's system prompt.

**Recommendation**: Support both. Option B is preferred for programmatic agent-to-agent calls; Option A works for human-driven interactions.

---

## 6. Security Considerations

Since this agent is externally accessible, security is critical (though implemented post-MVP alongside Phase 9 security hardening):

| Concern | Mitigation |
|---------|------------|
| **Authentication** | Bearer token (API key or JWT) required on all A2A requests |
| **Job-ID Authorization** | Verify the requesting agent/user has access to the specified `job_id` |
| **Read-Only Enforcement** | Agent tools are strictly read-only; no write operations exposed |
| **Rate Limiting** | Per-client rate limits on the A2A endpoint |
| **Response Size** | Tool responses capped at 50 records; no full data dumps |
| **Input Validation** | Reject malformed A2A messages; sanitize `job_id` input |

---

## 7. Technology Stack

| Component | Technology | Notes |
|-----------|------------|-------|
| A2A Server Framework | **Python (`a2a-sdk`)** or custom FastAPI | Google provides a Python SDK for A2A server implementation |
| Agent Runtime | **LangGraph** or **LangChain** | Same JobQueryAgent logic as MVP |
| LLM | Same as MVP conversational engine | GPT-4o, Claude, or Gemini |
| Transport | **HTTPS + JSON-RPC 2.0** | A2A standard transport |
| Streaming | **Server-Sent Events (SSE)** | For `tasks/sendSubscribe` |
| Discovery | `/.well-known/agent.json` | Static JSON served at well-known URL |

---

## 8. Use Cases for External A2A Access

| Consumer | Use Case |
|----------|----------|
| **Supplier Agent** | Supplier's own AI agent queries validation results after receiving job completion notification: *"Did my latest fitment file pass? What failed?"* |
| **Partner Validation System** | A partner's validation tool queries our system to cross-reference results |
| **Internal Dashboard Agent** | A separate BI/reporting agent aggregates validation metrics across multiple jobs |
| **Automated Remediation Agent** | An agent that receives failure summaries and automatically generates corrected ACES XML for resubmission |

---

## 9. Implementation Plan (Post-MVP)

### Phase 10: A2A Agent Externalization

| Step | Task | Dependency |
|------|------|------------|
| 1 | Extract JobQueryAgent into standalone module | MVP Phase 7 complete |
| 2 | Implement A2A server using `a2a-sdk` or FastAPI | — |
| 3 | Define and publish Agent Card at `/.well-known/agent.json` | — |
| 4 | Implement `tasks/send` endpoint (synchronous) | Steps 1-2 |
| 5 | Implement `tasks/sendSubscribe` endpoint (SSE streaming) | Step 4 |
| 6 | Add job_id metadata extraction from A2A messages | Step 4 |
| 7 | Add bearer token authentication | Phase 9 (Security) |
| 8 | Add job-ID authorization check | Phase 9 (Security) |
| 9 | Deploy A2A server as standalone service | — |
| 10 | Integration testing with external A2A client | Step 9 |

**Estimated Effort**: 2-3 sprints after MVP completion.

---

## 10. References

| Document | Relevance |
|----------|-----------|
| [E2E System Design](./00-e2e-design.md) | Master architecture context |
| [HITL UI Design](./02-hitl-ui-design.md) | JobQueryAgent MVP implementation (Section 5) |
| [Google A2A Protocol Spec](https://github.com/google/A2A) | Official A2A protocol repository |
| [A2A SDK (Python)](https://github.com/google/A2A/tree/main/sdk/python) | Reference implementation |

---

*ACES Fitment Validation — A2A Agent Design v1.0 (Post-MVP)*
