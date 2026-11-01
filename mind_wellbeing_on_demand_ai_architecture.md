# Mind Wellbeing Journal --- On-Demand AI Analysis Architecture

**Document type:** Architecture and implementation guide\
**Status:** Proposed MVP architecture\
**Last updated:** 1 November 2026

------------------------------------------------------------------------

## 1. Purpose

This document describes the proposed architecture for a mental wellbeing
journaling application.

The application lets users submit daily journal entries and structured
wellbeing information. The application stores this information first.
When the user chooses, they press a frontend button labelled **Analyze
My Mind Report**.

That action starts a backend workflow. The backend invokes an AI agent
that uses tools exposed through an MCP server to retrieve the
authenticated user's latest data from the database. The agent analyses
the retrieved information and produces a structured report. The backend
validates and stores the report, and the frontend retrieves and displays
it.

The key product decision is that **analysis is user-triggered**, not
automatically performed every time a journal entry is submitted.

### Main goals

-   Keep daily journaling simple.
-   Store original user-submitted data reliably.
-   Let the user decide when to request an analysis.
-   Use MCP tools to provide the agent with controlled access to
    application data.
-   Generate a predictable, structured report.
-   Store reports so users can view previous analyses.
-   Keep the initial implementation small enough to build and test
    incrementally.

------------------------------------------------------------------------

## 2. High-Level Architecture

``` text
                  USER
                    |
                    v
           FRONTEND APPLICATION
                    |
                    | Submit journal entry
                    v
               FASTAPI API
                    |
                    | Validate and persist
                    v
                 MONGODB
                    |
                    |
          User opens Mind Dashboard
                    |
                    v
       [ ANALYZE MY MIND REPORT ]
                    |
                    | User clicks button
                    v
          FASTAPI ANALYSIS ENDPOINT
                    |
                    | Starts analysis job
                    v
           ANALYSIS ORCHESTRATOR
                    |
                    v
               AI AGENT
                    |
                    | Calls tools through MCP
                    v
                MCP SERVER
                    |
          +---------+----------+
          |         |          |
          v         v          v
       Fetch     Fetch       Fetch
      Journals   Mood       Sleep/Stress
          |         |          |
          +---------+----------+
                    |
                    v
              MONGODB DATA
                    |
                    v
             AI AGENT ANALYSIS
                    |
                    v
          STRUCTURED REPORT OUTPUT
                    |
                    v
         BACKEND VALIDATION (Pydantic)
                    |
                    v
           SAVE REPORT IN MONGODB
                    |
                    v
          FRONTEND REPORT API
                    |
                    v
             MIND DASHBOARD
```

### Architecture in one sentence

**Frontend collects data → FastAPI stores it → user requests analysis →
backend invokes the agent → agent uses MCP tools to retrieve data →
agent generates structured output → backend validates and saves the
report → frontend displays it.**

------------------------------------------------------------------------

## 3. Application Workflow

### Step 1 --- User submits a journal entry

The user completes the daily journaling page. The page may collect:

-   Mood rating
-   Emotion tags
-   Free-text journal content
-   Optional triggers
-   Sleep duration and quality
-   Optional stress rating
-   Positive activities
-   Tomorrow's intention

Not every field should be mandatory. The free-text journal and the
user's ability to save an entry are central.

The frontend sends the entry to FastAPI.

Example:

``` http
POST /api/v1/journals
Authorization: Bearer <access-token>
Content-Type: application/json
```

Example request body:

``` json
{
  "entry_date": "2026-11-01",
  "mood": {
    "rating": 3,
    "label": "Okay"
  },
  "emotions": ["anxiety", "loneliness"],
  "triggers": ["work", "relationships"],
  "journal_text": "Today was stressful...",
  "sleep": {
    "duration_hours": 6.5,
    "quality": 3
  },
  "stress_level": 4,
  "positive_activities": ["gym"],
  "tomorrow_intention": "Take a short walk"
}
```

### Step 2 --- FastAPI validates and stores the entry

FastAPI should:

1.  Authenticate the user.
2.  Validate the request with Pydantic.
3.  Derive the user's identity from the authenticated session or token.
4.  Apply business validation, such as valid rating ranges.
5.  Save the original entry in MongoDB.
6.  Return the created entry identifier and status.

Example response:

``` json
{
  "entry_id": "entry_101",
  "status": "saved"
}
```

**Important:** Journal submission and AI analysis are separate
operations. Saving an entry should not require the AI agent to run.

### Step 3 --- User presses "Analyze My Mind Report"

The dashboard contains a clear action:

**Analyze My Mind Report**

Supporting text can explain:

> Review your recent journal entries, mood, emotions, sleep and stress
> to generate a personal wellbeing report.

When the user presses the button, the frontend calls the analysis
endpoint.

``` http
POST /api/v1/mind-report/analyze
Authorization: Bearer <access-token>
Content-Type: application/json
```

For the MVP, the analysis period can default to the **last 7 days**. The
backend should calculate the date range using the user's configured
timezone.

An optional future request body could support a date range:

``` json
{
  "period": "last_7_days"
}
```

The backend should not trust a user ID supplied in the request body. It
should use the authenticated user's identity.

### Step 4 --- Backend starts the analysis workflow

The analysis endpoint should not keep the HTTP request open while a
potentially slow LLM operation runs.

Recommended behaviour:

1.  Check whether the user already has an analysis job in progress.
2.  Create an analysis job record.
3.  Enqueue or start the analysis workflow.
4.  Return a job ID and `processing` status.

Example response:

``` json
{
  "job_id": "job_501",
  "status": "processing",
  "message": "Your Mind Report is being prepared."
}
```

For a robust deployment, use a background worker and durable job queue.
A lightweight background task can be useful for local experimentation,
but it should not be treated as a durable production job system.

### Step 5 --- Agent uses MCP tools to retrieve data

The analysis orchestrator invokes the AI agent. The agent receives
access to a limited set of MCP tools.

The tools retrieve data for the authenticated user and requested date
range.

Suggested initial tools:

  -----------------------------------------------------------------------
  MCP tool                            Responsibility
  ----------------------------------- -----------------------------------
  `get_user_journals`                 Retrieve journal entries in the
                                      analysis period

  `get_mood_history`                  Retrieve mood ratings and emotion
                                      records

  `get_sleep_history`                 Retrieve sleep duration and quality

  `get_stress_history`                Retrieve stress records, if stored
                                      separately

  `get_previous_mind_report`          Retrieve the previous report for
                                      comparison, if available
  -----------------------------------------------------------------------

The exact tool list should match the final database design. If mood,
sleep and stress are embedded in each journal document, a single
authorised retrieval tool may initially return all of these fields.
Avoid creating tools that merely duplicate one another.

#### MCP responsibilities

MCP provides a standard tool interface between the agent and application
capabilities.

The MCP server should: - Expose only the tools the agent needs. -
Validate tool arguments. - Enforce user and date-range access
restrictions. - Return only the minimum necessary data. - Handle
database errors and empty results clearly.

The agent must not be allowed to select arbitrary user IDs to retrieve
another user's records. The authenticated identity should be established
by the trusted backend/orchestrator and passed through a controlled
context or server-side mechanism.

MCP is the tool-access layer. It does not itself perform the analysis.

### Step 6 --- Agent analyses the retrieved data

The agent receives the authorised records and generates an analysis.

Its responsibilities may include:

-   Summarising the user's week.
-   Identifying frequently reported emotions.
-   Highlighting recurring themes in journal text.
-   Describing changes in self-reported mood.
-   Identifying possible associations between reported stressors, sleep
    and mood.
-   Highlighting activities the user says were helpful.
-   Suggesting gentle reflection questions or practical wellbeing
    activities.

The agent should distinguish between: - Directly reported information. -
Deterministic calculations. - AI-generated interpretations. -
Suggestions.

It must not present an association as proof of cause and effect.

#### Deterministic calculations versus LLM interpretation

Use Python for calculations that should be repeatable:

-   Average mood rating.
-   Number of journal entries.
-   Emotion frequency.
-   Average sleep duration.
-   Stress distribution.
-   Comparison with the previous week.

Use the LLM for language and context:

-   Summarising journal entries.
-   Describing recurring themes.
-   Explaining possible patterns.
-   Producing reflection questions.
-   Generating supportive suggestions.

Do not rely on the LLM to calculate numerical metrics from raw journal
text when ordinary code can calculate them reliably.

### Step 7 --- Agent produces structured output

The agent should return data that follows a defined JSON contract.

Example:

``` json
{
  "period": {
    "start_date": "2026-10-26",
    "end_date": "2026-11-01"
  },
  "summary": "Your mood ratings were higher toward the end of the week.",
  "metrics": {
    "journal_entries_count": 7,
    "average_mood": 3.2,
    "average_sleep_hours": 6.4
  },
  "observations": [
    {
      "type": "recurring_theme",
      "title": "Work-related stress",
      "description": "Work pressure appeared in several entries alongside anxiety.",
      "evidence_entry_ids": [
        "entry_102",
        "entry_104"
      ]
    }
  ],
  "suggestions": [
    {
      "title": "Create a short transition after work",
      "description": "Consider a short activity that helps you unwind after work."
    }
  ],
  "limitations": [
    "This report is based on self-reported journal data.",
    "Observed associations do not establish causation."
  ]
}
```

This is an illustrative contract. Final field names, enums, limits and
validation rules should be defined in a Pydantic model.

### Step 8 --- Backend validates and stores the report

Never save arbitrary LLM output directly as trusted application data.

The backend should:

1.  Parse the agent response.
2.  Validate it against a Pydantic schema.
3.  Verify that referenced entry IDs belong to the authenticated user
    and analysis period.
4.  Reject or safely handle malformed output.
5.  Save the report and its metadata in MongoDB.
6.  Mark the analysis job as completed or failed.

Store useful metadata, such as: - User ID - Analysis period - Source
entry IDs - Generation timestamp - Model identifier/version - Prompt or
analysis version - Report schema version - Job ID and processing status

Keep original journal entries unchanged. AI reports are derived records,
not replacements for user writing.

### Step 9 --- Frontend retrieves and displays the report

The frontend should not need to call MCP directly.

It calls the backend:

``` http
GET /api/v1/mind-report/latest
Authorization: Bearer <access-token>
```

The backend retrieves the latest completed report belonging to the
authenticated user and returns the structured data.

The frontend renders predefined components: - Weekly summary - Mood
chart - Emotion frequency - Sleep and stress metrics - Recurring
themes - Suggestions - Evidence links to relevant journal entries -
Limitations and report period

The frontend controls presentation. The LLM should not generate
arbitrary HTML or frontend code.

------------------------------------------------------------------------

## 4. API Endpoints

A small initial API surface is sufficient.

  ---------------------------------------------------------------------------------------
  Method                  Endpoint                                Purpose
  ----------------------- --------------------------------------- -----------------------
  `POST`                  `/api/v1/journals`                      Create a journal entry

  `GET`                   `/api/v1/journals`                      Retrieve journal
                                                                  history

  `GET`                   `/api/v1/journals/{entry_id}`           Retrieve one entry

  `PATCH`                 `/api/v1/journals/{entry_id}`           Update an entry

  `DELETE`                `/api/v1/journals/{entry_id}`           Delete an entry

  `POST`                  `/api/v1/mind-report/analyze`           Start on-demand
                                                                  analysis

  `GET`                   `/api/v1/mind-report/status/{job_id}`   Check analysis status

  `GET`                   `/api/v1/mind-report/latest`            Retrieve latest
                                                                  completed report

  `GET`                   `/api/v1/mind-report/history`           Retrieve previous
                                                                  reports
  ---------------------------------------------------------------------------------------

Every user-specific endpoint must enforce authentication and record
ownership.

### Analysis status values

Use a small, explicit state model:

-   `queued` --- accepted and waiting to run.
-   `processing` --- analysis is running.
-   `completed` --- validated report saved successfully.
-   `failed` --- analysis failed.

The frontend can poll the status endpoint at a reasonable interval while
the job is running. A future implementation can use Server-Sent Events
or WebSockets if live updates become necessary.

### Duplicate request handling

Prevent multiple accidental clicks from creating several simultaneous
jobs for the same user and period.

Possible approach: - Disable the button while a job is active. - Enforce
a server-side active-job check. - Use an idempotency key if retry-safe
submission is required.

------------------------------------------------------------------------

## 5. MongoDB Collections

The initial database can use the following collections.

### `users`

Stores account and basic profile information. Authentication credentials
should be securely hashed or delegated to an identity provider.

### `journal_entries`

Stores one document per journal entry. Example:

``` json
{
  "_id": "entry_101",
  "user_id": "user_101",
  "entry_date": "2026-11-01",
  "mood": {
    "rating": 3,
    "label": "Okay"
  },
  "emotions": ["anxiety", "loneliness"],
  "triggers": ["work", "relationships"],
  "journal_text": "Today was stressful...",
  "sleep": {
    "duration_hours": 6.5,
    "quality": 3
  },
  "stress_level": 4,
  "positive_activities": ["gym"],
  "created_at": "2026-11-01T20:30:00Z",
  "updated_at": "2026-11-01T20:30:00Z",
  "schema_version": 1
}
```

Do not store an unlimited number of entries inside one user document.

### `analysis_jobs`

Tracks user-triggered analysis requests.

``` json
{
  "_id": "job_501",
  "user_id": "user_101",
  "period": {
    "start_date": "2026-10-26",
    "end_date": "2026-11-01"
  },
  "status": "processing",
  "report_id": null,
  "error_code": null,
  "created_at": "2026-11-01T21:00:00Z",
  "completed_at": null
}
```

Avoid storing raw journal text or sensitive model prompts in job error
logs.

### `mind_reports`

Stores validated, structured analysis output.

``` json
{
  "_id": "report_501",
  "job_id": "job_501",
  "user_id": "user_101",
  "period": {
    "start_date": "2026-10-26",
    "end_date": "2026-11-01"
  },
  "source_entry_ids": [
    "entry_101",
    "entry_102"
  ],
  "summary": "Your mood ratings were higher toward the end of the week.",
  "metrics": {},
  "observations": [],
  "suggestions": [],
  "limitations": [
    "Based on self-reported journal information."
  ],
  "model_version": "configured-model-version",
  "prompt_version": "1.0",
  "schema_version": 1,
  "generated_at": "2026-11-01T21:05:00Z"
}
```

The examples are illustrative. Use MongoDB `ObjectId` or another
consistent identifier strategy in the implementation; do not mix
arbitrary string IDs and database IDs without a defined conversion
policy.

------------------------------------------------------------------------

## 6. Suggested Backend Project Structure

A maintainable FastAPI structure could look like:

``` text
app/
├── main.py
├── core/
│   ├── config.py
│   ├── security.py
│   └── logging.py
├── api/
│   └── v1/
│       ├── journals.py
│       └── mind_reports.py
├── models/
│   ├── journal.py
│   ├── mind_report.py
│   └── analysis_job.py
├── schemas/
│   ├── journal.py
│   ├── mind_report.py
│   └── analysis_job.py
├── services/
│   ├── journal_service.py
│   ├── report_service.py
│   └── analysis_service.py
├── repositories/
│   ├── journal_repository.py
│   ├── report_repository.py
│   └── job_repository.py
├── workers/
│   └── analysis_worker.py
├── integrations/
│   └── mcp_client.py
└── db/
    └── mongodb.py

mcp_server/
├── server.py
└── tools/
    ├── journal_tools.py
    ├── mood_tools.py
    └── sleep_tools.py

tests/
├── api/
├── services/
├── workers/
└── mcp/
```

This is a suggested structure, not a requirement to create every file
immediately. Start with the modules needed for journal CRUD and add the
analysis modules when the first workflow is ready.

------------------------------------------------------------------------

## 7. Recommended Technology

  Layer                   Technology
  ----------------------- ----------------------------------------
  Frontend                React
  Backend                 FastAPI
  Database                MongoDB
  Validation              Pydantic
  MongoDB access          Current PyMongo asynchronous API
  Background processing   A durable worker and queue when needed
  AI tool integration     MCP
  Testing                 Pytest and HTTPX
  Local environment       Docker Compose
  API documentation       FastAPI OpenAPI/Swagger

Do not introduce a message broker, multiple agents, or elaborate
orchestration before the basic user-triggered workflow is working. A
simple local implementation can prove the workflow first; production
deployment should use durable job execution and recovery.

------------------------------------------------------------------------

## 8. Security and Privacy Requirements

Journal data can contain highly sensitive personal information. Apply
privacy and security controls from the beginning.

-   Require authentication for journal and report endpoints.
-   Derive user identity from the authenticated request, not a
    client-supplied user ID.
-   Enforce ownership checks in API routes, database queries, and MCP
    tools.
-   Restrict MCP tools to the minimum data and operations required.
-   Do not expose the database directly to the LLM.
-   Treat journal text as untrusted content. Text in a journal must
    never override agent instructions or tool permissions.
-   Encrypt traffic and stored sensitive data.
-   Avoid writing raw journal text into application logs.
-   Obtain clear consent for AI processing and any clinician sharing.
-   Keep original user writing separate from generated interpretations.
-   Provide a way to correct and delete entries and derived reports,
    subject to a clearly stated retention policy.
-   Maintain audit records for sensitive access and sharing.
-   Establish backups and test restoration.

### Wellbeing and clinical boundary

This application provides journaling and wellbeing reflection. It is not
a diagnostic tool and should not claim to diagnose or treat a mental
health condition.

AI observations must be presented as tentative interpretations of
self-reported information. Do not present correlation as causation or
generate an arbitrary mental health score.

Any disclosure of imminent self-harm or danger requires a separately
designed safety process directing the user to appropriate human or
emergency support. A weekly report agent is not an emergency-response
system.

------------------------------------------------------------------------

## 9. Failure Handling

The workflow should account for ordinary failures.

  -----------------------------------------------------------------------
  Failure                             Expected behaviour
  ----------------------------------- -----------------------------------
  No journal entries in the selected  Return a clear message that there
  period                              is insufficient data

  MCP tool unavailable                Mark the job failed or retry
                                      safely; do not fabricate missing
                                      data

  Database unavailable                Return a service error and preserve
                                      job state where possible

  LLM timeout                         Retry within bounded limits, then
                                      mark failure

  Invalid LLM JSON                    Validate, optionally retry with a
                                      repair instruction, then fail
                                      safely

  Report save failure                 Do not mark the job completed

  User clicks repeatedly              Reuse or reject while an active job
                                      exists

  User deletes a source entry         Apply a defined report invalidation
                                      or regeneration policy
  -----------------------------------------------------------------------

Do not show a report as completed until its output has passed validation
and has been persisted successfully.

------------------------------------------------------------------------

## 10. Development Plan

Build this in stages. Do not start with the agent.

### Phase 1 --- Journaling API and database

-   Create the FastAPI project.
-   Configure MongoDB.
-   Define the journal document schema.
-   Implement create, read, update, and delete operations.
-   Add authentication and ownership checks.
-   Test validation and persistence.

**Completion criterion:** A user can securely submit and retrieve their
own journal entries.

### Phase 2 --- Basic frontend

-   Build the daily journaling form.
-   Connect it to the journal API.
-   Build journal history.
-   Build the initial dashboard.
-   Add the "Analyze My Mind Report" button, initially disabled or
    connected to a placeholder state.

**Completion criterion:** The user can submit, edit, and review journal
entries through the UI.

### Phase 3 --- Analysis job lifecycle

-   Add `analysis_jobs`.
-   Implement the analysis endpoint.
-   Implement status retrieval.
-   Prevent duplicate active jobs.
-   Add a simple worker.
-   Test queued, processing, completed, and failed states.

**Completion criterion:** Clicking the button creates a trackable
backend analysis job, even before AI is connected.

### Phase 4 --- MCP server and tools

-   Build the MCP server.
-   Implement authorised journal retrieval.
-   Add mood, sleep, and previous-report tools only as required by the
    data model.
-   Test tool inputs, outputs, permissions, and error handling.

**Completion criterion:** The agent can retrieve only the correct user's
records for the requested period.

### Phase 5 --- Agent and structured report

-   Connect the agent to the MCP tools.
-   Define the report Pydantic schema.
-   Add deterministic Python analytics.
-   Add LLM-based contextual interpretation.
-   Validate the result.
-   Save the report and update the job status.

**Completion criterion:** A user-triggered job produces a valid,
persisted report from actual source data.

### Phase 6 --- Dashboard report

-   Implement the latest-report endpoint.
-   Render summary, metrics, charts, observations, and suggestions.
-   Display report period and data limitations.
-   Add report history.
-   Add retry behaviour for failed analysis.

**Completion criterion:** The user can press the button, wait for
analysis, and view the completed report.

------------------------------------------------------------------------

## 11. Decisions to Make Before Implementation

These decisions affect the API, database and report logic:

1.  **Analysis period:** Start with the last 7 days, or allow the user
    to select a date range?
2.  **Timezone:** Which user timezone defines the start and end of a day
    and week?
3.  **Minimum data:** Should the report require a minimum number of
    entries, or generate a limited report with a clear data warning?
4.  **Multiple entries:** Can a user create multiple journal entries per
    day?
5.  **Report history:** Should every generated report be retained?
6.  **Regeneration:** If a user edits or deletes an entry after a report
    is generated, should the report be marked stale?
7.  **AI consent:** How will the application obtain and record consent
    before sending journal content for AI processing?

Recommended MVP defaults: - Last 7 days. - User-configured timezone. -
Generate only what the available data supports; clearly disclose
insufficient data. - Allow multiple entries per day. - Retain report
history. - Mark a report as potentially stale when its source entries
change. - Obtain explicit consent before AI processing.

------------------------------------------------------------------------

## 12. Final Architecture Principle

Keep the workflow understandable:

**Collect → Store → User requests analysis → Backend starts job → Agent
retrieves data through MCP → Analyse → Validate structured output →
Store report → Frontend displays report.**

The first engineering milestone is a working journal API and MongoDB
persistence. The next milestone is the button and analysis-job
lifecycle. MCP and the agent can then be integrated into a workflow that
already has clear inputs, outputs, permissions, and failure handling.

This keeps the MVP focused while leaving a clean path toward the
AI-powered Mind Report experience.
