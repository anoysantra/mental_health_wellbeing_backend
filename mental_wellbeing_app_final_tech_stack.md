# Mental Wellbeing Journaling App --- Finalized Tech Stack

**Status:** Finalized\
**Architecture:** React frontend + FastAPI backend + MongoDB\
**Purpose:** A private journaling application with user-triggered
AI-assisted wellbeing reports.

------------------------------------------------------------------------

## 1. Technology Stack

  -----------------------------------------------------------------------
  Layer                   Technology              Responsibility
  ----------------------- ----------------------- -----------------------
  Frontend                React                   Component-based user
                                                  interface

  Frontend tooling        Vite                    Development server and
                                                  production builds

  UI styling              Tailwind CSS            Responsive styling and
                                                  layout

  Client-side routing     React Router            Navigation between
                                                  application pages

  API communication       Axios                   HTTP requests from
                                                  React to FastAPI

  Charts                  Recharts                Mood, sleep and journal
                                                  trend visualizations

  Backend                 FastAPI                 REST API, request
                                                  handling and
                                                  application
                                                  orchestration

  Data validation         Pydantic                Request/response
                                                  schemas and structured
                                                  AI-output validation

  Database                MongoDB                 Users, journal entries,
                                                  analysis jobs and
                                                  reports

  MongoDB driver          PyMongo                 Database access and
                                                  queries

  AI orchestration        Microsoft Agent         Agent and workflow
                          Framework               orchestration

  Tool integration        Model Context Protocol  Controlled tools
                          (MCP)                   exposed to the agent

  Background processing   Celery + Redis          Durable asynchronous
                                                  analysis jobs and task
                                                  coordination

  Backend testing         Pytest                  Unit and integration
                                                  tests

  Frontend / end-to-end   Playwright              Browser-based workflow
  testing                                         testing

  Containers              Docker                  Reproducible
                                                  application and service
                                                  environments

  Deployment              AWS EC2                 Application hosting

  Reverse proxy / web     Nginx                   HTTPS termination,
  server                                          routing and static
                                                  asset delivery

  CI/CD                   Jenkins or GitHub       Automated testing and
                          Actions                 deployment pipeline
  -----------------------------------------------------------------------

### Stack decisions

-   Use **React**, not server-rendered Django templates.
-   Use **FastAPI** as the backend framework for both API endpoints and
    application logic.
-   Use **MongoDB** as the primary application database, accessed
    through **PyMongo**.
-   Use **Pydantic** for API contracts and validation.
-   Use **Microsoft Agent Framework** for AI orchestration and **MCP**
    for controlled tool access.
-   Run potentially long AI analysis outside the HTTP request lifecycle
    using a background worker.
-   Use **Pytest** and **Playwright** for automated testing.
-   Package services with **Docker** and deploy to **AWS EC2**, with
    **Nginx** in front of the application.

------------------------------------------------------------------------

## 2. High-Level Architecture

``` text
                         USER
                          |
                          v
                 REACT FRONTEND
             React + Vite + Tailwind
                          |
                    HTTP / JSON
                          |
                          v
                  FASTAPI BACKEND
          Routes | Auth | Pydantic Schemas
                          |
                          v
                 SERVICE LAYER
                 Business Logic
                          |
                          v
               REPOSITORY LAYER
                  PyMongo Access
                          |
                          v
                      MONGODB
       Users | Journals | Jobs | Mind Reports


          USER-TRIGGERED AI ANALYSIS
                          |
                          v
              FastAPI Analysis Endpoint
                          |
                          v
              Background Task Queue
                    Celery + Redis
                          |
                          v
                 WORKER PROCESS
                          |
                          v
             MICROSOFT AGENT FRAMEWORK
                          |
                          v
                    MCP TOOLS
                          |
                          v
               Authorized Data Access
                          |
                          v
             Python Metrics + LLM Analysis
                          |
                          v
               Pydantic Output Validation
                          |
                          v
                 Save Report to MongoDB
                          |
                          v
             React retrieves report/status
```

The diagram represents logical responsibilities. The worker, API server,
MCP server and MongoDB may run as separate processes or containers.

------------------------------------------------------------------------

## 3. Frontend

### Selected technologies

-   **React:** Build reusable UI components and application pages.
-   **Vite:** Bootstrap, run and build the React application.
-   **Tailwind CSS:** Implement responsive layouts and styling.
-   **React Router:** Handle routes such as login, dashboard, journal
    and reports.
-   **Axios:** Communicate with FastAPI endpoints.
-   **Recharts:** Display mood and wellbeing trends.

### Initial frontend pages

1.  Landing page
2.  Registration
3.  Login
4.  Dashboard
5.  New journal entry
6.  Journal history
7.  Journal entry details / edit
8.  Mood and wellbeing trends
9.  Mind Report
10. Account settings

React is responsible for presentation and client-side interaction. It
must not connect directly to MongoDB or contain secrets, LLM credentials
or privileged business logic.

------------------------------------------------------------------------

## 4. Backend

FastAPI is the single backend framework.

### Responsibilities

-   Expose REST endpoints.
-   Authenticate and authorize users.
-   Validate incoming requests and outgoing response structures.
-   Apply application and business rules.
-   Call service and repository functions.
-   Queue long-running AI analysis tasks.
-   Return consistent JSON responses and appropriate HTTP status codes.

### Backend layering

  Layer          Responsibility
  -------------- -----------------------------------------------------
  Routes         HTTP endpoints; parse requests and return responses
  Schemas        Pydantic request and response validation
  Services       Business rules and application workflows
  Repositories   MongoDB queries and persistence
  Database       MongoDB client and connection lifecycle
  Agents         Microsoft Agent Framework workflows
  MCP server     Narrow, authorized tools for agent access
  Workers        Execute background jobs

**Important:** Keep database queries out of route handlers. Routes
should call services; services should use repositories. This keeps
business logic testable and reusable.

------------------------------------------------------------------------

## 5. Database

### MongoDB + PyMongo

MongoDB is the primary persistence layer. PyMongo provides direct access
to collections, queries, indexes and aggregation pipelines.

Initial collections:

-   `users`
-   `journal_entries`
-   `analysis_jobs`
-   `mind_reports`
-   `user_preferences` (if required later)

### Data-access principles

-   Every user-owned record must be associated with an authenticated
    user identifier.
-   Enforce ownership checks in backend services/repositories; never
    trust a user ID supplied by the browser or an LLM.
-   Add indexes for frequent access patterns, such as user ID and entry
    date.
-   Validate data at the API boundary and enforce domain rules in
    services.
-   Define retention, export and deletion behaviour before production.
-   Do not log raw journal content or sensitive AI prompts by default.

------------------------------------------------------------------------

## 6. AI Analysis Architecture

AI analysis is **user-triggered**, not automatically run after every
journal submission.

### Workflow

1.  The user creates journal entries.
2.  The user selects **Analyze My Mind Report**.
3.  React sends a request to a FastAPI analysis endpoint.
4.  FastAPI authenticates the user and checks for an existing active
    analysis job.
5.  FastAPI creates a job record and queues the work.
6.  A worker runs the Microsoft Agent Framework workflow.
7.  The agent uses approved MCP tools to retrieve the user's permitted
    journal and wellbeing data.
8.  Python calculates deterministic metrics, such as mood averages and
    entry counts.
9.  The LLM produces a structured interpretation grounded in the
    available data.
10. Pydantic validates the output; backend checks that cited source
    entries belong to the user and requested period.
11. The report is saved to MongoDB and the job is marked complete.
12. React checks job status and displays the completed report.

### Job states

-   `queued`
-   `processing`
-   `completed`
-   `failed`

### Design constraints

-   Do not run long LLM operations inside an ordinary HTTP request.
-   Start with one agent and a small set of well-defined tools; add
    multi-agent orchestration only when there is a demonstrated need.
-   Use deterministic Python code for calculations. Use the LLM for
    language interpretation and reflective summaries, not arithmetic
    that can be computed reliably.
-   MCP tools must enforce authenticated-user data boundaries.
-   Treat journal text as untrusted input and defend against prompt
    injection.
-   Validate structured model output before persisting or displaying it.
-   The application is a journaling and wellbeing reflection tool, not a
    diagnostic or clinical decision system.

------------------------------------------------------------------------

## 7. API Design

Initial endpoint groups (exact paths and payload schemas will be
finalized during implementation):

  -----------------------------------------------------------------------
  Area                                Example operation
  ----------------------------------- -----------------------------------
  Authentication                      Register, login, logout, current
                                      user

  Journals                            Create, list, retrieve, update and
                                      delete entries

  Trends                              Retrieve mood and wellbeing data
                                      for charts

  Mind Reports                        Start analysis, check job status,
                                      retrieve latest report and report
                                      history

  Preferences                         Read and update user settings
  -----------------------------------------------------------------------

Use versioned API routes, for example `/api/v1/...`.

The frontend and backend should agree on request/response contracts.
Keep those contracts documented and test them. Do not let frontend
components depend on undocumented database structures.

------------------------------------------------------------------------

## 8. Security and Privacy

Journal entries can contain highly personal information. Privacy and
access control are core requirements, not later enhancements.

-   Use HTTPS in deployed environments.
-   Store secrets and credentials in environment variables or a managed
    secrets solution.
-   Never commit `.env` files or API keys.
-   Use secure authentication and password hashing.
-   Apply authorization checks to every user-owned resource.
-   Configure CORS for only the intended frontend origins.
-   Validate and limit request sizes and inputs.
-   Avoid logging journal text, passwords, access tokens or sensitive
    prompts.
-   Protect AI-provider credentials on the backend only.
-   Define user consent for AI processing and explain what data is sent
    to the model provider.
-   Support user data export and deletion.
-   Restrict MongoDB, Redis and internal MCP services to private network
    access.
-   Back up data and periodically test restoration.

Authentication details (for example, secure cookie sessions versus
access and refresh tokens) will be selected and documented before
implementation. Do not assume that storing long-lived tokens in browser
local storage is an acceptable default.

------------------------------------------------------------------------

## 9. Testing Strategy

### Pytest

Use Pytest for:

-   Unit tests for services and business rules.
-   Repository tests against a test database.
-   API tests for request validation and response contracts.
-   Authentication and authorization tests.
-   Cross-user access-denial tests.
-   Analysis-job lifecycle and duplicate-job tests.
-   Pydantic report-schema validation.
-   MCP tool authorization tests.

### Playwright

Use Playwright for end-to-end workflows:

-   Registration and login.
-   Create, edit and delete journal entries.
-   View journal history and trends.
-   Start an AI report.
-   Observe queued and processing states.
-   View a completed report.
-   Verify that one user cannot access another user's data.

Use synthetic test journal data. Do not use real private journal entries
in routine test runs.

------------------------------------------------------------------------

## 10. Deployment

### Target deployment layout

-   Nginx: public HTTPS entry point and reverse proxy.
-   React: production-built static assets served through Nginx or a
    static hosting service.
-   FastAPI: application API service.
-   MongoDB: private database service or managed MongoDB deployment.
-   Redis: private task broker.
-   Celery worker: asynchronous analysis execution.
-   MCP server: internal service/process accessible only to authorized
    components.

Docker should provide reproducible environments for the services. AWS
EC2 is the initial hosting target; managed services can be considered
later where they reduce operational burden.

### CI/CD

A pipeline should progressively automate:

1.  Dependency installation.
2.  Formatting and static checks.
3.  Pytest unit and integration tests.
4.  Frontend build and checks.
5.  Playwright end-to-end tests in an appropriate test environment.
6.  Docker image build.
7.  Deployment after the required checks pass.

Jenkins or GitHub Actions can implement this pipeline. Choose one
initially rather than maintaining both without a need.

------------------------------------------------------------------------

## 11. Implementation Order

Build and validate one layer at a time.

  Phase   Deliverable
  ------- -----------------------------------------------------------
  1       Repository setup, React/Vite frontend and FastAPI backend
  2       MongoDB connection, configuration and repository pattern
  3       Authentication and authorization
  4       Journal CRUD APIs and React journal screens
  5       Dashboard, journal history and mood visualizations
  6       Background queue and analysis-job lifecycle
  7       MCP server and narrowly scoped data tools
  8       Microsoft Agent Framework workflow
  9       Validated Mind Report persistence and UI
  10      Pytest and Playwright coverage
  11      Docker, CI/CD and AWS EC2 deployment
  12      Security review, backup/restore and operational hardening

Do not build all layers simultaneously. First prove the
React--FastAPI--MongoDB path with authentication and journal CRUD. Add
the AI workflow only after the core application is reliable.

------------------------------------------------------------------------

## 12. Final Architecture Decision

The project will use:

**React + Vite + Tailwind CSS + FastAPI + Pydantic + MongoDB + PyMongo +
Microsoft Agent Framework + MCP + Celery + Redis + Pytest + Playwright +
Docker + AWS EC2 + Nginx.**

Django and Django REST Framework are not part of the selected
architecture. The frontend will not use Django templates. FastAPI is the
sole application backend.

This is the finalized baseline. Individual library versions and exact
API contracts will be pinned and documented when implementation begins.
