# Frontend / UI Backlog (UI-X.Y)

> **Scope**: React SPA — code structure, state management, routing, components, web workers, SSE/streaming clients, and integration hooks against the BFF.
> **Orchestration**: Execution order across components is dictated by [master_backlog.md](./master_backlog.md).
> **Conventions**: `apps/web` — Vite + React + TypeScript + Tailwind CSS + Lucide icons. Server state via TanStack Query; light client state via Zustand. API base URL from `VITE_API_BASE_URL` (the BFF Cloud Run URL). Deployed continuously to Firebase Hosting (infra in BE-1.8).

---

#### Phase 1: The Deployed Walking Skeleton

* **Task UI-1.1: Project Scaffold + Application Shell**
  * **Description:** Initialize Vite + React + TypeScript with Tailwind and Lucide. Build the application shell: top header (product name, environment badge sourced from env), main content area, and a route container (React Router) with a single `/` route. Establish folder structure: `src/components`, `src/features/{jobs,upload,progress,analysis,chat}`, `src/api`, `src/hooks`, `src/workers`, `src/stores`.
  * **Dependencies:** None
  * **Incremental Verification:** `npm run dev` renders the shell locally; `npm run build` produces a clean production bundle with zero TypeScript errors.

* **Task UI-1.2: Typed API Client + Environment Config**
  * **Description:** Build `src/api/client.ts`: a thin typed fetch wrapper (base URL from `VITE_API_BASE_URL`, JSON parsing, normalized `ApiError` with status/message, request-id header). Install TanStack Query with a configured `QueryClient` (sane retry/stale defaults) and a `<QueryClientProvider>` at the root. Define the first response types (`HealthStatus`, `JobRun`) mirroring BFF models.
  * **Dependencies:** UI-1.1
  * **Incremental Verification:** With `VITE_API_BASE_URL` pointed at the deployed dev BFF, a scratch call to `/healthz` from the local dev server succeeds and logs the typed response (also proves CORS from a non-deployed origin if allowed, else verify after deploy).

* **Task UI-1.3: System Status Widget (First Live Slice)**
  * **Description:** A small footer/header widget showing three dots — BFF (`/healthz`), Snowflake (`/api/health/snowflake`), and later Redis (`/api/health/redis`, wired in Phase 3) — green/red with tooltips showing latency and version info, refreshed every 60s via TanStack Query polling. This is the permanent canary that the deployed UI is talking to live cloud services.
  * **Dependencies:** UI-1.2, BE-1.6
  * **Incremental Verification:** On the deployed Firebase URL, both dots are green; temporarily break the Snowflake secret in Cloud Run and the Snowflake dot turns red within a minute.

* **Task UI-1.4: Jobs Table v0 (Read-Only List)**
  * **Description:** On the `/` route, render an unstyled-but-functional table bound to `GET /api/jobs` via a `useJobs()` query hook: columns Job ID (monospace), File Name, Line Code, Status (plain text for now), Created At. Empty state: "No jobs yet — upload your first ACES file." This proves the full UI → BFF → Snowflake read path before any real feature work.
  * **Dependencies:** UI-1.2, BE-1.7
  * **Incremental Verification:** Seed a job via `POST /api/jobs/dev-seed`; it appears in the live deployed table within one refetch without a code change.

* **Task UI-1.5: SPA Continuous Deployment Hook-Up**
  * **Description:** Add the SPA build step to the Cloud Build trigger (BE-1.8): typecheck → build with the dev `VITE_API_BASE_URL` → `firebase deploy` on every `main` push touching `apps/web/**`. Inject the commit SHA into the bundle and surface it in the footer for deploy verification.
  * **Dependencies:** UI-1.1, BE-1.8
  * **Incremental Verification:** Push a visible text change; within minutes the Firebase URL serves it and the footer SHA matches `git rev-parse HEAD`.

---

#### Phase 2: Ingestion & Staging Slice

* **Task UI-2.1: Upload Modal Component (Dropzone + Client Checks)**
  * **Description:** Build the Screen-2 modal per design doc 02 §3: dashed drag-and-drop zone (plus click-to-browse) accepting only `.xml`, immediate client-side rejection of files >150MB or wrong extension with inline error states, selected-file summary row (name, size), and Cancel/Start-Ingestion buttons (Start disabled until preflight passes in UI-2.2). Opened from an "Upload New Fitments File" button on the dashboard.
  * **Dependencies:** UI-1.1
  * **Incremental Verification:** On the deployed site: dropping a `.txt` or a 200MB file shows the correct inline error; dropping a valid XML shows its name/size and an enabled flow path.

* **Task UI-2.2: Client-Side Preflight Web Worker**
  * **Description:** Implement `src/workers/preflight.worker.ts` per design doc 02 §3 Screen 2: read the first ~50KB of the dropped file off the main thread, parse the ACES header (company, transfer date, schema version, line code), estimate record count by sampling `<App` tag density and extrapolating by file size, and detect structural non-conformance (missing `<ACES>`/`<App>`). Render the results as the modal's "Client-Side Preflight Summary" block; allow manual Line Code override when detection fails or mismatches.
  * **Dependencies:** UI-2.1
  * **Incremental Verification:** Dropping the 5K-record sample shows version/line-code/~estimate within ~1s with zero main-thread jank (DevTools performance trace); dropping a renamed `.zip` shows the "not a valid ACES structure" alert.

* **Task UI-2.3: Upload Submission Flow (Streaming POST + 202 Handling)**
  * **Description:** Wire Start Ingestion: POST the file as `multipart/form-data` with the (possibly overridden) `line_code`, render upload progress (XHR/fetch-with-progress percentage bar), handle the 202 response by storing the returned `job_id` and transitioning the modal into the progress experience (placeholder "Processing started" panel until Phase 3's tracker). Handle failure paths: 413 (too large), network error mid-upload (retry affordance — full resumability arrives in UI-7.1).
  * **Dependencies:** UI-2.2, BE-2.2
  * **Incremental Verification:** Upload the 5K sample on the live site: progress bar climbs to 100%, the modal shows the returned `job_id`, and the new job appears in the table. Kill wifi mid-upload of a large file: a clear retry prompt appears (no silent hang).

* **Task UI-2.4: Job History Dashboard v1 (Metrics Bar + Real Table + Polling)**
  * **Description:** Upgrade Screen 1 to design doc 02 §3: primary metrics bar (Total Uploaded Files, Average Pass Rate — shows "—" until Phase 4 data exists, Active Jobs ticker), and the full Job Runs table (Job ID, Transfer Date, File Name, Line Code, Record Count, color-coded Status badges for STAGED/LOADING/VALIDATING/COMPLETED/COMPLETED_WITH_ERRORS/FAILED, Actions column placeholder). While any job is non-terminal, poll `GET /api/jobs` every 5s (TanStack Query `refetchInterval`), stopping when all jobs are terminal.
  * **Dependencies:** UI-1.4, BE-2.8
  * **Incremental Verification:** Upload a file and watch the row's badge advance STAGED → … → COMPLETED purely via polling (no manual refresh); record count appears once staging finishes and matches the Snowflake count.

---

#### Phase 3: Real-Time Progress

* **Task UI-3.1: SSE Client Hook (`useJobEvents`)**
  * **Description:** Build a robust `useJobEvents(jobId)` hook wrapping `EventSource` against `GET /api/jobs/{id}/events`: typed event parsing (`progress`, `metrics_update`, `log`, `complete`), automatic reconnect with exponential backoff, replay handling (server resends last state on connect), automatic close on `complete`, and graceful degradation to the polling endpoint (`/status`, BE-3.4) after repeated SSE failures. Expose `{ currentNode, percentage, message, metrics, logs[], isComplete, transport }`.
  * **Dependencies:** BE-3.3
  * **Incremental Verification:** During a live run, a debug panel rendering the hook's raw state updates in real time; toggling DevTools offline→online shows reconnect + correct state replay; blocking the SSE URL flips `transport` to "polling" and progress continues.

* **Task UI-3.2: Real-Time Progress Tracker Screen (Stepper)**
  * **Description:** Build Screen 3 per design doc 02 §3: the 5-stage Orchestrator Progress Stepper (Parsing & Staging → Schema Validation → Ref Data Checks → Catalog Change Comparison → Finalizing) driven by the node→stage mapping; per-stage states (pending/active-with-spinner/done/failed) with the live message under the active stage (*"Parsed and staged 54,000 / 100,000 records into Snowflake"*); overall percentage bar. Two presentation modes: full-screen focus view (replaces upload modal after submission) and a minimized floating card that persists across navigation (Zustand store holds active-job state).
  * **Dependencies:** UI-3.1, AG-3.1
  * **Incremental Verification:** Upload on the live site: each stage activates in order without refresh, staging counts climb, and minimizing then navigating to the dashboard keeps the floating card live; the card auto-converts to a "View Results" affordance on the `complete` event.

* **Task UI-3.3: Live Error Log Ticker**
  * **Description:** Terminal-style scrollable panel at the bottom of the tracker rendering `log` events: monospace `[HH:MM:SS] SEVERITY: message` lines, color-coded by severity, auto-scroll pinned to bottom with a "pause on hover/scroll-up" interaction, capped buffer (~500 lines) with virtualized rendering.
  * **Dependencies:** UI-3.1, AG-3.3
  * **Incremental Verification:** Running the salted-errors file shows warnings/errors streaming live in the ticker; scrolling up pauses auto-scroll and a "↓ resume" pill appears; 1,000+ events don't degrade frame rate.

* **Task UI-3.4: Dashboard Live Status Integration**
  * **Description:** Replace Phase-2 polling on the dashboard with event-driven updates for jobs the session is tracking: the Active Job ticker in the metrics bar mirrors the floating card, row badges/progress update from the shared store on each event, and the `complete` event invalidates the jobs query for final server truth. Polling remains only as the fallback transport.
  * **Dependencies:** UI-3.2, BE-3.4
  * **Incremental Verification:** With the dashboard open during a run, the row's badge transitions in real time (network tab shows no 5s polling while SSE is healthy) and flips to its terminal badge the instant the run completes.

---

#### Phase 4: Validation Core

* **Task UI-4.1: Live Validation Counters (Ref Data Checks Stage)**
  * **Description:** Enrich stage 3 of the stepper using `metrics_update` events: per-category live counters (VCDB / PCDB / QDB / Product) that increment as each MCP tool reports (*"VCDB base vehicle resolution... 1,420 errors found"*), with a subtle pulse animation on change and a running valid/invalid tally.
  * **Dependencies:** UI-3.2, AG-4.4
  * **Incremental Verification:** Running the salted file shows each category counter land on its known expected value during the run — before the job even finishes.

* **Task UI-4.2: Job Detail Validation Summary Card**
  * **Description:** Create a `/jobs/:jobId` detail route showing the post-run validation summary from `GET /api/jobs/{id}/summary`: pass-rate hero stat (e.g., "98.2% Pass — 102.4k valid"), error counts by category with mini-bars, file metadata (name, line code, ACES version, transfer date), and pipeline duration. Status-aware: renders the tracker (UI-3.2) while running, the summary once terminal.
  * **Dependencies:** UI-3.4, BE-4.10
  * **Incremental Verification:** Navigating to a completed salted job shows category counts identical to the endpoint's JSON (and Snowsight truth); visiting a running job shows the live tracker instead.

* **Task UI-4.3: Failed-Records Drawer v1**
  * **Description:** On the job detail route, a collapsible drawer listing failed records from `GET /api/jobs/{id}/failed-records`: paginated table (Part #, Vehicle ID, Part Type, Position, Errors), an error-type filter chip row (VCDB/PCDB/QDB/PRODUCT/ALL), and multi-error rendering — each record's `errors[]` array shown as stacked typed badges with messages (showcasing the VARIANT error model).
  * **Dependencies:** UI-4.2, BE-4.11
  * **Incremental Verification:** Filtering by QDB on the live salted job shows only QDB-failing records; a record salted with two error types displays both badges; pagination next/prev returns distinct rows.

* **Task UI-4.4: Preflight Warnings Modal (Override / Abort)**
  * **Description:** Handle server-side preflight outcomes surfaced through events/log stream per design doc 02 §8.1: when the pipeline emits a blocking preflight failure (version incompatibility), show a clear failure explanation on the tracker with guidance; for the line-code mismatch case, implement the pre-upload warning in the upload modal — if the worker-detected line code differs from the user's selection, show a Warnings Modal allowing "Use detected", "Keep mine", or "Abort".
  * **Dependencies:** UI-2.2, AG-4.3
  * **Incremental Verification:** Uploading a file whose header line code differs from the selector triggers the modal with both values shown; uploading the wrong-VCDB-version fixture ends with a human-readable version-mismatch explanation on the tracker (not a generic failure).

---

#### Phase 5: Comparison, Statistics & Reports

* **Task UI-5.1: Split-Screen Analysis Layout Scaffold**
  * **Description:** Build Screen 4's frame per design doc 02 §3: route `/jobs/:jobId/analysis` with a 60/40 split — left pane scrollable dashboard region, right pane fixed chat sidebar shell (header "Job Query Assistant — Chatting about Job: {id}", body placeholder until Phase 6). Responsive behavior: panes stack on narrow viewports with a tab switcher. Header carries the job filename + status badge + back-to-dashboard.
  * **Dependencies:** UI-4.2
  * **Incremental Verification:** The deployed route renders the split frame for a completed job with correct job context in both pane headers; resizing to mobile width switches to tabs without layout breakage.

* **Task UI-5.2: Header Summary Card + Uplift Pie Chart**
  * **Description:** Left-pane top section: overall file-health summary card (pass %, valid count, execution time, final status) and the interactive uplift pie (Recharts or similar) for ADD/UPDATE/UNCHANGED/DELETE from `GET /api/jobs/{id}/uplift` — hover tooltips show count + percentage + up to 3 representative part numbers per segment; clicking a segment preselects that classification in the records list (UI-5.4 wiring).
  * **Dependencies:** UI-5.1, BE-5.6
  * **Incremental Verification:** For the A/B catalog scenario job, the four segments show exactly 3/2/4/3 with correct sample part numbers on hover; clicking "DELETE" scrolls/filters the drawer to the 3 deleted fitments.

* **Task UI-5.3: Validation Error Bar Chart + Top Failures Widget**
  * **Description:** Left-pane middle section: horizontal bar chart of failure counts per category (VCDB/PCDB/QDB/Brand/Product) from `GET /api/jobs/{id}/error-distribution`, plus a "Top Failure" widget surfacing the most frequent failing part numbers with occurrence counts (e.g., "BP-2120 (420×)") from the brand/part performance data.
  * **Dependencies:** UI-5.1, BE-5.6
  * **Incremental Verification:** Bars match the summary card's category counts exactly; the top-failure part number matches a hand-run Snowsight `GROUP BY supplier_part_number ORDER BY count DESC LIMIT 1`.

* **Task UI-5.4: Sample Drawer v2 (Spreadsheet + Raw XML Snippet)**
  * **Description:** Upgrade the drawer into the searchable spreadsheet from design doc 02 §3: virtualized grid over failed records *and* classified uplift records (toggle), client-side text search within the loaded page, server-side classification/error-type filters, and row-click detail panel that fetches `GET /api/jobs/{id}/records/{app_id}/source` and renders the syntax-highlighted raw `<App>` XML snippet that produced the record.
  * **Dependencies:** UI-4.3, BE-5.6
  * **Incremental Verification:** Clicking a failed row on the live site shows its original XML with the matching `app_id`; toggling to "UPDATE" classification lists the 2 scenario updates with their multi-facet `change_details` visible.

* **Task UI-5.5: Report Download Integration**
  * **Description:** Add download affordances: a "DL" action icon per completed dashboard row and a "Download Reports" menu on the analysis header, both listing artifacts from `GET /api/jobs/{id}/reports` (Summary JSON / Uplift Excel / Failed CSV) and triggering browser downloads via the signed URLs. Disabled state with tooltip for jobs without artifacts (FAILED before report stage).
  * **Dependencies:** UI-5.1, BE-5.5
  * **Incremental Verification:** All three files download from the live UI and open correctly; a pre-report FAILED job shows the disabled state instead of a broken link.

---

#### Phase 6: Conversational Analytics

* **Task UI-6.1: Chat Sidebar Component (Messages + Markdown)**
  * **Description:** Fill the right pane: scrollable message list with user/assistant bubbles, markdown rendering for assistant messages (react-markdown + GFM for the tables/lists/bold callouts the agent emits, with safe-by-default sanitization), auto-scroll on new content, and the agent's opening greeting generated from the job summary ("I've loaded the validation data. I detected: …").
  * **Dependencies:** UI-5.1
  * **Incremental Verification:** Rendering a fixture assistant message containing a markdown table + bold text displays a properly formatted table (storybook/dev harness check), and the greeting shows real numbers for a live completed job.

* **Task UI-6.2: Streaming Consumption + Activity Indicator**
  * **Description:** Implement the chat transport: `fetch` POST to `/api/jobs/{id}/chat` consuming the chunked body via `ReadableStream`, appending tokens to the active assistant bubble as they arrive; parse interleaved activity markers (AG-6.3) to show the animated *"Assistant is querying Snowflake tables..."* indicator while tools run; handle mid-stream errors with a retry affordance on the failed message; pass the session id header for memory continuity.
  * **Dependencies:** UI-6.1, BE-6.2
  * **Incremental Verification:** On the live site, asking a drill-down question shows: indicator appears → indicator clears → tokens type out progressively (visibly incremental in the UI, not one paste); a follow-up "show me more" continues the same context.

* **Task UI-6.3: Suggestion Chips + Input Box (History + Mic)**
  * **Description:** Composer features per design doc 02 §3: pre-built suggestion chips above the input ("Show the top 3 vehicle resolution failures.", "List the parts that have brand warnings.", "What was the impact on our Ford fitment catalog?") that send on click; input with Enter-to-send/Shift-Enter newline; ArrowUp cycling through this session's prompt history; optional mic input via the Web Speech API where supported (hidden otherwise); send disabled while a response is streaming.
  * **Dependencies:** UI-6.2
  * **Incremental Verification:** Clicking a chip sends that exact prompt and gets a grounded answer; ArrowUp recalls the previous prompt; in Chrome the mic dictates a question into the input.

* **Task UI-6.4: Chat Entry Points (Dashboard Icon + Post-Completion Flow)**
  * **Description:** Wire the two entry paths: a prominent chat icon in each completed/warning dashboard row's Actions column (tooltip: *"Converse with AI about validation issues and statistics for Job {job_id}"*) navigating to `/jobs/:id/analysis`, and the post-pipeline flow where the tracker's completion state offers "Analyze Results" leading to the same screen with chat ready. Hide/disable the icon for running or hard-FAILED jobs (with tooltip explaining why).
  * **Dependencies:** UI-6.2, UI-3.2
  * **Incremental Verification:** From a fresh upload through completion, clicking "Analyze Results" lands on a working split-screen with chat answering about that exact job; a running job's row shows no chat icon.

---

#### Phase 7: Hardening, Performance & Operations

* **Task UI-7.1: Resumable Upload UX**
  * **Description:** Integrate the chunked/resumable protocol (BE-7.3): persist upload session state (job placeholder, offset) in localStorage; on connection drop show "Connection lost — resume upload?" and continue from the stored offset on resume or page reload; show chunk-level progress for large files.
  * **Dependencies:** UI-2.3, BE-7.3
  * **Incremental Verification:** Start a ~120MB upload, kill wifi at ~40%, reconnect: the resume prompt continues from ~40% (network tab confirms no re-sent earlier chunks) and the job completes with correct counts.

* **Task UI-7.2: State Polish + Reconnection UX Sweep**
  * **Description:** Systematic pass over every screen: skeleton loaders for all queries, designed empty states, actionable error states (retry buttons, support hints), an SSE "reconnecting…" banner on the tracker, stale-data indicators when on polling fallback, and a global error boundary with a friendly recovery screen.
  * **Dependencies:** Phases 2–6 UI complete
  * **Incremental Verification:** A scripted walkthrough with DevTools network throttling/offline toggles on each screen shows a designed state (never a blank pane, spinner-forever, or unhandled exception) in every case.

* **Task UI-7.3: Production Build Optimization**
  * **Description:** Route-level code splitting (analysis + chart libs lazy-loaded), bundle analysis with budget enforcement (initial JS < ~250KB gz), chart render memoization for large datasets, Lighthouse pass on the dashboard and analysis routes targeting 90+ performance, and production `VITE_API_BASE_URL` wiring for the prod hosting site (BE-7.5).
  * **Dependencies:** UI-7.2, BE-7.5
  * **Incremental Verification:** Lighthouse on the prod URL scores ≥90 performance on the dashboard; the network tab confirms chart chunks load only when the analysis route is opened.

---

#### Phase 8: Security Hardening (Future / Nice-to-Have)

* **Task UI-8.1: Login Flow + Session Handling**
  * **Description:** Integrate Identity Platform (BE-8.1): login screen (email/password + Google SSO), Firebase ID token attached to every API call by the client wrapper, silent refresh, 401-triggered redirect to login with return-path, user menu with sign-out, and per-user empty states (a new user sees only their jobs per BE-8.3).
  * **Dependencies:** BE-8.1, BE-8.3
  * **Incremental Verification:** Logged out, every route redirects to login; after login as user A, only A's jobs render; expiring the token (clock skew test) silently refreshes without a visible logout.

---

#### Phase 9: A2A Externalization (Post-MVP)

*No frontend tasks — the HITL UI continues using the BFF REST chat endpoint per design doc 05 §3.2. If the team later opts to route the UI through the A2A server, add a follow-up task to swap the chat transport behind the existing `useChat` abstraction (UI-6.2), which requires no component changes.*
