# AI Agent Integration Backlog (AG-X.Y)

> **Scope**: LangGraph orchestration (nodes, state, checkpointing, retry), MCP client scaffolding, LLM client integration, the JobQueryAgent (prompting, tools, memory, streaming), and A2A externalization.
> **Orchestration**: Execution order across components is dictated by [master_backlog.md](./master_backlog.md).
> **Conventions**: All agent code is Python. The LangGraph pipeline lives in `services/orchestrator`; the JobQueryAgent lives in `libs/job_query_agent` (hosted inside the BFF for MVP, extracted in Phase 9). LLM access goes through one shared wrapper (`libs/llm_client`).

---

#### Phase 1: The Deployed Walking Skeleton

* **Task AG-1.1: LLM Client Wrapper (`libs/llm_client`)**
  * **Description:** Build a thin provider wrapper around the chosen LLM API exposing two methods: `complete(messages) -> str` and `stream(messages) -> AsyncIterator[str]`, plus native tool-calling passthrough for later phases. Model name, temperature, and max tokens come from env config; the API key is read from the Secret-Manager-mounted env var (BE-1.6). Include basic retry on transient API errors and token-usage logging per call.
  * **Dependencies:** BE-1.6
  * **Incremental Verification:** A unit script (`python -m llm_client.smoke`) run inside the deployed BFF container (Cloud Run jobs exec or a dev route) returns a completion and logs token counts — proving the key and egress work from Cloud Run, not just locally.

* **Task AG-1.2: Streaming Echo Chat Endpoint (Walking-Skeleton AI Path)**
  * **Description:** Add `POST /api/chat/echo` to the BFF (dev-flag guarded): takes `{message}`, calls `llm_client.stream()` with a trivial system prompt, and returns tokens as a chunked `text/plain` response. This de-risks the entire Phase-6 streaming stack (LLM → FastAPI chunked response → Cloud Run streaming → browser) in week one.
  * **Dependencies:** AG-1.1, BE-1.4
  * **Incremental Verification:** `curl -N -X POST https://aces-bff-dev.../api/chat/echo -d '{"message":"say hi in 3 languages"}'` shows tokens arriving progressively (watch them print one-by-one, not in a single flush).

* **Task AG-1.3: LangGraph Skeleton Graph (Start → FinalStatus)**
  * **Description:** Inside `services/orchestrator`, define a minimal `ACESValidationState` (job_id, status, error_message) and a two-node LangGraph (`StartNode` → `FinalStatusNode`) compiled with an in-memory checkpointer. `StartNode` reads the `JOB_RUN` row; `FinalStatusNode` updates its status to `COMPLETED`. Wire `POST /internal/jobs/{job_id}/run` to execute the graph. This establishes graph structure, state typing, and Snowflake writes from the orchestrator before any real logic exists.
  * **Dependencies:** BE-1.9, BE-1.7
  * **Incremental Verification:** Trigger the orchestrator for a dev-seeded job via the BFF dev route; `JOB_RUN.status` flips to `COMPLETED` in Snowsight and orchestrator logs show both node executions with the job_id.

---

#### Phase 2: Ingestion & Staging Slice

* **Task AG-2.1: Full LangGraph State Schema + Status Writer Helper**
  * **Description:** Expand state to the full `ACESValidationState` from design doc 01 §2.3 (job_id, file_path, line_code, aces_version, total_records, status, schema_valid, preflight_passed, staging_loaded, validation_summary, comparison_summary, uplift_summary, error_message, retry_count). Implement a `transition(job_id, status)` helper that atomically updates `JOB_RUN.status` + `updated_at` and logs the transition — the single funnel every node must use (guarantees BE-3.4's truthfulness contract later).
  * **Dependencies:** AG-1.3
  * **Incremental Verification:** A local graph run over a fake state shows each transition appearing in `JOB_RUN` with monotonically increasing `updated_at`; invalid transitions (e.g., from a terminal state) raise and are logged.

* **Task AG-2.2: StagingLoadNode (GCS Stream → SAX Parse → COPY INTO)**
  * **Description:** Implement the real StagingLoadNode per design doc 01 §3.3: open a streaming read of `gs://aces-uploads-dev/jobs/{job_id}/source.xml`, feed it through `libs/aces_parser` (BE-2.4) computing DFH per record (BE-2.5), and load via `libs/snowflake_loader` (BE-2.6). On success: set `total_records` in state and `JOB_RUN`, mark `staging_loaded=True`, transition to `STAGED`. On parse failure: capture the line number into `error_message` and route to the failure edge.
  * **Dependencies:** AG-2.1, BE-2.4, BE-2.5, BE-2.6, BE-2.1
  * **Incremental Verification:** Run the orchestrator locally against a dev-uploaded 5K file; Snowflake row count for the job equals the file's `<App>` count; orchestrator container memory stays flat during the run (Cloud Run metrics).

* **Task AG-2.3: Failure Edges + FinalStatusNode v1**
  * **Description:** Add conditional edges so any node failure routes to `FinalStatusNode`, which now implements v1 logic: set `FAILED` with `error_message` persisted into `JOB_RUN.error_summary`, or `COMPLETED` on the happy path. Wrap node bodies in a standard exception handler that increments `retry_count` and re-raises for LangGraph retry handling (full retry policy arrives in AG-4.5).
  * **Dependencies:** AG-2.1
  * **Incremental Verification:** Upload a deliberately truncated XML; the job lands in `FAILED` and `error_summary` contains the parser's line-number message; a clean file lands in `COMPLETED`.

* **Task AG-2.4: Graph Wiring v1 + Idempotent Run Endpoint**
  * **Description:** Compile the Phase-2 graph (`START → StagingLoadNode → FinalStatusNode` with failure edges) behind `POST /internal/jobs/{job_id}/run`. Add the duplicate-dispatch guard required by Cloud Tasks' at-least-once delivery: if `JOB_RUN.status` is already terminal or actively running (with a staleness window), return 200 no-op. Loading is idempotent end-to-end because BE-2.6 clears per-job data first.
  * **Dependencies:** AG-2.2, AG-2.3
  * **Incremental Verification:** Invoke the run endpoint twice in quick succession for one job: exactly one load executes (Snowflake count correct, second call logs "duplicate dispatch ignored").

---

#### Phase 3: Real-Time Progress

* **Task AG-3.1: Node Lifecycle Event Decorator**
  * **Description:** Implement a `@publishes_progress(node_name, pct_start, pct_end)` decorator applied to every node: publishes a `progress` event (design doc 01 §5.2 payload) on entry (`IN_PROGRESS`) and exit (`COMPLETED`/`FAILED`) via `libs/events` (BE-3.2), with a human-readable message per node (e.g., "Performing set-based validation across VCDB, PCDB, and QDB..."). Define the canonical node→percentage map for the 5-stage UI stepper in one shared constant.
  * **Dependencies:** BE-3.2, AG-2.4
  * **Incremental Verification:** `curl -N` the SSE endpoint during a run: every node emits an entry and exit event with sensible ascending percentages and the documented JSON shape.

* **Task AG-3.2: Granular Staging Progress Events**
  * **Description:** Use the loader's progress callback (BE-2.6) to publish a `progress` event every batch (10K records): *"Parsed and staged {n} / {total_estimate} records into Snowflake"*, with percentage interpolated within the staging stage's band. Total estimate comes from the client preflight value stored at upload, falling back to "unknown" phrasing.
  * **Dependencies:** AG-3.1
  * **Incremental Verification:** During a 100K-file run, the SSE stream shows ~10 incrementing staging messages with climbing record counts.

* **Task AG-3.3: Warning/Error Log Ticker Events**
  * **Description:** Publish `log`-type events (severity, timestamp, message) for noteworthy conditions: parse warnings, per-category invalid counts as they're discovered, retry attempts, and preflight warnings. These feed the UI's terminal-style ticker (`[17:12:15] ERROR: Vehicle ID 54101 does not exist in VCDB master`). Cap emission rate (e.g., summarize beyond 20 events/stage) to protect the channel.
  * **Dependencies:** AG-3.1
  * **Incremental Verification:** Run a salted-errors file; the SSE stream interleaves `log` events with `progress` events, severities are correct, and a pathological all-bad file doesn't flood the stream (summarization kicks in).

---

#### Phase 4: Validation Core

* **Task AG-4.1: MCP Client Module (`libs/mcp_client`)**
  * **Description:** Implement a Snowflake-Managed-MCP client: JSON-RPC 2.0 over HTTPS to the server URL with the PAT bearer header (BE-4.9), `list_tools()` and `call_tool(name, arguments)` methods, response unwrapping into typed result objects, structured error mapping (auth vs tool vs Snowflake errors), and per-call latency logging. No LangChain dependency required at this layer — keep it a clean transport client the nodes call directly.
  * **Dependencies:** BE-4.9
  * **Incremental Verification:** From the deployed orchestrator (dev route or exec), `list_tools()` returns the 8 Phase-4 tools and `call_tool('validate_part_type_ids_job', {job_id})` returns the same count as the Snowsight `CALL`.

* **Task AG-4.2: SchemaValidationNode (XSD + Header Metadata)**
  * **Description:** Implement per design doc 01 §3.1, positioned **before** StagingLoadNode: stream the first bytes of the GCS source to extract the ACES `version` attribute; fetch the matching XSD from `gs://aces-xsd-dev/{version}/` (BE-2.1); run full lxml XSD validation over the streamed document; parse the header (company, transfer date, submission type, version dates) into state. On failure: copy the file to the archive bucket under `rejected/`, set `FAILED`, halt.
  * **Dependencies:** AG-2.4, BE-2.1
  * **Incremental Verification:** A schema-valid 4.1 file passes and state shows the parsed header fields (visible in logs); a file with an illegal element fails fast with the XSD violation list in `error_summary`, and the original lands in `archive/rejected/`.

* **Task AG-4.3: PreflightNode (Filename, Submission Type, Version Compatibility)**
  * **Description:** Implement per design doc 01 §3.2: construct the expected ACES filename from header fields and compare with the actual; verify submission-type consistency across filename suffix, upload metadata, and XML header; check declared VCDB/QDB/PCDB version dates against the `*_VERSION` tables' `is_current` rows (direct read or MCP); detect Delete actions inside a FULL submission (warning). Classify findings as blocking (halt + precheck report event) vs warnings (annotate state, publish `log` events, continue).
  * **Dependencies:** AG-4.2, BE-4.2
  * **Incremental Verification:** A correctly-named compatible file passes silently; a file declaring an unloaded VCDB version halts with a version-mismatch message in the SSE stream; a misnamed-but-compatible file proceeds with a visible warning log event.

* **Task AG-4.4: ValidationNode with Parallel Fan-Out**
  * **Description:** Implement per design doc 01 §3.4 as a LangGraph fan-out/fan-in: **Stage 1 (parallel branches)** — `validate_base_vehicle_ids_job`, `validate_part_type_ids_job`, `validate_position_ids_job`, `validate_qualifier_ids_job`, `validate_attribute_values_job`; **Stage 2** — `validate_part_type_position_job`; **Stage 3** — `resolve_vehicle_configuration_job`; **Stage 4** — `resolve_products_job(job_id, line_code)`. Each branch calls via `libs/mcp_client`, accumulates `invalid_count`s into `state.validation_summary`, and publishes a `metrics_update` event after each tool returns. Transition `VALIDATING` on entry.
  * **Dependencies:** AG-4.1, BE-4.8
  * **Incremental Verification:** Run the salted-errors file: orchestrator logs show the five Stage-1 calls overlapping in time (parallelism real), SSE `metrics_update` events arrive per tool, and final per-category counts equal the procedure-level truth from BE-4.4–4.7 verifications.

* **Task AG-4.5: Retry Policy + Checkpoint Integration**
  * **Description:** Implement the design's retry strategy (doc 01 §4): wrap MCP calls with 3-attempt exponential backoff (longer timeout on later attempts); configure LangGraph node-level retry so a failed node re-executes from its checkpoint without re-running prior nodes (procedures are idempotent per job_id); after 3 node failures, route to FinalStatusNode with `FAILED`. Publish a `log` event on every retry attempt.
  * **Dependencies:** AG-4.4
  * **Incremental Verification:** Temporarily point the MCP URL at an invalid host for one tool: SSE shows 3 retry log events then a clean `FAILED`; restore the URL and re-trigger the same job — it resumes and completes without re-staging (staging events absent, validation events present).

* **Task AG-4.6: Full Graph Wiring v2 (Schema → Preflight → Staging → Validation)**
  * **Description:** Recompile the production graph in the design's order: `START → SchemaValidationNode → PreflightNode → StagingLoadNode → ValidationNode → FinalStatusNode`, with conditional failure/warning edges per the state machine in design doc 01 §2.2 and status transitions `SCHEMA_VALIDATING → PREFLIGHT_CHECKING → LOADING → VALIDATING`. Update the node→percentage map for the 5-stage stepper.
  * **Dependencies:** AG-4.2, AG-4.3, AG-4.4, AG-4.5
  * **Incremental Verification:** A clean end-to-end upload walks every status in order (verify the sequence in `JOB_RUN` history via SSE capture); each of the three early-exit paths (bad XSD, bad version, corrupt mid-file) terminates at the correct node with the correct status.

---

#### Phase 5: Comparison, Statistics & Reports

* **Task AG-5.1: ComparisonNode**
  * **Description:** Implement per design doc 01 §3.5: transition to `COMPARING`, call `classify_fitment_changes_job(job_id)` then `identify_deleted_fitments_job(job_id)` via MCP, store both result objects in `state.comparison_summary`, and publish progress + a `metrics_update` carrying add/update/unchanged/delete counts.
  * **Dependencies:** AG-4.6, BE-5.4
  * **Incremental Verification:** Run the Phase-5 A/B catalog scenario (BE-5.2): SSE shows the comparison counts and they equal the procedure-level ground truth (3/2/4/3).

* **Task AG-5.2: StatisticsNode**
  * **Description:** Implement per design doc 01 §3.6: transition `COMPUTING_STATS`, call `compute_validation_summary` and `compute_uplift_summary` via MCP, store both in state, and persist the combined object to `JOB_RUN.error_summary` (via `PERSIST_JOB_SUMMARY`, BE-5.3) so the dashboard and chat agent read one canonical summary.
  * **Dependencies:** AG-5.1
  * **Incremental Verification:** After a run, `SELECT error_summary FROM JOB_RUN WHERE job_id=...` contains both summaries and matches what `GET /api/jobs/{id}/summary` returns.

* **Task AG-5.3: ReportWriterNode**
  * **Description:** Implement per design doc 01 §3.7: generate three artifacts from state + targeted Snowflake reads — `summary.json` (full validation + uplift object), `failed_records.csv` (streamed query over invalid rows, written incrementally to handle large counts), and `uplift_report.xlsx` (openpyxl: summary sheet + per-classification sheets, capped sample rows + counts). Upload all three to `gs://aces-reports-dev/{job_id}/` (BE-5.5 layout).
  * **Dependencies:** AG-5.2, BE-2.1
  * **Incremental Verification:** After a salted-file run, all three objects exist in GCS; the xlsx opens with correct totals on the summary sheet; the CSV row count equals the job's invalid-record count.

* **Task AG-5.4: FinalStatusNode v2 (Terminal Logic, Archive, Cleanup, Final Event)**
  * **Description:** Upgrade to the design's final logic (doc 01 §3.8): status = `COMPLETED` (zero errors) / `COMPLETED_WITH_ERRORS` (partial) / `FAILED` (zero valid); move the original XML from `uploads` to `archive/{job_id}/`; remove the job's internal-stage files (staging *table* rows are retained for chat analytics until BE-7.2 retention purges them); publish the final `complete` SSE event with the payload from design doc 02 §6.2 (totals + duration).
  * **Dependencies:** AG-5.2, AG-5.3
  * **Incremental Verification:** A salted run ends `COMPLETED_WITH_ERRORS` with the original gone from uploads and present in archive; the `complete` SSE event carries totals matching the summary endpoint; `LIST @ACES_STAGING_STAGE/{job_id}/` is empty.

---

#### Phase 6: Conversational Analytics (JobQueryAgent)

* **Task AG-6.1: JobQueryAgent Scaffold (System Prompt + Tool Loop)**
  * **Description:** Build the agent in `libs/job_query_agent`: the system prompt template from design doc 02 §5.2 with `{{JOB_ID}}` injection, a tool-calling loop over `libs/llm_client` (LLM proposes tool → execute → feed result back → repeat until final answer, max ~6 tool rounds), and markdown-formatted final answers. Persona: automotive data analyst translating VCDB/PCDB/QDB jargon into plain explanations; hard rule: never invent numbers — only report tool outputs.
  * **Dependencies:** AG-1.1, BE-6.1
  * **Incremental Verification:** A dev harness (`python -m job_query_agent.repl --job <id>`) answers "give me a summary" for a real completed job, and the transcript log shows it called `get_job_summary` and quoted only returned values.

* **Task AG-6.2: Tool Bindings with Server-Forced job_id**
  * **Description:** Bind the five BE-6.1 functions as LLM tool definitions (JSON-schema'd inputs/outputs per design doc 02 §5.3). Critically: `job_id` is **not** an LLM-controllable parameter — the binding layer injects the session's job_id into every call regardless of model output, and clamps `limit` to 50. Tool descriptions teach the model when to use each (summary vs drill-down vs grouping).
  * **Dependencies:** AG-6.1
  * **Incremental Verification:** In the REPL, explicitly ask the agent to query a *different* job_id — transcript shows every executed call still used the session job_id; ask for "all 15,000 failures" — calls are capped at 50 and the answer directs the user to the drawer/export.

* **Task AG-6.3: Streaming Tokens + Tool-Activity Events**
  * **Description:** Make the agent's response an async stream: emit structured interleaved chunks — `activity` markers when tool calls start/finish (powering the UI's "Assistant is querying Snowflake tables..." indicator) and `token` chunks for the final answer text. Expose as `JobQueryAgent.stream_chat(job_id, message, history)` consumed by BE-6.2.
  * **Dependencies:** AG-6.1, AG-6.2
  * **Incremental Verification:** `curl -N` against the chat endpoint shows an activity marker appear *before* tokens, then progressive token delivery; total time-to-first-byte under ~2s on a summary question.

* **Task AG-6.4: Edge-Case Behaviors + Prompt Tuning Pass**
  * **Description:** Implement and verify the design's chat edge cases (doc 02 §8.2): clean-job questions ("This job completed with 100% success!"), huge-result pagination guidance, ambiguous queries ("why is everything broken?" → metrics-first structured summary with top causes + percentages + examples). Build a small golden-question set (10–15 Q/A pairs against fixture jobs) and iterate the system prompt until all pass; check the set into `libs/job_query_agent/evals/`.
  * **Dependencies:** AG-6.3
  * **Incremental Verification:** Run the golden-question script against the live dev environment: every question produces a structurally-correct, numerically-accurate answer (numbers cross-checked against Snowsight by the script).

* **Task AG-6.5: Session Memory (Short-Term Conversation History)**
  * **Description:** Persist per-session message history in Redis (key `chat:{job_id}:{session_id}`, TTL 1h, capped at last ~20 turns with oldest-first truncation) so follow-ups like "show me 5 more of those" resolve correctly. History is injected into the agent's context each turn; tool results are summarized, not replayed verbatim, to control token usage.
  * **Dependencies:** AG-6.3, BE-3.1
  * **Incremental Verification:** In the live UI: ask "top VCDB failures", then "show me 5 more" — the second answer continues the same drill-down (correct offset/context); after 1h idle or a new session, the context is fresh.

---

#### Phase 7: Hardening, Performance & Operations

* **Task AG-7.1: Durable Checkpointing + Resume Capability**
  * **Description:** Replace the in-memory checkpointer with a durable LangGraph checkpointer (Postgres on a minimal Cloud SQL instance, or equivalent supported backend) keyed by `job_id` as `thread_id`, so an orchestrator instance crash or Cloud Tasks retry resumes from the last completed node instead of restarting. Add `POST /internal/jobs/{job_id}/resume` and a runbook entry for operator-initiated resume of `FAILED` jobs.
  * **Dependencies:** AG-5.4
  * **Incremental Verification:** Kill the orchestrator container (scale to zero) mid-ValidationNode on a 100K job; the Cloud Tasks retry resumes the run — staging events do not re-fire, validation completes, and final counts equal a clean control run.

* **Task AG-7.2: Failure Injection & Idempotency Drills**
  * **Description:** Scripted chaos pass over the pipeline: force an MCP timeout mid-fan-out, a Snowflake transient error during COPY INTO, and a Redis outage during event publishing (events must degrade gracefully — pipeline continues, status still lands in `JOB_RUN`). Verify every documented failure mode in design doc 01 §4.2 behaves as specified, and fix anything that double-writes (classification duplicates, repeated error appends).
  * **Dependencies:** AG-7.1
  * **Incremental Verification:** The drill script's checklist passes: every injected failure ends in either successful resume or clean `FAILED`, and post-run data audits show zero duplicate classifications and zero duplicated error-array entries.

* **Task AG-7.3: Concurrent Jobs Validation**
  * **Description:** Confirm multi-job isolation per design doc 01 §6.3: run 3 simultaneous pipelines (mixed sizes), each as its own graph thread; verify per-job event channels don't cross, warehouse queuing is acceptable, and Cloud Run orchestrator concurrency settings (instances × concurrency) are tuned so jobs don't starve.
  * **Dependencies:** AG-7.1
  * **Incremental Verification:** All 3 jobs complete with counts identical to their solo-run baselines; each job's SSE stream contained only its own job_id events; document the measured concurrent-throughput numbers in `infra/performance.md`.

---

#### Phase 8: Security Hardening (Future / Nice-to-Have)

* **Task AG-8.1: Per-User Job Authorization in Agent Tools**
  * **Description:** Thread the authenticated `user_id` (BE-8.1/8.3) into the JobQueryAgent session and enforce job ownership *inside the tool-binding layer* (defense in depth beyond the BFF route guard): any tool call for a job the user doesn't own raises an authorization error the agent reports plainly.
  * **Dependencies:** BE-8.3, AG-6.2
  * **Incremental Verification:** With user A's session token but B's job_id forced into the request, the chat responds with an access-denied message and audit logs show the blocked tool call.

---

#### Phase 9: A2A Externalization (Post-MVP)

* **Task AG-9.1: Extract JobQueryAgent into a Transport-Agnostic Module**
  * **Description:** Refactor `libs/job_query_agent` so it has zero FastAPI/BFF imports: pure interface `stream_chat(job_id, message, history) -> AsyncIterator[AgentChunk]` plus its tool bindings and config. The BFF chat endpoint becomes a thin adapter — proving the same logic can host both REST (MVP) and A2A transports per design doc 05 §3.2.
  * **Dependencies:** Phase 6 complete (AG-6.5)
  * **Incremental Verification:** The existing live chat UI works unchanged after the refactor (regression: golden-question set still passes), and the module imports cleanly in an isolated venv with no web-framework dependencies.

* **Task AG-9.2: A2A Server + Agent Card**
  * **Description:** Build `services/a2a` using the Python `a2a-sdk` (or FastAPI implementing the protocol): serve the Agent Card from design doc 05 §2 verbatim at `/.well-known/agent.json` (4 skills: job-summary, failure-analysis, uplift-details, performance-metrics; streaming=true), and the JSON-RPC method router at `/a2a`.
  * **Dependencies:** AG-9.1
  * **Incremental Verification:** `curl https://<host>/.well-known/agent.json` returns the card and it validates against the A2A Agent Card schema; an unknown JSON-RPC method returns a spec-compliant error.

* **Task AG-9.3: `tasks/send` (Synchronous Task Lifecycle)**
  * **Description:** Implement the synchronous flow per design doc 05 §4.1: accept a task with a user message, run the agent to completion, and return the task in `completed` state with the answer as a text artifact; map agent/tool failures to `failed` state with diagnostic messages. Maintain an in-memory/Redis task store backing `tasks/get`.
  * **Dependencies:** AG-9.2
  * **Incremental Verification:** A `tasks/send` asking for a job summary returns `completed` with an artifact whose numbers match the REST chat answer for the same question; `tasks/get` with the task id replays the result.

* **Task AG-9.4: `tasks/sendSubscribe` (SSE Streaming)**
  * **Description:** Implement the streaming variant per design doc 05 §4.2: SSE responses emitting `status: working`, incremental artifact chunks as the agent streams, and a terminal `status: completed/failed` event.
  * **Dependencies:** AG-9.3
  * **Incremental Verification:** `curl -N` of a `tasks/sendSubscribe` request shows the working event, then progressive artifact chunks, then completed — with chunk text concatenating to the same answer as `tasks/send`.

* **Task AG-9.5: job_id Resolution (Metadata + In-Message)**
  * **Description:** Support both resolution modes from design doc 05 §5: prefer structured `message.metadata.job_id` (injected into the agent's system prompt), fall back to extracting an explicit job_id from the message text; sanitize/validate the id format and return a `failed` task for unknown jobs.
  * **Dependencies:** AG-9.3
  * **Incremental Verification:** Three requests behave correctly: metadata-only (works), text-mention-only (works), neither/garbage id (clean `failed` with explanatory message — no tool calls executed).
