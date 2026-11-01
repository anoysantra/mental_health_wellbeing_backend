# Mental Wellbeing Journaling Application

## Product Concept, Architecture, Data Model, and Development Plan

**Document status:** Initial product and technical planning\
**Date:** 1 November 2026

------------------------------------------------------------------------

## 1. Product Vision

Build a mental wellbeing journaling application that allows users to
record daily experiences, emotions, mood, sleep, stress, and personal
reflections. Over time, the application can analyse the user's own
historical entries and provide a weekly dashboard with understandable
trends, recurring themes, and practical reflection suggestions.

The product should feel like a personal diary rather than a clinical
assessment form.

### Core product principles

-   Daily journaling should take approximately 3--5 minutes.
-   Free-text writing is the primary activity; structured questions are
    optional supporting inputs.
-   Users should be able to review and search their own historical
    entries.
-   Weekly insights should be grounded in the user's actual records.
-   AI-generated interpretations must remain distinguishable from
    user-reported facts.
-   The application must not diagnose mental health conditions or
    present AI-generated observations as clinical conclusions.
-   Privacy, consent, security, and user control are foundational
    requirements.

------------------------------------------------------------------------

## 2. Daily Journaling Experience

The daily journal is the central feature.

### Suggested daily entry flow

1.  **Mood check-in**

    -   Very bad
    -   Low
    -   Okay
    -   Good
    -   Great

    Store the rating on a consistent 1--5 scale.

2.  **Emotion selection**

    -   Happy
    -   Anxious
    -   Sad
    -   Angry
    -   Calm
    -   Lonely
    -   Stressed
    -   Motivated
    -   Overwhelmed

    Allow users to select multiple emotions. The list should be
    configurable and should not imply a clinical classification.

3.  **Free-text journal**

    -   Prompt: "What happened today? What is on your mind?"
    -   Users can write freely without being forced to answer a fixed
        questionnaire.
    -   Support saving drafts and editing entries.

4.  **Optional mood triggers**

    -   Work
    -   Relationships
    -   Family
    -   Health
    -   Finances
    -   Exercise
    -   Sleep
    -   Other

5.  **Optional sleep and stress details**

    -   Sleep duration
    -   Sleep quality
    -   Stress rating

6.  **Tomorrow's intention**

    -   Example: "Go for a walk" or "Finish an important task."

7.  **Optional additional note**

### UX requirements

-   Keep the page uncluttered and mobile-friendly.
-   Make free-text writing prominent.
-   Do not require every optional field.
-   Clearly indicate whether an entry is saved.
-   Allow users to edit or delete their entries.
-   Explain how entries are stored, processed, and shared.
-   Do not share journal entries with a clinician or another person
    without explicit user consent and appropriate access controls.

------------------------------------------------------------------------

## 3. Main Application Screens

The initial product can be organised into four screens.

### Screen 1 --- Daily Dashboard

Display: - Today's mood check-in - Start writing action - Recent journal
entries - Weekly mood overview - Reminder or journaling consistency
information, if enabled by the user

### Screen 2 --- Journal Writing

Display: - Mood selector - Emotion tags - Large free-text writing area -
Optional trigger tags - Optional sleep and stress inputs - Save draft
and submit actions

### Screen 3 --- Journal History and Trends

Display: - Calendar of journal entries - Search and date filters - Mood
trend chart - Frequently selected emotions - User-reported triggers -
Sleep and stress history

### Screen 4 --- Optional Professional Review

If the product later supports psychologists or counsellors, provide a
separate permission-controlled view containing: - Journal entries
explicitly shared by the user - Mood and sleep trends - Recurring themes
with source references - User-reported concerns - Clinician notes, where
appropriate

The professional view should not automatically expose a user's entire
private journal.

------------------------------------------------------------------------

## 4. Weekly Dashboard and Insights

After a user has accumulated enough entries, the application can
generate a weekly report.

### Example dashboard metrics

-   Number of journal entries
-   Average self-reported mood
-   Mood trend during the week
-   Most frequently selected emotions
-   Average sleep duration
-   Stress rating distribution
-   Commonly reported triggers
-   Activities the user reports as helpful

### Example AI observations

-   "Work pressure appeared in three entries alongside anxiety."
-   "Your mood ratings were higher toward the end of the week."
-   "You mentioned exercise as helpful in several entries."

These are observations about self-reported information, not proof of
causation or clinical conclusions.

### Example practical suggestions

-   Try a short relaxation activity after work.
-   Continue recording how you feel after activities you find helpful.
-   Consider whether a recurring stressor is something you can address
    or discuss with someone you trust.

Suggestions should be supportive, non-prescriptive, and proportionate to
the available data.

### Trend interpretation rules

-   Compare equivalent periods, such as one calendar week against the
    previous calendar week.
-   Show the number of entries behind each metric.
-   Make missing data visible; do not silently treat missing values as
    zero.
-   Do not infer improvement from a small or inconsistent sample.
-   Avoid creating an arbitrary composite "mental health score."
-   Do not claim that sleep, exercise, work, or relationships caused a
    mood change based only on correlation.

------------------------------------------------------------------------


------------------------------------------------------------------------

## 6. Database Choice

The initial database recommendation is **MongoDB**, a document-oriented
NoSQL database.

### Why MongoDB is a reasonable fit

The application stores a mixture of: - Free-text journal entries - Mood
ratings - Emotion tags - Triggers - Sleep and stress measurements -
AI-generated summaries and observations - Future report metadata

A document model provides flexibility as the product evolves.

MongoDB does not inherently improve AI analysis. Its benefit here is
flexible document modelling and convenient retrieval of related
journal-entry fields.

### Other options considered

  -----------------------------------------------------------------------
  Database                Data model              Assessment
  ----------------------- ----------------------- -----------------------
  MongoDB                 Document                Recommended starting
                                                  point

  Amazon DynamoDB         Key-value/document      Useful for AWS-native,
                                                  high-scale workloads

  Google Cloud Firestore  Document                Useful for applications
                                                  needing managed
                                                  realtime capabilities

  Cassandra               Wide-column             Not necessary for the
                                                  initial product
                                                  requirements
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 7. Proposed MongoDB Collections

Start with a small number of collections. The following is an initial
logical design, not a final production schema.

### 7.1 `users`

Stores account and basic profile information.

``` json
{
  "_id": "user_101",
  "name": "User A",
  "email": "user@example.com",
  "created_at": "2026-11-01T10:00:00Z"
}
```

Do not store plaintext passwords. Use a secure password-hashing
mechanism or a trusted identity provider. Keep sensitive authentication
data appropriately separated.

### 7.2 `journal_entries`

The primary collection. One document represents one journal entry.

``` json
{
  "_id": "entry_101",
  "user_id": "user_101",
  "entry_date": "2026-11-01",
  "mood": {
    "rating": 3,
    "label": "Okay"
  },
  "emotions": [
    "anxiety",
    "loneliness"
  ],
  "triggers": [
    "work",
    "relationships"
  ],
  "journal_text": "Today was stressful...",
  "sleep": {
    "duration_hours": 6.5,
    "quality": 3
  },
  "stress_level": 4,
  "positive_activities": [
    "gym"
  ],
  "tomorrow_intention": "Take a short walk",
  "created_at": "2026-11-01T20:30:00Z",
  "updated_at": "2026-11-01T20:30:00Z",
  "schema_version": 1
}
```

Design considerations: - Keep each entry in its own document. - Do not
embed an unlimited history of entries in one user document. - Use a
consistent timezone strategy for timestamps. - Decide whether the
application permits multiple entries per day. - Preserve the original
journal text when generating AI summaries. - Validate rating ranges and
field lengths at the API boundary.

### 7.3 `weekly_reports`

Stores generated weekly summaries.

``` json
{
  "_id": "report_101",
  "user_id": "user_101",
  "period": {
    "start_date": "2026-10-26",
    "end_date": "2026-11-01"
  },
  "metrics": {
    "average_mood": 3.2,
    "average_sleep_hours": 6.4,
    "journal_entries_count": 7
  },
  "observations": [],
  "suggestions": [],
  "limitations": [
    "Based on self-reported journal entries."
  ],
  "generated_at": "2026-11-01T21:00:00Z",
  "analysis_version": "1.0",
  "schema_version": 1
}
```

Reports should record the period analysed, the report version, and the
source records or source identifiers used.

### 7.4 `user_goals`

Stores goals voluntarily created by users.

``` json
{
  "_id": "goal_101",
  "user_id": "user_101",
  "title": "Maintain a consistent sleep schedule",
  "status": "active",
  "created_at": "2026-11-01T10:00:00Z"
}
```

### 7.5 `ai_analysis_history`

Stores AI interpretations separately from original journal data.

``` json
{
  "_id": "analysis_101",
  "user_id": "user_101",
  "entry_id": "entry_101",
  "themes": [
    "work_pressure"
  ],
  "summary": "Work pressure was mentioned in this entry.",
  "evidence_entry_ids": [
    "entry_101"
  ],
  "model_version": "configured-model-version",
  "prompt_version": "1.0",
  "created_at": "2026-11-01T21:00:00Z",
  "schema_version": 1
}
```

Do not store a fabricated numerical confidence value. If confidence is
later used, define what it means, how it is calibrated, and how it is
presented.

------------------------------------------------------------------------

## 8. API Design --- Initial Endpoints

Use versioned REST endpoints.

  ----------------------------------------------------------------------------
  Method                  Endpoint                     Purpose
  ----------------------- ---------------------------- -----------------------
  POST                    `/api/v1/journals`           Create a journal entry

  GET                     `/api/v1/journals`           Retrieve journal
                                                       history

  GET                     `/api/v1/journals/{id}`      Retrieve one entry

  PUT/PATCH               `/api/v1/journals/{id}`      Update an entry

  DELETE                  `/api/v1/journals/{id}`      Delete an entry

  POST                    `/api/v1/moods`              Submit a mood record,
                                                       if separate records are
                                                       needed

  GET                     `/api/v1/moods/history`      Retrieve mood history

  POST                    `/api/v1/sleep`              Submit a sleep record,
                                                       if separate records are
                                                       needed

  GET                     `/api/v1/analytics/weekly`   Retrieve weekly
                                                       statistics

  GET                     `/api/v1/reports/weekly`     Retrieve a generated
                                                       weekly report
  ----------------------------------------------------------------------------

The exact endpoint set should follow the final data model. If mood and
sleep are always part of a journal entry, separate write endpoints may
not be necessary initially.

### API engineering requirements

-   Authenticate users.
-   Derive the user identity from the authenticated session/token, not
    from an arbitrary client-supplied `user_id`.
-   Enforce record ownership on every read, update, and delete.
-   Validate input with Pydantic.
-   Add pagination and date-range filters to history endpoints.
-   Use consistent error response formats.
-   Add request IDs and structured logs.
-   Apply rate limits where appropriate.
-   Document the API using OpenAPI/Swagger.

------------------------------------------------------------------------

## 9. Future MCP Tools

Do not build MCP first. Once the backend and analytics services are
stable, expose selected capabilities through MCP.

Potential tools:

  MCP tool                     Responsibility
  ---------------------------- -----------------------------------------------
  `get_user_journals`          Retrieve entries for an authorised date range
  `get_mood_history`           Retrieve mood history
  `get_weekly_statistics`      Retrieve deterministic weekly calculations
  `get_sleep_history`          Retrieve sleep records
  `get_previous_week_report`   Retrieve a prior report
  `save_weekly_insights`       Save a validated generated report
  `get_user_goals`             Retrieve user-defined goals

Every tool must enforce the same user-level permissions as the API. Do
not allow the LLM to access arbitrary user records by choosing an
arbitrary user ID.

------------------------------------------------------------------------

## 10. AI Analysis Strategy

Use two distinct analysis layers.

### Layer A --- Deterministic Python analytics

Use ordinary, testable code for: - Average mood - Week-over-week mood
comparison - Emotion frequency - Sleep averages - Stress distribution -
Journaling consistency - Missing-data detection

The same input data should produce the same numerical results.

### Layer B --- LLM interpretation

Use the LLM for: - Summarising journal context - Identifying recurring
themes - Describing user-reported emotional experiences - Highlighting
possible associations between reported factors - Generating reflection
questions - Suggesting appropriate everyday wellbeing activities

Example: 1. Python calculates that anxiety was selected in four of seven
entries. 2. The LLM reviews the relevant entries. 3. It observes that
work pressure was mentioned alongside anxiety in three entries. 4. The
report includes the observation and references the source entries.

This is more reliable than asking an LLM to calculate all statistics
directly from raw journal text.

### Example report contract

``` json
{
  "period": "2026-W44",
  "summary": "Mood ratings were higher toward the end of the week.",
  "metrics": {
    "average_mood": 3.2,
    "average_sleep_hours": 6.4,
    "journal_entries_count": 7
  },
  "observations": [
    {
      "type": "recurring_theme",
      "title": "Work-related stress",
      "description": "Work pressure appeared in three entries alongside anxiety.",
      "evidence_entry_ids": [
        "entry_102",
        "entry_104",
        "entry_106"
      ]
    }
  ],
  "suggestions": [
    "Consider a short relaxation activity after work."
  ],
  "limitations": [
    "Observations are based on self-reported journal entries."
  ]
}
```

Validate this response against a strict schema before saving or
displaying it. The frontend should render predefined components rather
than arbitrary HTML generated by the model.

------------------------------------------------------------------------

## 11. Single Agent or Multiple Agents?

Start with one orchestrator agent and deterministic Python services.

  Component                  Responsibility
  -------------------------- ---------------------------------------
  Orchestrator               Controls the weekly analysis workflow
  Python analytics service   Calculates numerical trends
  LLM                        Interprets journal content
  MCP server                 Exposes controlled application tools

Multiple independent agents are not necessary for the first version.
They add orchestration complexity, latency, cost, and debugging
overhead. Introduce specialised agents only when a demonstrated
requirement justifies them.

### Recommended initial workflow

1.  User submits journal entries during the week.
2.  A scheduled job or user action triggers weekly analysis.
3.  Backend retrieves the authorised records.
4.  Python analytics calculates metrics.
5.  LLM interprets relevant journal content.
6.  Backend validates the structured report.
7.  Report is saved in `weekly_reports`.
8.  Dashboard retrieves and displays it.

This is a controlled workflow, not an autonomous agent continuously
reading and acting on user data.

------------------------------------------------------------------------

## 12. Technology Stack

  Layer                   Suggested technology
  ----------------------- ---------------------------
  Backend                 FastAPI
  Primary database        MongoDB
  Database driver         Current PyMongo Async API
  Data validation         Pydantic
  Frontend                React
  Backend testing         Pytest + HTTPX
  Containerisation        Docker Compose
  API documentation       FastAPI OpenAPI/Swagger
  Future AI integration   MCP + LLM orchestrator

For a new Python project, prefer the current PyMongo asynchronous API
rather than starting a new implementation on the deprecated Motor
development path.

------------------------------------------------------------------------

## 13. Development Roadmap

Build in the following order.

### Phase 1 --- Database design

-   Finalise collections and document schemas.
-   Define field types, required fields, and validation rules.
-   Define indexes.
-   Decide multiple-entry-per-day behaviour.
-   Define timezone and date-boundary rules.

### Phase 2 --- Backend API

-   Create the FastAPI project.
-   Configure MongoDB connection management.
-   Implement authentication and authorisation.
-   Implement journal CRUD endpoints.
-   Add mood and sleep handling.
-   Add pagination, filtering, and error handling.
-   Write API tests.

### Phase 3 --- Frontend journaling

-   Build the daily dashboard.
-   Build the journal writing screen.
-   Build journal history and editing.
-   Add mood and emotion selection.
-   Connect the frontend to the API.

### Phase 4 --- Deterministic analytics

-   Calculate daily and weekly metrics.
-   Build week-over-week comparisons.
-   Add missing-data handling.
-   Test analytics using known datasets.

### Phase 5 --- MCP integration

-   Expose authorised read tools.
-   Expose validated report-writing tools only when required.
-   Test permissions and failure handling.

### Phase 6 --- AI agent

-   Implement weekly interpretation.
-   Validate structured JSON output.
-   Store report and model/prompt versions.
-   Add evidence references and user feedback.
-   Evaluate output quality before broad release.

------------------------------------------------------------------------

## 14. Security, Privacy, and Safety Requirements

Journal entries may contain highly sensitive personal information. Treat
them accordingly.

-   Encrypt data in transit and at rest.
-   Apply strict user-level access controls.
-   Never expose one user's records to another user.
-   Obtain explicit consent for AI processing and clinician sharing.
-   Keep original journal text separate from AI-generated summaries.
-   Allow users to access, export, correct, and delete their records
    under the product's retention policy.
-   Establish backup and tested recovery procedures.
-   Avoid logging raw journal text or sensitive personal details.
-   Treat journal text as untrusted input; it must never override system
    or agent instructions.
-   Define data retention and deletion behaviour for backups and derived
    AI reports.
-   Keep an audit trail for sensitive access and sharing actions.

### Safety boundary

The product is a wellbeing and journaling tool, not a diagnostic system.
AI observations must not be presented as diagnoses or substitutes for
professional assessment.

If a user describes imminent self-harm or danger, use a separate,
carefully designed safety workflow that directs the user toward
appropriate human or emergency support. Do not rely on a weekly summary
agent to handle emergencies.

------------------------------------------------------------------------

## 15. Key Decisions to Finalise Before Coding

1.  Can a user create more than one journal entry per day?
2.  Are mood, sleep, and stress fields part of each journal entry, or
    can users submit them independently?
3.  Are journal entries private by default? (Recommended: yes.)
4.  Can users share selected entries with a clinician?
5.  What timezone determines the user's daily and weekly reporting
    boundaries?
6.  How long are original entries and AI-derived reports retained?
7.  Will weekly reports be generated automatically or only when
    requested?

------------------------------------------------------------------------

## 16. Final Recommendation

Build a reliable journaling application first.

The initial milestone is not an AI agent. It is a secure application
that can:

1.  Register and authenticate a user.
2.  Create, retrieve, update, and delete journal entries.
3.  Store mood, emotion, sleep, and stress data.
4.  Display journal history.
5.  Calculate accurate weekly statistics.

Once this foundation works and has meaningful test data, introduce MCP
and AI-generated insights.

**The central architectural principle is to keep data storage,
deterministic analytics, AI interpretation, and dashboard presentation
as separate responsibilities.** This will make the application easier to
test, maintain, secure, and extend.
