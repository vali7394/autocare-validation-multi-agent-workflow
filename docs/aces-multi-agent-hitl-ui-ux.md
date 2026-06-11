# ACES Fitment Validation — HITL UI/UX Screen Design & Ingestion Workflow

> **Document Purpose**: Design specification for the Human-In-The-Loop (HITL) UI screens, user experience (UX) flows, file upload architecture, and agent-side ingestion plumbing.  
> **Last Updated**: June 2026

---

## 1. HITL Screen Architecture

The application is structured as a Single Page Application (SPA) divided into three high-level layouts: **Job History Dashboard**, **Upload & Tracking Workspace**, and the **Interactive Chat & Analysis Split-Screen**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                              APPLICATION SHELL                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ SCREEN 1: Job History Dashboard                                  │  │
│  │ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐            │  │
│  │ │ Total Jobs    │ │ Avg Pass Rate │ │ Change Ratio  │            │  │
│  │ └───────────────┘ └───────────────┘ └───────────────┘            │  │
│  │ ┌──────────────────────────────────────────────────────────────┐ │  │
│  │ │ Job Runs Table                                               │ │  │
│  │ │ ID  │ File Name │ Supplier │ Status   │ Progress │ Actions   │ │  │
│  │ │ ─── │ ───────── │ ──────── │ ──────── │ ──────── │ ───────── │ │  │
│  │ │ 001 │ ACES_1    │ Bosch    │ SUCCESS  │ [======] │ [📥] [*💬*]│ │  │
│  │ └──────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Screen-by-Screen UX Specifications

### Screen 1: Job History / Management Dashboard
The primary landing screen. Provides a historical record of all submissions and serves as the starting point for analytical deep-dives.

*   **Primary Metrics Bar**:
    *   **Total Uploaded Files**: Aggregate count of files processed.
    *   **Average Pass Rate**: Quality index showing the percentage of valid records across all files.
    *   **Active Job Status**: Real-time ticker of jobs currently running.
*   **The Job Runs Table**:
    *   Columns: `Job ID` (Monospace), `Transfer Date`, `File Name`, `Line Code`, `Record Count`, `Status` (labeled with color-coded badges: Success, Warning, Failed, Processing), and `Actions`.
    *   **Critical Chat Icon Action**: Next to each completed or warning row, a prominent **"Chat Assistant" Icon** (styled as a chatbot or chat bubble with an active spark indicator) is displayed. 
    *   Hovering over this icon presents a tooltip: *"Converse with AI about validation issues and statistics for Job {job_id}"*.
    *   Clicking the chat icon slides open the **Screen 4 (Split-Screen Interactive Query)** layout, pre-initializing the chat agent with that specific job's context.
*   **Action Button**: "Upload New Fitments File" triggers the Screen 2 Modal overlay.

---

### Screen 2: File Upload / Ingestion Modal
An overlay dialog designed to capture the user's ACES XML file while conducting fast, low-latency pre-validations directly in the browser.

```
┌────────────────────────────────────────────────────────────────┐
│  Upload ACES XML Fitments File                                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌────────────────────────────────────────────────────────┐   │
│   │               [📂 Drag & Drop ACES XML File]           │   │
│   │                     Max limit: 150MB                   │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                │
│   Client-Side Preflight Summary:                               │
│   • Detected File: ACES_BP_2026.xml (112.4 MB)                │
│   • Schema Version: 4.1  |  Line Code: BP (Auto-detected)     │
│   • Estimated Records: ~105,000 Apps                           │
│                                                                │
│   [ Target Catalog Environment: [ Staging (Default) ] ]        │
│                                                                │
│   [ Cancel ]                                  [ Start Ingestion ]│
└────────────────────────────────────────────────────────────────┘
```

*   **Drag-and-Drop Area**: Highlighted dashed dropzone with support for `.xml` extensions.
*   **Instant Preflight Worker**:
    *   Upon selection, a lightweight web worker processes the first 50KB of the file.
    *   Parses the XML header to extract the company name, transmission date, ACES version, and supplier part line code.
    *   Retrieves an estimate of the record counts by scanning block structures in the file without locking up the main browser UI thread.
    *   Alerts the user early if the file doesn't conform to ACES structures (e.g., missing `<ACES>` or `<App>` nodes).
*   **Configuration Selectors**: Option to specify target catalog (e.g., Sandbox, Staging, Master production) and manual Line Code override.

---

### Screen 3: Real-Time Ingestion Progress Tracker
This interface replaces the upload modal immediately upon submission. It can run in a full-screen focus view or a minimized background card.

*   **The Orchestrator Progress Stepper**:
    Displays five stages corresponding to the backend's active agentic node:
    1.  **Parsing & Staging**: Showcases network upload progress and staging progress (e.g., *"Parsed and staged 54,000 / 100,000 records into Snowflake"*).
    2.  **Schema Validation**: Animates as a validation wheel. Checks XSD schema alignment.
    3.  **Ref Data Checks (VCDB, PCDB, QDB)**: Displays live check counters (e.g., *"VCDB base vehicle resolution checking... 1,420 errors found"*).
    4.  **Catalog Change Comparison**: Tracks change categorization activity (ADD, UPDATE, DELETE).
    5.  **Finalizing**: Generates final reports, registers the completed run, and archives raw storage.
*   **Live Error Log Ticker**:
    *   A miniature, scrollable shell terminal window at the bottom of the screen.
    *   Streams incoming warning logs in real-time to maintain transparency, e.g.:
        `[17:12:04] WARNING: Part BP-901 has invalid qualifier Q-441`
        `[17:12:15] ERROR: Vehicle ID 54101 does not exist in VCDB master`

---

### Screen 4: Split-Screen Interactive Query Workspace
The definitive analytical view triggered by clicking the "Chat Assistant" icon on the dashboard or upon pipeline completion.

```
┌───────────────────────────────────────────┬────────────────────────────┐
│ JOB ANALYSIS: ACES_BP_2026_FULL.xml       │ 💬 Job Query Assistant     │
├───────────────────────────────────────────┼────────────────────────────┤
│ ┌───────────────────┐  ┌────────────────┐ │ Chatting about Job:        │
│ │   98.2% Pass      │  │ Uplift Summary │ │ job_01h9ybc93k018s96abc491 │
│ │  (102.4k Valid)   │  │   +1,240 Adds  │ ├────────────────────────────┤
│ └───────────────────┘  │   -430 Deletes │ │ [AI] Hello! I've loaded the│
│ ┌───────────────────┐  └────────────────┘ │ validation data for this   │
│ │ Error Distribution│  ┌────────────────┐ │ run. I detected:           │
│ │ VCDB: [======== ] │  │ Top Failure    │ │ • 3,890 Validation Errors  │
│ │ PCDB: [===      ] │  │ BP-2120 (420x) │ │ • 1,240 New catalog additions│
│ │ QDB : [=        ] │  │                │ │ What would you like to drill│
│ └───────────────────┘  └────────────────┘ │ down into?                 │
│ ┌───────────────────────────────────────┐ │                            │
│ │ Failed Record Sample Drawer           │ │ [User] why did BP-2120 fail│
│ │ Part  │ Vehicle │ Msg                 │ │ so many times?             │
│ │ ───── │ ─────── │ ─────────────────── │ │                            │
│ │ 2120  │ 48202   │ Vehicle missing     │ │ [AI] BP-2120 failed due to │
│ └───────────────────────────────────────┘ │ [..Analyzing Snowflake..]  │
│                                           │ VCDB Base Vehicle ID       │
│                                           │ mismatch on ID 48202.      │
└───────────────────────────────────────────┴────────────────────────────┘
```

#### Left Pane (60% Width): Visual Status Dashboard
Provides a detailed visual snapshot of the file's results.
*   **Header Summary Card**: Displays overall file health, final validation status, and pipeline execution time.
*   **Uplift Pie Chart**: Displays interactive segments of ADDs, UPDATEs, UNCHANGED, and DELETEs. Hovering over a segment shows representative part numbers.
*   **Validation Error Groupings (Bar Chart)**: Segregates VCDB, PCDB, QDB, and Brand/Product failures.
*   **Sample Drawer**: A searchable, scrollable spreadsheet panel showing failed records, matching the exact schema of `FITMENT_STAGE`. Clicking a row displays the raw XML snippet that generated the error.

#### Right Pane (40% Width): Conversational Chat Assistant Sidebar
A slide-out drawer providing natural language access to the backend's `JobQueryAgent`.
*   **Quick Suggestions Chips**: Floating pill buttons with common queries to speed up interaction:
    *   *"Show the top 3 vehicle resolution failures."*
    *   *"List the parts that have brand warnings."*
    *   *"What was the impact on our Ford fitment catalog?"*
*   **AI Chat Area**:
    *   Renders messages using fully-formatted Markdown tables, lists, and bold callouts.
    *   Includes animated state indicators when the agent is querying Snowflake (e.g., *"Assistant is querying Snowflake tables..."*).
*   **Input Box**: Standard typing area with automated prompt history (Arrow Up key) and mic input for voice query capability.

---

## 3. Upload & Ingestion Flow Architecture

To successfully move 100K+ records from a user's browser into Snowflake, the system uses a streaming ingest pattern that avoids loading entire file datasets into either client or backend memory.

### 3.1 Step-by-Step Data Flow

```
   User File Drop (Browser)
             │
             │ (1) Stream Upload via Multipart POST
             ▼
      Backend BFF Ingestion
             │
             │ (2) Streaming SAX XML Parser
             ▼
      Batch JSONL Files (10k batches)
             │
             │ (3) Stream Upload via AWS/GCS SDK
             ▼
      Snowflake Internal Stage
             │
             │ (4) SQL command: COPY INTO FITMENT_STAGE
             ▼
     Snowflake Staging Tables
             │
             │ (5) Trigger LangGraph Orchestrator (pass job_id)
             ▼
     LangGraph Node Executions (Set-Based SQL Procedures)
```

1.  **Client-to-BFF Stream Ingest**:
    The React UI reads the file from the local file input and uses a `ReadableStream` to POST the binary XML to `/api/jobs/upload`. The browser's memory consumption remains under 10MB during this transfer.

2.  **BFF Memory-Efficient Parsing**:
    The BFF API gateway receives the file stream. Instead of buffer caching or saving the entire file to disk, it routes the incoming stream directly through a **SAX (Simple API for XML) Parser** (e.g., Python `xml.etree.ElementTree.iterparse` or Node `saxy` / `sax-js`).
    *   *How it works*: The SAX parser is event-driven; it triggers code blocks only when hitting opening tags like `<App>` and closing tags like `</App>`, discarding parsed elements immediately to keep memory usage locked under 100MB.

3.  **Chunking & Gzip Staging**:
    *   As the BFF SAX parser extracts each `<App>` block, it converts it into a flattened JSON structure representing a staging record.
    *   The BFF collects these structures into batches of **10,000 records**.
    *   Each batch is written to a local compression buffer (Gzip), creating a line-delimited JSON (JSONL) file.
    *   The BFF uploads these compressed JSONL files asynchronously to a secure **Snowflake Internal Stage** (`@ACES_STAGING_STAGE/job_id/`) using the Cloud SDK.

4.  **Bulk Copy Execution**:
    Once the BFF parser hits EOF (End-of-File) and all batch files are staged, it issues a single SQL command to Snowflake using JDBC/ODBC:
    ```sql
    COPY INTO ACES_VALIDATION.STAGING.FITMENT_STAGE
    FROM @ACES_STAGING_STAGE/job_id/
    FILE_FORMAT = (TYPE = JSON COMPRESSION = GZIP)
    MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
    ```
    This instructs Snowflake to ingest the staged data in parallel. Staging 100,000 records takes **less than 10 seconds** and ensures zero row-by-row overhead.

5.  **Agent Orchestration Activation**:
    With staging tables fully loaded, the BFF issues an asynchronous trigger to start the **LangGraph Orchestrator**, passing only the newly registered `job_id`.
    *   The orchestrator initiates the state machine.
    *   Because the data is already inside Snowflake, the validation nodes can execute set-based stored procedures instantly.

---

## 4. Ingestion Failures & Edge-Case Management

### 4.1 File Ingest & Processing Edge Cases

| Edge Case | Problem | Mitigation Strategy |
| :--- | :--- | :--- |
| **Connection Drop during Upload** | File upload is interrupted at 50% transfer. | Implements **tus-protocol** or multipart chunked uploads. Browser stores upload offset in local storage, enabling resume capabilities without restarting the file transfer from scratch. |
| **Extremely Large Files (>150MB)** | SAX Parser processes indefinitely, potentially causing BFF resource exhaustion. | BFF limits maximum upload boundary. Files above 150MB are rejected at the REST API gate, or redirected to a direct cloud staging bucket (e.g., AWS S3 bucket), bypassing the BFF completely and staging via Snowflake external stages. |
| **Malformed XML Elements** | The XML has broken tag enclosures or missing closing structures. | The SAX parser captures the parse error event, immediately aborts the upload, writes a descriptive error log (e.g., *"Malformed XML at Line 42011: Expected closing tag </App>"*), and sets the job status to `FAILED`. |
| **Invalid/Missing Line Code** | The file header contains a different brand line code than the user's workspace role. | The preflight step compares the header line code to the user's session token. A mismatch triggers a **Warnings Modal** allowing the user to either override the workspace line code or abort the run. |

### 4.2 Analytical Agent Chat Edge Cases

| Edge Case | Problem | Mitigation Strategy |
| :--- | :--- | :--- |
| **No Validation Failures Found** | User asks: *"Why did the validation fail?"* when the job status is `SUCCESS`. | The agent reads the job summary first. If `invalid_records = 0`, it triggers a friendly response: *"This job completed with 100% success! No validation failures were encountered. The catalog changes are ready for review."* |
| **Huge DB Datasets Returned to Chat** | User asks: *"List all the failed records in detail."* (where there are 15,000 errors). | The agent's tools enforce a hard pagination limit (maximum 50 records) and return a paginated summary message. The agent outputs: *"I've displayed the first 10 validation failures. You can scroll through the spreadsheet drawer on the left side of the screen to view all 15,000 failures in detail."* |
| **Ambiguous Conversational Queries** | User asks: *"Why is everything broken?"* | The agent queries the overall metrics to find the highest-frequency failure cause and structures an organized summary: *"Looking at the data, the main issue is that 84% of your errors (3,267 occurrences) are due to missing positions in PCDB. Let me show you what that means..."* |

---

*ACES Fitment Validation — HITL UI/UX Screen Design & Ingestion Workflow*
