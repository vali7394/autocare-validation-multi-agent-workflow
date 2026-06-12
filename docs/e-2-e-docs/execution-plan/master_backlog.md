# ACES Fitment Validation — Master Execution Backlog (Orchestrator)

> **Purpose**: Master cross-component orchestration timeline. Dictates the exact sequential ordering of tasks across `ui_tasks.md` (UI-X.Y), `backend_tasks.md` (BE-X.Y), and `agent_tasks.md` (AG-X.Y) so that every phase ends with a **deployed, live-verifiable end-to-end slice**.
> **Source Designs**: [00-e2e-design.md](../00-e2e-design.md), [01-multi-agent-orchestration-design.md](../01-multi-agent-orchestration-design.md), [02-hitl-ui-design.md](../02-hitl-ui-design.md), [03-snowflake-mcp-design.md](../03-snowflake-mcp-design.md), [04-review-comments.md](../04-review-comments.md), [05-a2a-agent-design.md](../05-a2a-agent-design.md)

---

## Locked Architectural Decisions (resolving the design docs' Open Items)

| Open Item | Decision | Rationale |
|-----------|----------|-----------|
| LangGraph language | **Python** | Single language across BFF + orchestrator; richest LangGraph/MCP ecosystem |
| BFF framework | **FastAPI** (Python) | Native async streaming (SSE + chunked chat), shares parsing/DFH libs with orchestrator |
| Frontend hosting | **Firebase Hosting** (static SPA) | CDN + instant deploys; SPA calls the BFF Cloud Run URL directly (CORS enabled on BFF) so SSE/chat streaming are never buffered by a CDN rewrite |
| API/Worker hosting | **Cloud Run** — two services: `aces-bff`, `aces-orchestrator` (60-min timeout) | Containerized, scale-to-zero, supports response streaming |
| Async job trigger | **Cloud Tasks** (BFF → orchestrator, OIDC-authenticated) | Reliable at-least-once dispatch; orchestrator nodes are idempotent per `job_id` |
| Redis | **GCP Memorystore** + Serverless VPC Access connector | Managed Pub/Sub backbone for pipeline progress events |
| File/report/XSD storage | **GCS buckets** (`uploads/`, `archive/`, `reports/`, `xsd/`) | Raw XML persisted before processing; resolves XSD-storage and report-delivery gaps from review doc §3.4/§3.5 |
| CI/CD | **Cloud Build** triggers from GitHub (per-service `cloudbuild.yaml`) | Continuous deployment from day one |
| Snowflake auth (MVP) | Service user + key-pair for SQL; **Programmatic Access Token** for MCP REST. OAuth deferred to Phase 8 | Matches design's "OAuth deferred" stance |
| Ingestion split | BFF streams raw file to **GCS** and registers the job; the **orchestrator** runs SchemaValidation → Preflight → StagingLoad (SAX parse + COPY INTO) | Reconciles 00-e2e §9 vs 01-orchestration §2.2; keeps upload requests short on Cloud Run |
| Review-comment fixes | Error fields become VARIANT arrays (review §1.2/§1.3), idempotent delete-identification (§4.4), multi-value `change_details` (§4.3), canonical array ordering (§4.5) | Baked into Phase 4/5 tasks, not deferred |

---

## Phase Map (each phase ships a live increment)

| Phase | Milestone | Live capability unlocked |
|-------|-----------|--------------------------|
| 1 | Deployed Walking Skeleton | UI → BFF → Snowflake round-trip live in cloud; LLM streaming proven; CI/CD active |
| 2 | Ingestion & Staging Slice | Real ACES XML uploaded, parsed, staged into Snowflake; stub pipeline flips job status |
| 3 | Real-Time Progress | Redis + SSE live stepper and log ticker during processing |
| 4 | Validation Core | Reference data + stored procs + MCP server; full validation with live error counters |
| 5 | Comparison, Statistics & Reports | ADD/UPDATE/DELETE/UNCHANGED classification, uplift dashboard, downloadable reports |
| 6 | Conversational Analytics | JobQueryAgent chat over completed jobs, streaming, tool-grounded |
| 7 | Hardening, Performance & Ops | 100K-record SLA, retention, resumable uploads, prod environment |
| 8 | Security Hardening (future) | User auth, Snowflake OAuth, multi-tenancy, rate limiting |
| 9 | A2A Externalization (post-MVP) | JobQueryAgent published as a Google A2A service |

---

## Phase 1: The Deployed Walking Skeleton

*Deploys a thin but complete vertical sliver: a React shell on Firebase Hosting calls a live Cloud Run BFF, which queries a live Snowflake `JOB_RUN` table; an LLM streaming endpoint proves the AI path; a stub orchestrator service proves service-to-service invocation. Everything redeploys automatically from `git push`.*

* **Step 1.1:** Execute `BE-1.1` (GCP project + APIs) and `BE-1.2` (Artifact Registry + service accounts) to establish the cloud foundation.
* **Step 1.2:** Execute `BE-1.3` (monorepo + FastAPI BFF skeleton + Dockerfile), then `BE-1.4` (Cloud Build CI/CD → Cloud Run dev). **First live deploy happens here.**
* **Step 1.3:** Execute `BE-1.5` (Snowflake database/schemas/warehouse/service user) in parallel with Step 1.2.
* **Step 1.4:** Execute `BE-1.6` (Secret Manager + Snowflake connectivity + `/api/health/snowflake`), then `BE-1.7` (`JOB_RUN` table + `GET /api/jobs` + dev-seed endpoint).
* **Step 1.5:** Execute `UI-1.1` (React scaffold + app shell), `UI-1.2` (API client), and `BE-1.8` (Firebase Hosting + CORS on BFF), then `UI-1.5` (SPA CI deploy).
* **Step 1.6:** Execute `UI-1.3` (system status widget) and `UI-1.4` (jobs table v0) to light up the first UI → BFF → Snowflake read path.
* **Step 1.7:** Execute `AG-1.1` (LLM client wrapper) and `AG-1.2` (streaming echo chat endpoint) to prove token streaming through Cloud Run.
* **Step 1.8:** Execute `BE-1.9` (orchestrator Cloud Run service shell + OIDC auth) and `AG-1.3` (LangGraph skeleton graph that flips `JOB_RUN` status).
* **Step 1.9:** **Live Integration Check:** Open the Firebase Hosting URL on a phone (not localhost). The status widget shows BFF *and* Snowflake green. Seed a job via `POST /api/jobs/dev-seed`; it appears in the UI table. `curl` the echo chat endpoint and watch tokens stream. Trigger the orchestrator stub and watch the seeded job's status change to `COMPLETED` in the UI after refresh.

---

## Phase 2: Ingestion & Staging Slice

*A real ACES XML file dragged into the live UI streams to GCS, registers a job, and the orchestrator SAX-parses it (with DFH computation) and bulk-loads `FITMENT_STAGE` via COPY INTO. The dashboard shows real jobs with real record counts; status is visible via polling (SSE arrives in Phase 3).*

* **Step 2.1:** Execute `BE-2.1` (GCS buckets + IAM) and `BE-2.3` (`FITMENT_STAGE`/`FITMENT_CLASSIFICATION` DDL + internal stage + file format) in parallel.
* **Step 2.2:** Execute `BE-2.4` (SAX parser module), `BE-2.5` (DFH utility with Snowflake parity), and `BE-2.6` (JSONL-gzip batch loader + COPY INTO) as local-first shared libraries.
* **Step 2.3:** Execute `BE-2.2` (streaming `POST /api/jobs/upload` → GCS + `JOB_RUN` insert) and deploy.
* **Step 2.4:** Execute `AG-2.1` (LangGraph state schema + status writer), `AG-2.2` (StagingLoadNode), `AG-2.3` (failure edges + FinalStatusNode v1), then `AG-2.4` (graph wiring + run endpoint).
* **Step 2.5:** Execute `BE-2.7` (Cloud Tasks queue + post-upload enqueue) to connect upload → orchestrator, and `BE-2.8` (job detail/list endpoints).
* **Step 2.6:** Execute `UI-2.1` (upload modal), `UI-2.2` (preflight web worker), `UI-2.3` (upload submission flow), and `UI-2.4` (dashboard v1 with polling).
* **Step 2.7:** **Live Integration Check:** Drag a ~5K-record sample ACES XML into the live UI. Preflight summary shows detected version/line code/record estimate. After "Start Ingestion", the job appears as `STAGED` → (poll) → `COMPLETED`. In Snowsight, `SELECT COUNT(*) FROM STAGING.FITMENT_STAGE WHERE job_id = '<id>'` matches the UI's record count, and every row has a populated `distinct_fitment_hash`.

---

## Phase 3: Real-Time Progress (Redis + SSE)

*The polling dashboard becomes a live experience: orchestrator nodes publish progress to Memorystore Redis; the BFF relays via Server-Sent Events; the UI renders the 5-stage stepper and a terminal-style log ticker in real time.*

* **Step 3.1:** Execute `BE-3.1` (Memorystore + VPC connector + wiring both Cloud Run services).
* **Step 3.2:** Execute `BE-3.2` (event schema + publisher library), then `AG-3.1` (node lifecycle event decorator) and `AG-3.2` (granular staging progress events).
* **Step 3.3:** Execute `BE-3.3` (SSE endpoint with replay + heartbeat) and `BE-3.4` (polling fallback + last-event persistence).
* **Step 3.4:** Execute `AG-3.3` (warning/error log ticker events).
* **Step 3.5:** Execute `UI-3.1` (SSE hook), `UI-3.2` (progress stepper screen), `UI-3.3` (log ticker), `UI-3.4` (dashboard live updates).
* **Step 3.6:** **Live Integration Check:** Upload a file in the live UI and do not refresh. The stepper advances node-by-node, the staging stage shows "Parsed and staged X / Y records" climbing, the log ticker scrolls, and the dashboard row flips to its final badge the moment the `complete` event fires. Kill the browser tab mid-run, reopen the job — the stepper resumes at the correct current stage (replay works).

---

## Phase 4: Validation Core

*The pipeline becomes real: VCDB/QDB/PCDB reference data is loaded, set-based validation stored procedures are exposed as a Snowflake Managed MCP server, and the orchestrator's Schema/Preflight/Validation nodes execute them — with live error counters in the UI and a drill-down failed-records drawer.*

* **Step 4.1:** Execute `BE-4.1` (AUTOCARE DDL) then `BE-4.2` (reference data load + version tables).
* **Step 4.2:** Execute `BE-4.3` (VARIANT-array error model — review fix) **before** writing any validation procedure.
* **Step 4.3:** Execute `BE-4.4` (VCDB procs), `BE-4.5` (PCDB procs), `BE-4.6` (QDB proc), `BE-4.7` (catalog PRODUCT/BRAND + product resolution proc) — parallelizable; each verified standalone in Snowsight.
* **Step 4.4:** Execute `BE-4.8` (MCP server with validation tools + grants) and `BE-4.9` (MCP REST auth token).
* **Step 4.5:** Execute `AG-4.1` (MCP client module), then `AG-4.2` (SchemaValidationNode), `AG-4.3` (PreflightNode), `AG-4.4` (ValidationNode with parallel fan-out), `AG-4.5` (retry/backoff), and `AG-4.6` (full graph wiring + `metrics_update` events).
* **Step 4.6:** Execute `BE-4.10` (job summary endpoint) and `BE-4.11` (failed-records endpoint).
* **Step 4.7:** Execute `UI-4.1` (live validation counters), `UI-4.2` (validation summary card), `UI-4.3` (failed-records drawer v1), `UI-4.4` (preflight warnings modal).
* **Step 4.8:** **Live Integration Check:** Upload a sample file salted with known-bad records (invalid base vehicle ID, invalid qualifier, invalid part-type/position pair). Watch live counters increment per category during the run. When complete, the summary card shows the exact expected error counts per category, and the drawer filters to show each salted record with **all** of its errors (multi-error VARIANT array visible). Upload a file with a wrong declared VCDB version — the preflight warning modal appears.

---

## Phase 5: Comparison, Statistics & Reports

*The full business outcome ships: change classification against the existing catalog (ADD/UPDATE/DELETE/UNCHANGED via DFH), uplift and validation summaries, generated report artifacts in GCS, and the left-pane analysis dashboard with charts and the sample drawer.*

* **Step 5.1:** Execute `BE-5.1` (`CATALOG.FITMENT` + seeded snapshot from a prior staged job).
* **Step 5.2:** Execute `BE-5.2` (comparison procs with review fixes) and `BE-5.3` (aggregation procs), each verified standalone.
* **Step 5.3:** Execute `BE-5.4` (MCP server update with comparison/aggregation tools).
* **Step 5.4:** Execute `AG-5.1` (ComparisonNode), `AG-5.2` (StatisticsNode), `AG-5.3` (ReportWriterNode), `AG-5.4` (FinalStatusNode v2 with archive + final status logic).
* **Step 5.5:** Execute `BE-5.5` (report storage + download endpoint) and `BE-5.6` (analysis data endpoints).
* **Step 5.6:** Execute `UI-5.1` (split-screen layout), `UI-5.2` (summary card + uplift pie), `UI-5.3` (error bar chart + top failures), `UI-5.4` (sample drawer v2 + XML snippet), `UI-5.5` (report downloads).
* **Step 5.7:** **Live Integration Check:** Seed the catalog from file A, then upload modified file B (a few new fitments, a few changed qualifiers, a few removed). The analysis screen's pie chart shows the exact expected ADD/UPDATE/DELETE/UNCHANGED counts; clicking a failed row reveals its raw XML snippet; the Excel/CSV/JSON reports download from the live UI and their numbers match the dashboard.

---

## Phase 6: Conversational Analytics (JobQueryAgent)

*The right-pane chat assistant goes live: a job-scoped, read-only, tool-grounded agent that answers questions about completed runs with streaming markdown responses.*

* **Step 6.1:** Execute `BE-6.1` (5 read-only tool queries with 50-row caps) and `BE-6.3` (read-only role segregation).
* **Step 6.2:** Execute `AG-6.1` (agent scaffold + system prompt), `AG-6.2` (tool bindings with server-forced `job_id`), `AG-6.3` (streaming + tool-activity events).
* **Step 6.3:** Execute `BE-6.2` (chat endpoint with chunked streaming + completed-job guard).
* **Step 6.4:** Execute `UI-6.1` (chat sidebar + markdown), `UI-6.2` (stream consumption + "querying Snowflake" indicator), `UI-6.3` (suggestion chips + input history), `UI-6.4` (entry points).
* **Step 6.5:** Execute `AG-6.4` (edge-case behaviors + prompt tuning) and `AG-6.5` (session memory) against real live jobs.
* **Step 6.6:** **Live Integration Check:** On the live site, open chat for the Phase-5 job and ask: *"Why did part BP-2120 fail so many times?"* The activity indicator shows tool calls, the answer streams in with a markdown table, and every number quoted matches a direct Snowsight query. Ask about failures on a 100%-clean job — the agent answers "no failures" without hallucinating. Ask about a different job_id — the agent stays sandboxed to the active job.

---

## Phase 7: Hardening, Performance & Operations

*The system is proven at production scale (100K records), gains durable retry, retention cleanup, resumable uploads, monitoring, and a promoted production environment.*

* **Step 7.1:** Execute `BE-7.1` (100K benchmark + warehouse sizing decision).
* **Step 7.2:** Execute `AG-7.1` (durable checkpointing + resume), then `AG-7.2` (failure injection drills) and `AG-7.3` (concurrent jobs validation).
* **Step 7.3:** Execute `BE-7.2` (staging retention task), `BE-7.3` (resumable upload + >150MB gate), and `UI-7.1` (upload resume UX).
* **Step 7.4:** Execute `BE-7.4` (monitoring + alerting) and `UI-7.2`/`UI-7.3` (UX polish + build optimization).
* **Step 7.5:** Execute `BE-7.5` (production environment + tag-based promotion pipeline).
* **Step 7.6:** **Live Integration Check:** Upload a realistic 100K-record / ~120MB file to the **production** URL: it completes within the agreed SLA; kill the orchestrator instance mid-validation and confirm the job resumes from checkpoint and finishes with correct counts; run two jobs concurrently and verify zero cross-contamination; confirm an alert fires when a job is forced to fail.

---

## Phase 8: Security Hardening (Future / Nice-to-Have)

*Per the design's deferred-security stance: user login, Snowflake OAuth for MCP, tenancy, and abuse protection.*

* **Step 8.1:** Execute `BE-8.1` (user authentication via Identity Platform + JWT middleware) and `UI-8.1` (login/session UX).
* **Step 8.2:** Execute `BE-8.2` (Snowflake OAuth integration for MCP, replacing PAT).
* **Step 8.3:** Execute `BE-8.3` (multi-tenant `user_id` on `JOB_RUN` + BFF enforcement) and `AG-8.1` (per-user job authorization in agent tools).
* **Step 8.4:** Execute `BE-8.4` (rate limiting + CORS tightening).
* **Step 8.5:** **Live Integration Check:** An unauthenticated request to any `/api/*` endpoint returns 401; user A cannot read or chat about user B's job (403); MCP REST calls with the old PAT fail while OAuth-token calls succeed; hammering the chat endpoint triggers 429s.

---

## Phase 9: A2A Agent Externalization (Post-MVP)

*The JobQueryAgent becomes a standalone, discoverable Google A2A service per [05-a2a-agent-design.md](../05-a2a-agent-design.md).*

* **Step 9.1:** Execute `AG-9.1` (extract agent into a transport-agnostic module — BFF chat keeps working).
* **Step 9.2:** Execute `AG-9.2` (A2A server + Agent Card at `/.well-known/agent.json`), then `AG-9.3` (`tasks/send`) and `AG-9.5` (job_id metadata resolution).
* **Step 9.3:** Execute `AG-9.4` (`tasks/sendSubscribe` SSE streaming).
* **Step 9.4:** Execute `BE-9.1` (standalone Cloud Run deploy + bearer-token auth) and `BE-9.2` (job-ID authorization + rate limits).
* **Step 9.5:** **Live Integration Check:** From an external A2A reference client: fetch the Agent Card, send a `tasks/send` request with `job_id` in metadata, receive a `completed` task with a correct artifact; repeat via `tasks/sendSubscribe` and observe `working` → artifact chunks → `completed` over SSE; a request without a bearer token is rejected.

---

## Working Agreements

1. **Deploy on merge.** Every task that touches a service ends with the change live in the dev environment via Cloud Build — no local-only "done".
2. **Verify before proceeding.** Each task's *Incremental Verification* note (in the component files) must pass before its dependents start.
3. **Idempotency is non-negotiable.** Any procedure or node must be safely re-runnable per `job_id` (Cloud Tasks is at-least-once).
4. **Small sample files first.** Keep a library of test ACES XML files (clean 100-record, salted-errors 5K-record, realistic 100K-record) in `uploads-test/`; phases 2–6 verify with the small/medium files, Phase 7 with the large one.
5. **Review-comment fixes are in-scope** (VARIANT error arrays, idempotent delete identification, multi-value change details, canonical array ordering) — they are embedded in Phase 4/5 tasks, not a cleanup afterthought.
