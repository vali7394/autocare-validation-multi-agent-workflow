# Backend & Cloud Infrastructure Backlog (BE-X.Y)

> **Scope**: BFF APIs, Snowflake (schemas, procedures, MCP server), GCP infrastructure (Cloud Run, Cloud Build, GCS, Cloud Tasks, Memorystore, Secret Manager, IAM), networking, and CI/CD.
> **Orchestration**: Execution order across components is dictated by [master_backlog.md](./master_backlog.md).
> **Conventions**: Repo layout is a monorepo — `apps/web` (React), `services/bff` (FastAPI), `services/orchestrator` (LangGraph), `libs/` (shared Python: parsing, DFH, Snowflake, events), `snowflake/` (versioned DDL + procedure scripts), `infra/` (deployment configs). Dev environment first; production is created in Phase 7.

---

#### Phase 1: The Deployed Walking Skeleton

* **Task BE-1.1: GCP Project Bootstrap & API Enablement**
  * **Description:** Create the GCP project (e.g., `aces-validation-dev`), link billing, and enable required APIs: Cloud Run, Cloud Build, Artifact Registry, Secret Manager, Cloud Tasks, Cloud Storage, Serverless VPC Access, Memorystore for Redis, Cloud Logging/Monitoring, Firebase. Set default region (e.g., `us-central1`) and document it in `infra/README.md` as the single region for all resources.
  * **Dependencies:** None
  * **Incremental Verification:** `gcloud services list --enabled` shows all listed APIs; `gcloud config get-value project` returns the new project ID.

* **Task BE-1.2: Artifact Registry, Service Accounts & Baseline IAM**
  * **Description:** Create a Docker Artifact Registry repo (`aces-images`). Create three service accounts with least privilege: `sa-cloudbuild` (deployer: Cloud Run admin, Artifact Registry writer, SA user), `sa-bff` (runtime: Secret Manager accessor, GCS object admin on app buckets, Cloud Tasks enqueuer, Redis access), `sa-orchestrator` (runtime: Secret Manager accessor, GCS reader/writer, Redis access). Grant `sa-bff` the `roles/run.invoker` on the orchestrator service (added after BE-1.9 exists; record the binding in `infra/iam.md`).
  * **Dependencies:** BE-1.1
  * **Incremental Verification:** `gcloud artifacts repositories describe aces-images` succeeds; `gcloud iam service-accounts list` shows all three SAs; a test `docker push` of a hello image to the registry succeeds.

* **Task BE-1.3: Monorepo Scaffold + FastAPI BFF Skeleton + Dockerfile**
  * **Description:** Initialize the monorepo structure (`apps/web`, `services/bff`, `services/orchestrator`, `libs`, `snowflake`, `infra`, `uploads-test`). In `services/bff`, scaffold FastAPI with: `GET /healthz` (returns build SHA + timestamp), structured JSON logging, settings module reading env vars (12-factor), and a multi-stage Dockerfile (slim Python base, non-root user, uvicorn entrypoint honoring `$PORT`).
  * **Dependencies:** None (parallel with BE-1.1)
  * **Incremental Verification:** `docker build` + `docker run -p 8080:8080` locally, then `curl localhost:8080/healthz` returns 200 with the git SHA.

* **Task BE-1.4: Cloud Build CI/CD Pipeline → Cloud Run (dev)**
  * **Description:** Author `services/bff/cloudbuild.yaml`: build image, push to Artifact Registry tagged with commit SHA, deploy to Cloud Run service `aces-bff-dev` (min instances 0, max 5, 512Mi, `sa-bff` runtime SA, unauthenticated ingress for MVP). Create a Cloud Build GitHub trigger on pushes to `main` filtered to `services/bff/**` and `libs/**` paths. Repeat trigger pattern later for the other services.
  * **Dependencies:** BE-1.2, BE-1.3
  * **Incremental Verification:** Push a trivial change to `main`; Cloud Build run goes green; `curl https://aces-bff-dev-<hash>.run.app/healthz` returns the new commit SHA.

* **Task BE-1.5: Snowflake Foundation (Database, Schemas, Warehouse, Service User)**
  * **Description:** Run versioned scripts in `snowflake/00_foundation/`: create `ACES_VALIDATION` DB with `AUTOCARE`, `CATALOG`, `STAGING`, `MCP` schemas; create `ACES_VALIDATION_WH` (SMALL for dev, `AUTO_SUSPEND=60`, `AUTO_RESUME=TRUE`); create roles `ACES_MCP_USER` and `ACES_DATA_ADMIN`; create service user `SVC_ACES_APP` with RSA key-pair auth, `DEFAULT_ROLE=ACES_MCP_USER`, `DEFAULT_WAREHOUSE=ACES_VALIDATION_WH`; apply the grants from design doc 03 §8.
  * **Dependencies:** None
  * **Incremental Verification:** From `snowsql`/Snowsight as `SVC_ACES_APP` (key-pair): `SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE();` returns the expected values and `SHOW SCHEMAS IN DATABASE ACES_VALIDATION;` lists all four schemas.

* **Task BE-1.6: Secret Manager Wiring + Snowflake Connectivity Module**
  * **Description:** Store the Snowflake private key, account identifier, and (placeholder) LLM API key in Secret Manager; mount them as env vars/volumes in the Cloud Run deploy config. In `libs/snowflake_client`, build a connection module (snowflake-connector-python, key-pair auth, connection pooling, query helper with parameter binding). Add `GET /api/health/snowflake` to the BFF executing `SELECT CURRENT_VERSION()`.
  * **Dependencies:** BE-1.4, BE-1.5
  * **Incremental Verification:** `curl https://aces-bff-dev.../api/health/snowflake` returns 200 with the Snowflake version string — proving Cloud Run → Secret Manager → Snowflake works live.

* **Task BE-1.7: JOB_RUN Table + Jobs Read API + Dev-Seed Endpoint**
  * **Description:** Create `STAGING.JOB_RUN` per design doc 03 §4 (job_id, status, file_path, line_code, total_records, error_summary VARIANT, timestamps). In the BFF implement `GET /api/jobs` (ordered by created_at desc, limit 100, mapped to a typed response model) and a dev-only `POST /api/jobs/dev-seed` (inserts a synthetic row with a ULID `job_id`; guarded by an `ENABLE_DEV_ENDPOINTS` env flag).
  * **Dependencies:** BE-1.6
  * **Incremental Verification:** `curl -X POST .../api/jobs/dev-seed` then `curl .../api/jobs` returns the seeded job; the same row is visible in Snowsight.

* **Task BE-1.8: Firebase Hosting Site + CORS Middleware on BFF**
  * **Description:** Initialize Firebase Hosting for the SPA (`apps/web`), with a dedicated dev site. Add a Cloud Build trigger that builds the SPA (`VITE_API_BASE_URL` pointing at the `aces-bff-dev` Cloud Run URL) and runs `firebase deploy`. On the BFF, add CORS middleware allowing the Firebase Hosting origin(s) with credentials disabled, all standard methods, and exposed streaming headers. (SPA calls Cloud Run directly so SSE/chat streaming are never CDN-buffered — addresses review doc §3.1.)
  * **Dependencies:** BE-1.4
  * **Incremental Verification:** From the deployed Firebase URL's browser console, `fetch(API_BASE + '/healthz')` succeeds with no CORS error.

* **Task BE-1.9: Orchestrator Cloud Run Service Shell + Service-to-Service Auth**
  * **Description:** Scaffold `services/orchestrator` (FastAPI shell hosting LangGraph later): `GET /healthz` and stub `POST /internal/jobs/{job_id}/run`. Dockerfile + `cloudbuild.yaml` + trigger deploying `aces-orchestrator-dev` with: request timeout 3600s, concurrency 1–4, 1Gi memory, **authenticated ingress only** (no public invoker). Grant `sa-bff` `roles/run.invoker`; in the BFF add an internal HTTP client that mints an OIDC identity token for direct orchestrator calls (Cloud Tasks takes over in Phase 2).
  * **Dependencies:** BE-1.4
  * **Incremental Verification:** Unauthenticated `curl` to the orchestrator URL returns 403; invoking it through a temporary BFF dev route (`POST /api/jobs/{id}/dev-run`) returns 200 — proving OIDC service-to-service auth.

---

#### Phase 2: Ingestion & Staging Slice

* **Task BE-2.1: GCS Buckets + Lifecycle + IAM**
  * **Description:** Create buckets: `aces-uploads-dev` (raw incoming XML, path `jobs/{job_id}/source.xml`), `aces-archive-dev` (processed originals, 180-day lifecycle to coldline), `aces-reports-dev` (generated artifacts), `aces-xsd-dev` (ACES 3.x/4.x XSD files, uploaded now — resolves review doc §3.4). Grant `sa-bff` write on uploads, `sa-orchestrator` read on uploads/xsd and write on archive/reports.
  * **Dependencies:** BE-1.2
  * **Incremental Verification:** `gsutil ls` shows all four buckets; impersonating `sa-orchestrator`, copying a file into `aces-uploads-dev` is denied while reading succeeds (IAM correct); the 4.1 XSD object exists under `aces-xsd-dev/4.1/`.

* **Task BE-2.2: Streaming Upload Endpoint (`POST /api/jobs/upload`)**
  * **Description:** Implement multipart streaming upload in the BFF: stream the request body chunk-by-chunk to `gs://aces-uploads-dev/jobs/{job_id}/source.xml` (resumable GCS write, never buffering the whole file; memory stays <100MB), enforce `.xml` extension and a 150MB hard limit (`413` beyond), accept `line_code` form field, generate a ULID `job_id`, insert `JOB_RUN` with status `RECEIVED`, and return the 202 payload defined in design doc 02 §6.1.
  * **Dependencies:** BE-1.7, BE-2.1
  * **Incremental Verification:** `curl -F file=@sample_5k.xml -F line_code=BP .../api/jobs/upload` returns 202 with a `job_id`; the object exists in GCS at the expected path and a `RECEIVED` row exists in `JOB_RUN`. A 200MB dummy file returns 413.

* **Task BE-2.3: Staging Tables + Internal Stage + File Format**
  * **Description:** Create `STAGING.FITMENT_STAGE` and `STAGING.FITMENT_CLASSIFICATION` per design doc 03 §4 (including PKs on `(job_id, distinct_fitment_hash)`), the internal stage `@ACES_VALIDATION.STAGING.ACES_STAGING_STAGE`, and a named file format (`TYPE=JSON COMPRESSION=GZIP`). Add clustering by `job_id` on `FITMENT_STAGE`.
  * **Dependencies:** BE-1.5
  * **Incremental Verification:** In Snowsight: `PUT` a hand-made 3-record gzip JSONL to the stage, run `COPY INTO FITMENT_STAGE ... MATCH_BY_COLUMN_NAME=CASE_INSENSITIVE`, and confirm 3 rows land with correct column mapping.

* **Task BE-2.4: Streaming SAX Parser Module (`libs/aces_parser`)**
  * **Description:** Build an event-driven parser (lxml `iterparse`) that consumes an XML byte stream and yields one flattened dict per `<App>` block — `app_id, base_vehicle_id, part_type_id, position_id, supplier_part_number, qualifiers[], notes[], vehicle_conditions{}` — clearing elements as it goes (constant memory). Canonically sort `qualifiers` and `notes` arrays and `vehicle_conditions` keys during flattening (review doc §4.5 fix). Also expose `parse_header()` returning company/transfer-date/version/line-code metadata. Surface malformed-XML errors with line numbers.
  * **Dependencies:** None (pure library; parallel with BE-2.1–2.3)
  * **Incremental Verification:** Run the module's CLI (`python -m aces_parser sample_5k.xml --count`) — it prints the exact `<App>` count while RSS memory stays flat (observe via `/usr/bin/time -l`); feeding a truncated file reports the failure line number.

* **Task BE-2.5: Distinct Fitment Hash (DFH) Utility with Snowflake Parity**
  * **Description:** Implement the DFH exactly per design doc 00 §5.5 in `libs/aces_parser/dfh.py`: `MD5(CONCAT_WS('|', product_id|'', base_vehicle_id, part_type_id, position_id|'0', HASH(conditions), HASH(qualifiers), HASH(notes)))` over the canonically-sorted structures. Because Python must equal any future Snowflake-side recomputation, define the sub-hash function deterministically (e.g., SHA-256 of canonical JSON, decimal-encoded) and create a matching Snowflake SQL UDF `MCP.COMPUTE_DFH(...)` so both sides share one formula.
  * **Dependencies:** BE-2.4
  * **Incremental Verification:** A parity script hashes 20 fixture records in Python, inserts the raw fields into a temp Snowflake table, calls the UDF, and asserts all 20 hashes match byte-for-byte.

* **Task BE-2.6: Snowflake Batch Loader (`libs/snowflake_loader`)**
  * **Description:** Build the loader used by the orchestrator's StagingLoadNode: accumulate parsed records into 10K-record gzip JSONL batches, `PUT` each to `@ACES_STAGING_STAGE/{job_id}/batch_NNN.jsonl.gz`, then issue one `COPY INTO STAGING.FITMENT_STAGE`. Make it idempotent per job: `REMOVE @stage/{job_id}/` and `DELETE FROM FITMENT_STAGE WHERE job_id=...` before loading. Expose a progress callback (records staged so far) for event publishing later.
  * **Dependencies:** BE-2.3, BE-2.5
  * **Incremental Verification:** Run the loader CLI against `sample_5k.xml` with a test job_id; `SELECT COUNT(*)` in Snowflake equals the parser's count; run it twice and the count is unchanged (idempotent).

* **Task BE-2.7: Cloud Tasks Queue + Post-Upload Orchestrator Dispatch**
  * **Description:** Create Cloud Tasks queue `aces-jobs` (max attempts 3, min backoff 30s). After a successful upload, the BFF enqueues an HTTP task targeting `POST /internal/jobs/{job_id}/run` on the orchestrator with OIDC token (`sa-bff` identity). Remove/disable the Phase-1 direct dev-run path. Document the at-least-once contract: the orchestrator must no-op if the job is already running/terminal (guard via `JOB_RUN.status`).
  * **Dependencies:** BE-2.2, BE-1.9, AG-2.4
  * **Incremental Verification:** Upload a file; within seconds the Cloud Tasks console shows the task dispatched and succeeded, and `JOB_RUN.status` progresses without any manual trigger. Re-enqueue the same job_id manually — orchestrator logs show the duplicate-run guard firing.

* **Task BE-2.8: Job Detail & Enriched List Endpoints**
  * **Description:** Implement `GET /api/jobs/{job_id}` (full `JOB_RUN` row: status, filename, line_code, total_records, timestamps, error_summary) and enrich `GET /api/jobs` with the dashboard's columns plus aggregate metrics (total jobs, average pass rate once available, currently-active count). Return 404 on unknown job_id.
  * **Dependencies:** BE-2.2
  * **Incremental Verification:** `curl .../api/jobs/{id}` for a staged job returns the correct `total_records` matching Snowflake; unknown ID returns 404 JSON error.

---

#### Phase 3: Real-Time Progress (Redis + SSE)

* **Task BE-3.1: Memorystore Redis + Serverless VPC Access Wiring**
  * **Description:** Create a Memorystore Redis (Basic tier, 1GB) instance and a Serverless VPC Access connector in the same region; attach the connector to both `aces-bff-dev` and `aces-orchestrator-dev` Cloud Run services; inject `REDIS_HOST/PORT` env vars. Add `GET /api/health/redis` (PING) to the BFF. (Resolves review doc §3.2 — Redis as an explicit infra dependency.)
  * **Dependencies:** BE-1.4, BE-1.9
  * **Incremental Verification:** `curl .../api/health/redis` returns 200/PONG from the deployed BFF; the orchestrator logs a successful Redis PING on startup.

* **Task BE-3.2: Event Schema + Publisher/Subscriber Library (`libs/events`)**
  * **Description:** Define the canonical event payload from design doc 01 §5.2 (`job_id, node, status, message, percentage, timestamp`) plus event types `progress`, `metrics_update`, `log`, `complete` as typed models. Implement `publish(job_id, event)` → Redis channel `job:{job_id}:events`, and an async subscriber generator for the BFF. Include a per-job "last event" key (`SET job:{id}:last`) for replay.
  * **Dependencies:** BE-3.1
  * **Incremental Verification:** From a dev shell, run the library's demo publisher while `redis-cli psubscribe 'job:*'` (via a temporary VM or `kubectl`-less tunnel — or a dev-only BFF echo endpoint) shows correctly-shaped JSON events.

* **Task BE-3.3: SSE Endpoint (`GET /api/jobs/{job_id}/events`)**
  * **Description:** Implement the SSE stream in FastAPI: correct headers (`text/event-stream`, `no-cache`, `keep-alive`), named events per type, 15s heartbeat comments to defeat idle timeouts, and on-connect replay of the job's last event + current `JOB_RUN.status` so late subscribers see correct state. Close the stream after the `complete` event. Confirm Cloud Run HTTP streaming is enabled for the service.
  * **Dependencies:** BE-3.2
  * **Incremental Verification:** `curl -N .../api/jobs/{id}/events` during a live upload prints events in real time as the pipeline runs; connecting *after* completion immediately yields the replayed terminal event.

* **Task BE-3.4: Polling Fallback + Status Persistence Contract**
  * **Description:** Ensure every status transition written by the orchestrator persists to `JOB_RUN` (not just Redis) so `GET /api/jobs/{job_id}` is always truthful if SSE drops; add a lightweight `GET /api/jobs/{job_id}/status` (status + percentage + last message) optimized for polling fallback.
  * **Dependencies:** BE-3.3, AG-3.1
  * **Incremental Verification:** Block the SSE connection (kill `curl -N`), poll `/status` mid-run — values keep advancing and match the final SSE-observed state.

---

#### Phase 4: Validation Core

* **Task BE-4.1: AUTOCARE Reference Table DDL**
  * **Description:** Create all reference tables from design doc 03 §2 in `AUTOCARE`: `VCDB_BASE_VEHICLE`, `VCDB_VEHICLE`, attribute domain tables (DriveType, FuelType, EngineBase + the extended set: TransmissionType, BrakeSystem, BodyType, BedType, WheelBase, MfrBodyCode, SteeringType, SteeringSystem, SpringType, Aspiration…), `QDB_QUALIFIER`, `PCDB_PART_TYPE`, `PCDB_POSITION`, `PCDB_PART_TYPE_POSITION`, and the three `*_VERSION` tables. Scripts versioned in `snowflake/01_autocare/`.
  * **Dependencies:** BE-1.5
  * **Incremental Verification:** `SHOW TABLES IN SCHEMA ACES_VALIDATION.AUTOCARE;` lists every table; `DESCRIBE` of `VCDB_VEHICLE` shows all attribute FK columns.

* **Task BE-4.2: Reference Data Load Pipeline (VCDB/QDB/PCDB)**
  * **Description:** Build a repeatable load path for Autocare-delivered reference extracts: land source files in a GCS `aces-refdata-dev` bucket (or Snowflake stage), `COPY INTO` each AUTOCARE table, and upsert the corresponding `*_VERSION` row with `is_current=TRUE` (flipping the previous current to FALSE). For dev, also produce a deterministic synthetic subset (`snowflake/01_autocare/seed_dev.sql`) so validation behavior is fully predictable in tests.
  * **Dependencies:** BE-4.1
  * **Incremental Verification:** Row counts per table match the source extract manifest; `SELECT * FROM VCDB_VERSION WHERE is_current` returns exactly one row; a known base_vehicle_id from the seed resolves via ad-hoc `SELECT`.

* **Task BE-4.3: Error Model Upgrade — VARIANT Arrays (Review Fixes §1.2/§1.3)**
  * **Description:** Alter `FITMENT_STAGE`: replace scalar `error_type`/`error_message` with `errors VARIANT` (JSON array of `{type, message}` objects), retaining `is_valid BOOLEAN`. Establish the canonical proc pattern: `errors = ARRAY_APPEND(COALESCE(errors, ARRAY_CONSTRUCT()), OBJECT_CONSTRUCT('type','VCDB','message','...'))` and — per review §1.3 Option 1 — validation procedures do **not** filter on `is_valid=TRUE` for independent existence checks (all categories are captured per record); only dependent checks (vehicle resolution) skip already-failed records. Document the decision in `snowflake/README.md`.
  * **Dependencies:** BE-2.3
  * **Incremental Verification:** Manually run two different error-append UPDATEs against one staged test row; `SELECT errors` shows a 2-element JSON array with both `{type,message}` entries preserved.

* **Task BE-4.4: VCDB Validation Stored Procedures**
  * **Description:** Implement in `MCP` schema per design doc 03 §5.1, adapted to the VARIANT error model: `VALIDATE_BASE_VEHICLE_IDS_JOB(job_id)`, `VALIDATE_ATTRIBUTE_VALUES_JOB(job_id)` (extended beyond DriveType/FuelType to the full attribute matrix per review §4.1 — driven by a small mapping table `MCP.ATTRIBUTE_DOMAIN_MAP(attr_key, domain_table, domain_id_col)` iterated dynamically), and `RESOLVE_VEHICLE_CONFIGURATION_JOB(job_id)` (full attribute join coverage per review §4.2; runs only on rows with no errors so far). Each returns `{job_id, invalid_count}` and is idempotent.
  * **Dependencies:** BE-4.2, BE-4.3
  * **Incremental Verification:** Stage a hand-crafted 10-row test job (3 bad base vehicles, 2 bad drive types, 1 unresolvable config); `CALL` each proc in Snowsight and confirm the returned `invalid_count` and the per-row `errors` arrays match expectations exactly; `CALL` twice — results unchanged.

* **Task BE-4.5: PCDB Validation Stored Procedures**
  * **Description:** Implement `VALIDATE_PART_TYPE_IDS_JOB`, `VALIDATE_POSITION_IDS_JOB`, and `VALIDATE_PART_TYPE_POSITION_JOB` per design doc 03 §5.3 on the VARIANT error model (position check skips NULL positions; mapping check runs only when both IDs individually exist to avoid double-reporting).
  * **Dependencies:** BE-4.3
  * **Incremental Verification:** Extend the 10-row test job with a bad part type, a bad position, and a valid-IDs-but-invalid-pair record; each proc reports the right count and the pair-record carries only the mapping error.

* **Task BE-4.6: QDB Validation Stored Procedure**
  * **Description:** Implement `VALIDATE_QUALIFIER_IDS_JOB(job_id)` per design doc 03 §5.2: `FLATTEN` the `qualifiers` VARIANT array, anti-join against `QDB_QUALIFIER`, append a QDB error naming the offending qualifier ID(s) in the message.
  * **Dependencies:** BE-4.3
  * **Incremental Verification:** A test record with qualifiers `[valid, invalid]` gets exactly one QDB error whose message contains the invalid ID; a record with all-valid qualifiers is untouched.

* **Task BE-4.7: Catalog PRODUCT/BRAND Tables + Product Resolution Procedure**
  * **Description:** Create `CATALOG.PRODUCT`, `CATALOG.BRAND`, `CATALOG.BRAND_SUB_BRAND` per design doc 03 §3 and seed dev products matching the test files' line codes/part numbers. Implement `RESOLVE_PRODUCTS_JOB(job_id, line_code)`: UPDATE-join to set `product_id`, then append a PRODUCT error for unresolved rows; returns `{resolved_count, unresolved_count}`.
  * **Dependencies:** BE-4.3
  * **Incremental Verification:** `CALL` against the test job: known part numbers receive `product_id`, the deliberately-unknown one carries an "Unresolved Supplier Part Number" error; counts in the return object reconcile with `SELECT` totals.

* **Task BE-4.8: MCP Server Object — Validation & Resolution Tools**
  * **Description:** Create `ACES_VALIDATION.MCP.ACES_FITMENT_SERVER` per design doc 03 §6 with the 8 validation/resolution tools (`validate_base_vehicle_ids_job`, `validate_attribute_values_job`, `resolve_vehicle_configuration_job`, `validate_qualifier_ids_job`, `validate_part_type_ids_job`, `validate_position_ids_job`, `validate_part_type_position_job`, `resolve_products_job`), all GENERIC type backed by the procedures, warehouse `ACES_VALIDATION_WH`. Apply grants (`USAGE ON MCP SERVER`, procedure usage) to `ACES_MCP_USER`.
  * **Dependencies:** BE-4.4, BE-4.5, BE-4.6, BE-4.7
  * **Incremental Verification:** `SHOW MCP SERVERS` + `DESCRIBE MCP SERVER` list all 8 tools; a raw REST `tools/list` JSON-RPC call returns the tool schemas.

* **Task BE-4.9: MCP REST Authentication (PAT) + Secret Distribution**
  * **Description:** Since OAuth is deferred, mint a Snowflake Programmatic Access Token for `SVC_ACES_APP`, store it in Secret Manager, and expose it to the orchestrator as `SNOWFLAKE_MCP_TOKEN` along with the MCP server URL (`https://<account>/api/v2/databases/ACES_VALIDATION/schemas/MCP/mcp-servers/ACES_FITMENT_SERVER`). Document token rotation procedure in `infra/secrets.md`.
  * **Dependencies:** BE-4.8
  * **Incremental Verification:** A `curl` JSON-RPC `tools/call` of `validate_part_type_ids_job` with the PAT against the staged test job returns the same `invalid_count` observed in BE-4.5.

* **Task BE-4.10: Job Validation Summary Endpoint**
  * **Description:** Implement `GET /api/jobs/{job_id}/summary` in the BFF returning the validation summary (total/valid/error counts, errors by category VCDB/QDB/PCDB/PRODUCT) — read from `JOB_RUN.error_summary` when persisted, falling back to calling `COMPUTE_VALIDATION_SUMMARY` live (the proc itself ships in BE-5.3; until then compute via direct aggregate SQL in the endpoint).
  * **Dependencies:** BE-2.8, BE-4.4
  * **Incremental Verification:** After a salted-file pipeline run, the endpoint's category counts equal hand-run Snowsight aggregates over `FITMENT_STAGE.errors`.

* **Task BE-4.11: Failed-Records Endpoint (Paginated, Filterable)**
  * **Description:** Implement `GET /api/jobs/{job_id}/failed-records?error_type=&limit=&offset=` (limit cap 50) returning `app_id, supplier_part_number, base_vehicle_id, part_type_id, position_id, errors[], vehicle_conditions` from `FITMENT_STAGE` where `is_valid=FALSE`, with `error_type` filtering via a `FLATTEN` on the errors array. Parameterized SQL only.
  * **Dependencies:** BE-4.10
  * **Incremental Verification:** `?error_type=QDB` returns only records containing a QDB-typed error; `limit=2&offset=2` pages correctly; `limit=500` is clamped to 50.

---

#### Phase 5: Comparison, Statistics & Reports

* **Task BE-5.1: CATALOG.FITMENT Table + Snapshot Seeding**
  * **Description:** Create `CATALOG.FITMENT` per design doc 03 §3 (DFH primary key, VARIANT qualifiers/notes/conditions, product lookup optimization). Build a controlled seeding script `snowflake/02_catalog/seed_fitment_from_job.sql` that promotes a completed job's valid `FITMENT_STAGE` rows into `CATALOG.FITMENT` — used to create the "existing catalog" baseline for comparison testing.
  * **Dependencies:** BE-2.3, BE-4.7
  * **Incremental Verification:** Seed from the Phase-4 test job; `SELECT COUNT(*) FROM CATALOG.FITMENT` equals that job's valid-record count and DFH values are unique.

* **Task BE-5.2: Comparison Stored Procedures (with Review Fixes)**
  * **Description:** Implement `CLASSIFY_FITMENT_CHANGES_JOB(job_id)` per design doc 03 §5.5 with: delete-before-insert idempotency, `change_details` built via `CONCAT_WS` capturing **all** changed facets (review §4.3), and VARIANT comparisons over canonically-sorted arrays (guaranteed by BE-2.4 ingestion ordering; assert with `ARRAY_SORT` defensively — review §4.5). Implement `IDENTIFY_DELETED_FITMENTS_JOB(job_id)` with a `DELETE ... WHERE classification='DELETE'` guard before inserting (review §4.4). Note in proc comments the documented DFH nuance: qualifier-level changes surface as DELETE+ADD pairs (review §1.1 Option A — accepted).
  * **Dependencies:** BE-5.1
  * **Incremental Verification:** Build a 12-row scenario (3 ADD, 2 UPDATE-with-two-facet-changes, 4 UNCHANGED, 3 DELETE); `CALL` both procs and verify `FITMENT_CLASSIFICATION` matches exactly, an UPDATE row's `change_details` lists both facets, and re-running both procs leaves counts unchanged.

* **Task BE-5.3: Aggregation Stored Procedures + Summary Persistence**
  * **Description:** Implement `COMPUTE_VALIDATION_SUMMARY(job_id)` (adapted to count categories from the VARIANT `errors` array) and `COMPUTE_UPLIFT_SUMMARY(job_id)` per design doc 03 §5.6. Both pure-read, returning a single VARIANT object. Add a small helper proc `PERSIST_JOB_SUMMARY(job_id)` that writes the combined object into `JOB_RUN.error_summary`.
  * **Dependencies:** BE-5.2
  * **Incremental Verification:** `CALL` both against the 12-row scenario; the returned objects match the known truth (e.g., `add_count=3`); after `PERSIST_JOB_SUMMARY`, `JOB_RUN.error_summary` contains the identical JSON.

* **Task BE-5.4: MCP Server Update — Comparison & Aggregation Tools**
  * **Description:** Recreate/alter `ACES_FITMENT_SERVER` adding the remaining 4 tools (`classify_fitment_changes_job`, `identify_deleted_fitments_job`, `compute_validation_summary`, `compute_uplift_summary`) — total 12 tools, matching design doc 00 §7. Re-apply grants.
  * **Dependencies:** BE-5.3, BE-4.8
  * **Incremental Verification:** REST `tools/list` now returns 12 tools; a `tools/call` of `compute_uplift_summary` for the scenario job returns the correct counts.

* **Task BE-5.5: Report Storage + Download Endpoint**
  * **Description:** Define the report layout `gs://aces-reports-dev/{job_id}/` (`summary.json`, `uplift_report.xlsx`, `failed_records.csv` — written by AG-5.3). Implement `GET /api/jobs/{job_id}/reports` (lists available artifacts) and `GET /api/jobs/{job_id}/reports/{name}` returning a short-lived V4 signed URL (or streaming proxy). Resolves review doc §3.5.
  * **Dependencies:** BE-2.1, AG-5.3
  * **Incremental Verification:** After a full pipeline run, the list endpoint shows three artifacts and the signed URL downloads a valid `.xlsx` that opens in Excel with the right counts.

* **Task BE-5.6: Analysis Data Endpoints (Charts + Drawer + XML Snippet)**
  * **Description:** Implement the read APIs feeding the analysis dashboard: `GET /api/jobs/{id}/uplift` (classification counts + small representative samples per segment), `GET /api/jobs/{id}/error-distribution` (counts per category for the bar chart), `GET /api/jobs/{id}/uplift-records?classification=&limit=&offset=` (paged classified records with `change_details`), and `GET /api/jobs/{id}/records/{app_id}/source` (reconstructs/returns the raw `<App>` XML snippet by seeking the archived source in GCS using the stored `app_id`).
  * **Dependencies:** BE-5.3, BE-2.1
  * **Incremental Verification:** Each endpoint curled against the scenario job returns numbers matching Snowsight; the snippet endpoint returns well-formed XML whose `id` attribute equals the requested `app_id`.

---

#### Phase 6: Conversational Analytics

* **Task BE-6.1: Read-Only Chat Tool Queries (5 Tools, Hard Caps)**
  * **Description:** Implement the five JobQueryAgent data functions from design doc 02 §5.3 as parameterized read-only queries (or `MCP`-schema procs) in `libs/chat_tools`: `get_job_summary`, `get_failed_records`, `get_failed_records_by_cause`, `get_uplift_details`, `get_brand_or_part_performance`. Enforce `limit ≤ 50` and `job_id` as a required bind variable inside the SQL layer itself (not just the agent), keeping responses compact (<250KB).
  * **Dependencies:** BE-5.3
  * **Incremental Verification:** Invoke each function from a dev shell against the scenario job; outputs match Snowsight ground truth; passing `limit=5000` is clamped to 50.

* **Task BE-6.2: Chat Endpoint (`POST /api/jobs/{job_id}/chat`) with Chunked Streaming**
  * **Description:** Implement the chat route per design doc 02 §6.3: validate the job exists and is in a terminal state (else 409), inject the server-side `job_id` into the agent context (never trusting the message body), call the JobQueryAgent (AG-6.x) and stream its tokens back as `Transfer-Encoding: chunked` plain text. Maintain a per-session conversation key (header or query param) for short-term memory continuity.
  * **Dependencies:** BE-6.1, AG-6.3
  * **Incremental Verification:** `curl -N -X POST .../chat -d '{"message":"summarize this job"}'` streams tokens incrementally (visibly, not in one burst); chatting on a still-running job returns 409.

* **Task BE-6.3: Read-Only Role Segregation for Chat Path**
  * **Description:** Create Snowflake role `ACES_CHAT_READER` with SELECT-only on `STAGING` + `AUTOCARE` + `CATALOG` (no INSERT/UPDATE/DELETE, no MCP write procs); configure the chat tools' Snowflake session to use this role, enforcing the design's schema-segregation guardrail at the database layer.
  * **Dependencies:** BE-6.1
  * **Incremental Verification:** From a session under `ACES_CHAT_READER`, a `SELECT` on `FITMENT_STAGE` succeeds while an `UPDATE` fails with an authorization error.

---

#### Phase 7: Hardening, Performance & Operations

* **Task BE-7.1: 100K-Record Benchmark + Warehouse Sizing Decision**
  * **Description:** Generate a realistic ~100K-record / ~120MB ACES XML (script in `uploads-test/generators/`). Run full pipelines on SMALL vs MEDIUM `ACES_VALIDATION_WH`, recording per-node wall-clock from the event stream and Snowflake query history. Decide and set the dev/prod warehouse sizes; record the SLA baseline in `infra/performance.md` (closes the e2e doc's SLA open item).
  * **Dependencies:** Phase 5 complete (BE-5.4, AG-5.4)
  * **Incremental Verification:** The benchmark report exists with timings for both sizes; a fresh 100K upload on the chosen size completes within the documented SLA.

* **Task BE-7.2: Staging Retention & Cleanup Automation**
  * **Description:** Implement `MCP.PURGE_EXPIRED_STAGING()` deleting `FITMENT_STAGE`/`FITMENT_CLASSIFICATION` rows (and internal-stage files) for jobs older than 30 days in terminal states, scheduled via a Snowflake TASK (daily). Align GCS lifecycle rules (uploads 30d → delete; archive per BE-2.1). Resolves review doc §3.3.
  * **Dependencies:** BE-2.3
  * **Incremental Verification:** Backdate a test job's `created_at` 31 days, `EXECUTE TASK` manually, and confirm its staging rows and stage files are gone while a recent job's remain.

* **Task BE-7.3: Resumable Upload + Oversize Gate**
  * **Description:** Add chunked/resumable upload support (tus-style: initiate → PUT chunks with offsets → finalize, persisting offset state in Redis) so connection drops resume instead of restarting; keep the simple path for small files. Enforce the >150MB rejection at initiation time using the declared size.
  * **Dependencies:** BE-2.2, BE-3.1
  * **Incremental Verification:** Start a large upload, kill the network mid-transfer, resume — the job completes and the staged record count is correct (no duplicates/gaps); declaring a 200MB size at initiation is rejected immediately.

* **Task BE-7.4: Observability — Dashboards, Alerts, Audit Trail**
  * **Description:** Standardize structured logs (`job_id` on every line) across both services; build a Cloud Monitoring dashboard (request latency, 5xx rate, Cloud Tasks retries, jobs by terminal status, node durations from events); create alert policies (job FAILED rate spike, orchestrator 5xx, Redis disconnects, Cloud Build failures) routed to the team channel; document the Snowflake `QUERY_HISTORY` audit query pack.
  * **Dependencies:** BE-3.2
  * **Incremental Verification:** Force a job failure with a corrupt file — the alert arrives; filtering Cloud Logging by that `job_id` reconstructs the full cross-service execution story.

* **Task BE-7.5: Production Environment + Promotion Pipeline**
  * **Description:** Stand up prod: separate GCP project (or strictly-separated prod services + buckets + Redis + secrets), prod Firebase Hosting site, prod Snowflake database-or-account separation per team policy, and a tag-triggered (`v*`) Cloud Build promotion pipeline that deploys the already-built dev-verified image SHAs to prod (no rebuild). Keep `ENABLE_DEV_ENDPOINTS=false` in prod.
  * **Dependencies:** BE-7.1, BE-7.4
  * **Incremental Verification:** Pushing tag `v0.1.0` deploys to prod with zero manual steps; the prod URL serves the app; `POST /api/jobs/dev-seed` returns 404 in prod.

---

#### Phase 8: Security Hardening (Future / Nice-to-Have)

* **Task BE-8.1: User Authentication (Identity Platform + JWT Middleware)**
  * **Description:** Enable GCP Identity Platform (email/password + Google SSO), validate Firebase ID tokens in a BFF auth middleware applied to all `/api/*` routes, and propagate `user_id` into request context for downstream tasks.
  * **Dependencies:** Phase 7 complete
  * **Incremental Verification:** Requests without a valid `Authorization: Bearer` return 401; a logged-in browser session works end-to-end.

* **Task BE-8.2: Snowflake OAuth for MCP (Replace PAT)**
  * **Description:** Create the `ACES_MCP_OAUTH` security integration per design doc 03 §7, implement the OAuth client-credential/refresh flow in the orchestrator's MCP client, rotate out the PAT, and verify `DEFAULT_ROLE` behavior for `SVC_ACES_APP`.
  * **Dependencies:** BE-8.1
  * **Incremental Verification:** MCP `tools/call` succeeds with an OAuth access token; calls with the revoked PAT fail.

* **Task BE-8.3: Multi-Tenant Isolation (`user_id` on JOB_RUN)**
  * **Description:** Add `user_id` to `JOB_RUN`, stamp it at upload, and filter every job-scoped BFF query (list, detail, events, chat, reports) by the authenticated user. Backfill existing rows to an admin owner.
  * **Dependencies:** BE-8.1
  * **Incremental Verification:** User A's `GET /api/jobs/{B's job_id}` returns 403/404; A's job list shows only A's jobs.

* **Task BE-8.4: Rate Limiting + CORS Tightening**
  * **Description:** Add Redis-backed rate limiting on chat (e.g., 20 req/min/user) and upload (e.g., 5/hour/user); restrict CORS origins to the exact prod/dev hosting domains; add standard security headers.
  * **Dependencies:** BE-8.1
  * **Incremental Verification:** A scripted 30-requests-in-a-minute chat burst gets 429 on request 21; a `fetch` from an unlisted origin is CORS-blocked.

---

#### Phase 9: A2A Externalization (Post-MVP)

* **Task BE-9.1: Standalone A2A Service Deployment + Bearer Auth**
  * **Description:** Deploy the A2A server (AG-9.2) as Cloud Run service `aces-a2a` with its own SA (read-only Snowflake role `ACES_CHAT_READER`), public HTTPS ingress, and API-key/JWT bearer-token validation on all A2A methods (Agent Card endpoint remains public).
  * **Dependencies:** AG-9.2
  * **Incremental Verification:** `GET /.well-known/agent.json` is publicly reachable; `tasks/send` without a token returns 401, with a valid token returns a completed task.

* **Task BE-9.2: A2A Job-ID Authorization + Rate Limits**
  * **Description:** Map issued bearer tokens to permitted `job_id` scopes (token registry table or claims), reject out-of-scope job queries, and apply per-client rate limits mirroring BE-8.4.
  * **Dependencies:** BE-9.1, AG-9.5
  * **Incremental Verification:** A token scoped to job X gets `failed` task state when asking about job Y; burst traffic from one client is throttled while another client is unaffected.
