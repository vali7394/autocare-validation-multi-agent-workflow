# Task Status Tracker — Single Source of Truth

> **Purpose**: Tracks execution status of every task in the ACES Fitment Validation build. Workers MUST read this file before doing anything and update it as they work, following the protocol in [WORKER_PROMPT.md](./WORKER_PROMPT.md).
> **Task specs**: [master_backlog.md](./master_backlog.md) (ordering) · [backend_tasks.md](./backend_tasks.md) · [ui_tasks.md](./ui_tasks.md) · [agent_tasks.md](./agent_tasks.md)

## Status Legend

| Status | Meaning |
|--------|---------|
| `NOT_STARTED` | No work done (or owner has reset it — worker must reconcile against actual repo/cloud state before redoing) |
| `IN_PROGRESS` | A worker has picked this up; partial work may exist |
| `BLOCKED` | Cannot proceed — reason MUST be in Notes; needs owner input |
| `DONE` | Implemented AND its Incremental Verification passed; evidence in Notes |
| `SKIPPED` | Owner explicitly deferred (e.g., optional Phase 8/9 items) |

**Editing rules**: Only ever change the `Status`, `Updated`, and `Notes / Evidence` cells, and append to the Worklog. Never renumber, reorder, add, or delete rows without owner approval. `Step` = master backlog step; tasks sharing a Step are parallelizable. A `GATE-N` row is that phase's Live Integration Check from the master backlog — it must be `DONE` before any task in phase N+1 starts.

---

## Phase 1: The Deployed Walking Skeleton

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 1.1 | BE-1.1 | GCP project bootstrap + APIs | — | NOT_STARTED | — | — |
| 1.1 | BE-1.2 | Artifact Registry + SAs + IAM | BE-1.1 | NOT_STARTED | — | — |
| 1.2 | BE-1.3 | Monorepo + FastAPI BFF skeleton + Dockerfile | — | NOT_STARTED | — | — |
| 1.2 | BE-1.4 | Cloud Build CI/CD → Cloud Run (dev) | BE-1.2, BE-1.3 | NOT_STARTED | — | — |
| 1.3 | BE-1.5 | Snowflake foundation (DB/schemas/WH/user) | — | NOT_STARTED | — | — |
| 1.4 | BE-1.6 | Secret Manager + Snowflake connectivity | BE-1.4, BE-1.5 | NOT_STARTED | — | — |
| 1.4 | BE-1.7 | JOB_RUN table + jobs read API + dev-seed | BE-1.6 | NOT_STARTED | — | — |
| 1.5 | UI-1.1 | React scaffold + app shell | — | NOT_STARTED | — | — |
| 1.5 | UI-1.2 | Typed API client + env config | UI-1.1 | NOT_STARTED | — | — |
| 1.5 | BE-1.8 | Firebase Hosting + CORS on BFF | BE-1.4 | NOT_STARTED | — | — |
| 1.5 | UI-1.5 | SPA continuous deployment | UI-1.1, BE-1.8 | NOT_STARTED | — | — |
| 1.6 | UI-1.3 | System status widget | UI-1.2, BE-1.6 | NOT_STARTED | — | — |
| 1.6 | UI-1.4 | Jobs table v0 | UI-1.2, BE-1.7 | NOT_STARTED | — | — |
| 1.7 | AG-1.1 | LLM client wrapper | BE-1.6 | NOT_STARTED | — | — |
| 1.7 | AG-1.2 | Streaming echo chat endpoint | AG-1.1, BE-1.4 | NOT_STARTED | — | — |
| 1.8 | BE-1.9 | Orchestrator service shell + OIDC auth | BE-1.4 | NOT_STARTED | — | — |
| 1.8 | AG-1.3 | LangGraph skeleton graph | BE-1.9, BE-1.7 | NOT_STARTED | — | — |
| 1.9 | GATE-1 | Phase 1 Live Integration Check | All Phase 1 | NOT_STARTED | — | — |

## Phase 2: Ingestion & Staging Slice

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 2.1 | BE-2.1 | GCS buckets + lifecycle + IAM | BE-1.2 | NOT_STARTED | — | — |
| 2.1 | BE-2.3 | Staging tables + stage + file format | BE-1.5 | NOT_STARTED | — | — |
| 2.2 | BE-2.4 | Streaming SAX parser module | — | NOT_STARTED | — | — |
| 2.2 | BE-2.5 | DFH utility w/ Snowflake parity | BE-2.4 | NOT_STARTED | — | — |
| 2.2 | BE-2.6 | Snowflake batch loader (idempotent) | BE-2.3, BE-2.5 | NOT_STARTED | — | — |
| 2.3 | BE-2.2 | Streaming upload endpoint → GCS | BE-1.7, BE-2.1 | NOT_STARTED | — | — |
| 2.4 | AG-2.1 | Full state schema + status writer | AG-1.3 | NOT_STARTED | — | — |
| 2.4 | AG-2.2 | StagingLoadNode | AG-2.1, BE-2.4, BE-2.5, BE-2.6, BE-2.1 | NOT_STARTED | — | — |
| 2.4 | AG-2.3 | Failure edges + FinalStatusNode v1 | AG-2.1 | NOT_STARTED | — | — |
| 2.4 | AG-2.4 | Graph wiring v1 + idempotent run endpoint | AG-2.2, AG-2.3 | NOT_STARTED | — | — |
| 2.5 | BE-2.7 | Cloud Tasks queue + dispatch | BE-2.2, BE-1.9, AG-2.4 | NOT_STARTED | — | — |
| 2.5 | BE-2.8 | Job detail + enriched list endpoints | BE-2.2 | NOT_STARTED | — | — |
| 2.6 | UI-2.1 | Upload modal (dropzone + checks) | UI-1.1 | NOT_STARTED | — | — |
| 2.6 | UI-2.2 | Client preflight web worker | UI-2.1 | NOT_STARTED | — | — |
| 2.6 | UI-2.3 | Upload submission flow (202) | UI-2.2, BE-2.2 | NOT_STARTED | — | — |
| 2.6 | UI-2.4 | Dashboard v1 (metrics + polling) | UI-1.4, BE-2.8 | NOT_STARTED | — | — |
| 2.7 | GATE-2 | Phase 2 Live Integration Check | All Phase 2 | NOT_STARTED | — | — |

## Phase 3: Real-Time Progress

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 3.1 | BE-3.1 | Memorystore Redis + VPC wiring | BE-1.4, BE-1.9 | NOT_STARTED | — | — |
| 3.2 | BE-3.2 | Event schema + pub/sub library | BE-3.1 | NOT_STARTED | — | — |
| 3.2 | AG-3.1 | Node lifecycle event decorator | BE-3.2, AG-2.4 | NOT_STARTED | — | — |
| 3.2 | AG-3.2 | Granular staging progress events | AG-3.1 | NOT_STARTED | — | — |
| 3.3 | BE-3.3 | SSE endpoint (replay + heartbeat) | BE-3.2 | NOT_STARTED | — | — |
| 3.3 | BE-3.4 | Polling fallback + status persistence | BE-3.3, AG-3.1 | NOT_STARTED | — | — |
| 3.4 | AG-3.3 | Warning/error log ticker events | AG-3.1 | NOT_STARTED | — | — |
| 3.5 | UI-3.1 | SSE client hook | BE-3.3 | NOT_STARTED | — | — |
| 3.5 | UI-3.2 | Progress tracker stepper screen | UI-3.1, AG-3.1 | NOT_STARTED | — | — |
| 3.5 | UI-3.3 | Live error log ticker | UI-3.1, AG-3.3 | NOT_STARTED | — | — |
| 3.5 | UI-3.4 | Dashboard live status integration | UI-3.2, BE-3.4 | NOT_STARTED | — | — |
| 3.6 | GATE-3 | Phase 3 Live Integration Check | All Phase 3 | NOT_STARTED | — | — |

## Phase 4: Validation Core

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 4.1 | BE-4.1 | AUTOCARE reference DDL | BE-1.5 | NOT_STARTED | — | — |
| 4.1 | BE-4.2 | Reference data load pipeline | BE-4.1 | NOT_STARTED | — | — |
| 4.2 | BE-4.3 | Error model upgrade (VARIANT arrays) | BE-2.3 | NOT_STARTED | — | — |
| 4.3 | BE-4.4 | VCDB validation procedures | BE-4.2, BE-4.3 | NOT_STARTED | — | — |
| 4.3 | BE-4.5 | PCDB validation procedures | BE-4.3 | NOT_STARTED | — | — |
| 4.3 | BE-4.6 | QDB validation procedure | BE-4.3 | NOT_STARTED | — | — |
| 4.3 | BE-4.7 | PRODUCT/BRAND tables + resolution proc | BE-4.3 | NOT_STARTED | — | — |
| 4.4 | BE-4.8 | MCP server (8 validation tools) | BE-4.4, BE-4.5, BE-4.6, BE-4.7 | NOT_STARTED | — | — |
| 4.4 | BE-4.9 | MCP REST auth (PAT) | BE-4.8 | NOT_STARTED | — | — |
| 4.5 | AG-4.1 | MCP client module | BE-4.9 | NOT_STARTED | — | — |
| 4.5 | AG-4.2 | SchemaValidationNode (XSD) | AG-2.4, BE-2.1 | NOT_STARTED | — | — |
| 4.5 | AG-4.3 | PreflightNode | AG-4.2, BE-4.2 | NOT_STARTED | — | — |
| 4.5 | AG-4.4 | ValidationNode (parallel fan-out) | AG-4.1, BE-4.8 | NOT_STARTED | — | — |
| 4.5 | AG-4.5 | Retry policy + checkpoint integration | AG-4.4 | NOT_STARTED | — | — |
| 4.5 | AG-4.6 | Full graph wiring v2 | AG-4.2, AG-4.3, AG-4.4, AG-4.5 | NOT_STARTED | — | — |
| 4.6 | BE-4.10 | Job validation summary endpoint | BE-2.8, BE-4.4 | NOT_STARTED | — | — |
| 4.6 | BE-4.11 | Failed-records endpoint | BE-4.10 | NOT_STARTED | — | — |
| 4.7 | UI-4.1 | Live validation counters | UI-3.2, AG-4.4 | NOT_STARTED | — | — |
| 4.7 | UI-4.2 | Job detail validation summary card | UI-3.4, BE-4.10 | NOT_STARTED | — | — |
| 4.7 | UI-4.3 | Failed-records drawer v1 | UI-4.2, BE-4.11 | NOT_STARTED | — | — |
| 4.7 | UI-4.4 | Preflight warnings modal | UI-2.2, AG-4.3 | NOT_STARTED | — | — |
| 4.8 | GATE-4 | Phase 4 Live Integration Check | All Phase 4 | NOT_STARTED | — | — |

## Phase 5: Comparison, Statistics & Reports

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 5.1 | BE-5.1 | CATALOG.FITMENT + snapshot seeding | BE-2.3, BE-4.7 | NOT_STARTED | — | — |
| 5.2 | BE-5.2 | Comparison procs (w/ review fixes) | BE-5.1 | NOT_STARTED | — | — |
| 5.2 | BE-5.3 | Aggregation procs + persistence | BE-5.2 | NOT_STARTED | — | — |
| 5.3 | BE-5.4 | MCP server update (12 tools total) | BE-5.3, BE-4.8 | NOT_STARTED | — | — |
| 5.4 | AG-5.1 | ComparisonNode | AG-4.6, BE-5.4 | NOT_STARTED | — | — |
| 5.4 | AG-5.2 | StatisticsNode | AG-5.1 | NOT_STARTED | — | — |
| 5.4 | AG-5.3 | ReportWriterNode | AG-5.2, BE-2.1 | NOT_STARTED | — | — |
| 5.4 | AG-5.4 | FinalStatusNode v2 | AG-5.2, AG-5.3 | NOT_STARTED | — | — |
| 5.5 | BE-5.5 | Report storage + download endpoint | BE-2.1, AG-5.3 | NOT_STARTED | — | — |
| 5.5 | BE-5.6 | Analysis data endpoints | BE-5.3, BE-2.1 | NOT_STARTED | — | — |
| 5.6 | UI-5.1 | Split-screen analysis layout | UI-4.2 | NOT_STARTED | — | — |
| 5.6 | UI-5.2 | Summary card + uplift pie chart | UI-5.1, BE-5.6 | NOT_STARTED | — | — |
| 5.6 | UI-5.3 | Error bar chart + top failures | UI-5.1, BE-5.6 | NOT_STARTED | — | — |
| 5.6 | UI-5.4 | Sample drawer v2 + XML snippet | UI-4.3, BE-5.6 | NOT_STARTED | — | — |
| 5.6 | UI-5.5 | Report download integration | UI-5.1, BE-5.5 | NOT_STARTED | — | — |
| 5.7 | GATE-5 | Phase 5 Live Integration Check | All Phase 5 | NOT_STARTED | — | — |

## Phase 6: Conversational Analytics

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 6.1 | BE-6.1 | Read-only chat tool queries (5) | BE-5.3 | NOT_STARTED | — | — |
| 6.1 | BE-6.3 | Read-only role segregation | BE-6.1 | NOT_STARTED | — | — |
| 6.2 | AG-6.1 | JobQueryAgent scaffold | AG-1.1, BE-6.1 | NOT_STARTED | — | — |
| 6.2 | AG-6.2 | Tool bindings (server-forced job_id) | AG-6.1 | NOT_STARTED | — | — |
| 6.2 | AG-6.3 | Streaming + tool-activity events | AG-6.1, AG-6.2 | NOT_STARTED | — | — |
| 6.3 | BE-6.2 | Chat endpoint (chunked streaming) | BE-6.1, AG-6.3 | NOT_STARTED | — | — |
| 6.4 | UI-6.1 | Chat sidebar + markdown | UI-5.1 | NOT_STARTED | — | — |
| 6.4 | UI-6.2 | Streaming consumption + indicator | UI-6.1, BE-6.2 | NOT_STARTED | — | — |
| 6.4 | UI-6.3 | Suggestion chips + input box | UI-6.2 | NOT_STARTED | — | — |
| 6.4 | UI-6.4 | Chat entry points | UI-6.2, UI-3.2 | NOT_STARTED | — | — |
| 6.5 | AG-6.4 | Edge cases + prompt tuning | AG-6.3 | NOT_STARTED | — | — |
| 6.5 | AG-6.5 | Session memory (Redis) | AG-6.3, BE-3.1 | NOT_STARTED | — | — |
| 6.6 | GATE-6 | Phase 6 Live Integration Check | All Phase 6 | NOT_STARTED | — | — |

## Phase 7: Hardening, Performance & Operations

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 7.1 | BE-7.1 | 100K benchmark + warehouse sizing | GATE-5 | NOT_STARTED | — | — |
| 7.2 | AG-7.1 | Durable checkpointing + resume | AG-5.4 | NOT_STARTED | — | — |
| 7.2 | AG-7.2 | Failure injection drills | AG-7.1 | NOT_STARTED | — | — |
| 7.2 | AG-7.3 | Concurrent jobs validation | AG-7.1 | NOT_STARTED | — | — |
| 7.3 | BE-7.2 | Staging retention cleanup task | BE-2.3 | NOT_STARTED | — | — |
| 7.3 | BE-7.3 | Resumable upload + oversize gate | BE-2.2, BE-3.1 | NOT_STARTED | — | — |
| 7.3 | UI-7.1 | Resumable upload UX | UI-2.3, BE-7.3 | NOT_STARTED | — | — |
| 7.4 | BE-7.4 | Monitoring dashboards + alerts | BE-3.2 | NOT_STARTED | — | — |
| 7.4 | UI-7.2 | State polish + reconnect UX sweep | Phases 2–6 UI | NOT_STARTED | — | — |
| 7.5 | BE-7.5 | Production env + promotion pipeline | BE-7.1, BE-7.4 | NOT_STARTED | — | — |
| 7.5 | UI-7.3 | Production build optimization | UI-7.2, BE-7.5 | NOT_STARTED | — | — |
| 7.6 | GATE-7 | Phase 7 Live Integration Check (prod) | All Phase 7 | NOT_STARTED | — | — |

## Phase 8: Security Hardening (Future / Nice-to-Have)

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 8.1 | BE-8.1 | User auth (Identity Platform + JWT) | GATE-7 | NOT_STARTED | — | — |
| 8.1 | UI-8.1 | Login flow + session handling | BE-8.1, BE-8.3 | NOT_STARTED | — | — |
| 8.2 | BE-8.2 | Snowflake OAuth for MCP | BE-8.1 | NOT_STARTED | — | — |
| 8.3 | BE-8.3 | Multi-tenant isolation | BE-8.1 | NOT_STARTED | — | — |
| 8.3 | AG-8.1 | Per-user job authz in agent tools | BE-8.3, AG-6.2 | NOT_STARTED | — | — |
| 8.4 | BE-8.4 | Rate limiting + CORS tightening | BE-8.1 | NOT_STARTED | — | — |
| 8.5 | GATE-8 | Phase 8 Live Integration Check | All Phase 8 | NOT_STARTED | — | — |

## Phase 9: A2A Externalization (Post-MVP)

| Step | Task | Title | Depends On | Status | Updated | Notes / Evidence |
|------|------|-------|------------|--------|---------|------------------|
| 9.1 | AG-9.1 | Extract transport-agnostic agent module | AG-6.5 | NOT_STARTED | — | — |
| 9.2 | AG-9.2 | A2A server + Agent Card | AG-9.1 | NOT_STARTED | — | — |
| 9.2 | AG-9.3 | tasks/send (synchronous) | AG-9.2 | NOT_STARTED | — | — |
| 9.2 | AG-9.5 | job_id resolution (metadata + text) | AG-9.3 | NOT_STARTED | — | — |
| 9.3 | AG-9.4 | tasks/sendSubscribe (SSE) | AG-9.3 | NOT_STARTED | — | — |
| 9.4 | BE-9.1 | Standalone A2A deploy + bearer auth | AG-9.2 | NOT_STARTED | — | — |
| 9.4 | BE-9.2 | A2A job-ID authz + rate limits | BE-9.1, AG-9.5 | NOT_STARTED | — | — |
| 9.5 | GATE-9 | Phase 9 Live Integration Check | All Phase 9 | NOT_STARTED | — | — |

---

## Worklog (append-only — newest entry LAST; one line per work session)

> Format: `YYYY-MM-DD | <task-id(s)> | <worker> | <what happened / where it stopped / next action>`

| Date | Task(s) | Worker | Summary |
|------|---------|--------|---------|
| — | — | — | Tracker initialized; no work started. |
