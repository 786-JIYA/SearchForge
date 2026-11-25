# SearchForge Backend Roadmap & API Design

> **Status:** Proposed engineering plan --- not an implementation
> guarantee\
> **Product direction:** Engineering Knowledge Retrieval Platform\
> **Initial retrieval mode:** Keyword search\
> **Backend stack:** Node.js (CommonJS), Express, PostgreSQL, Prisma,
> Redis and BullMQ (introduced when asynchronous processing is needed)

------------------------------------------------------------------------

## 1. Product purpose

SearchForge helps engineering teams find and reuse internal technical
knowledge, including:

-   Previous production incidents and postmortems
-   Debugging notes and technical solutions
-   API documentation
-   Architecture Decision Records (ADRs)
-   Engineering guides and operational runbooks

The first version focuses on **reliable keyword-based retrieval** with
source traceability, document freshness and permission-aware access.

SearchForge is not initially a web crawler, a general-purpose Google
replacement, or an autonomous AI agent. Semantic retrieval and RAG
answer generation are future extensions, after the keyword retrieval
foundation is measured and reliable.

## 2. Core backend responsibilities

1.  Authenticate users and identify their workspace memberships.
2.  Accept and validate document uploads.
3.  Store document metadata, versions and source-file references.
4.  Extract and normalize text from supported formats.
5.  Process documents asynchronously when processing becomes expensive.
6.  Build and maintain searchable indexes.
7.  Parse search queries, retrieve candidates and rank results.
8.  Enforce authorization before returning titles, snippets, content or
    citations.
9.  Preserve source traceability and document-version information.
10. Expose health, operational and administrative endpoints safely.

## 3. Proposed architecture

``` mermaid
flowchart TD
    UI[React Client] --> API[Express API]
    API --> AUTH[Authentication and Authorization]
    API --> DB[(PostgreSQL)]
    API --> STORE[Document File Storage]
    API --> SEARCH[Retrieval Service]
    API --> QUEUE[BullMQ Queue]
    QUEUE --> WORKER[Document Processing Worker]
    WORKER --> EXTRACT[Text Extraction]
    EXTRACT --> NORMALIZE[Normalization and Tokenization]
    NORMALIZE --> INDEX[Index Builder]
    INDEX --> DB
    INDEX --> SEARCH
    SEARCH --> DB
    SEARCH --> RESPONSE[Ranked Results with Sources]
```

### Component responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Express API                         HTTP routing, request validation
                                      and response formatting

  Authentication                      Verify identity and issue/validate
                                      sessions or tokens

  Authorization                       Check workspace and document-level
                                      permissions

  PostgreSQL                          Users, workspaces, memberships,
                                      documents, versions, jobs and
                                      metadata

  File storage                        Original uploaded files; do not
                                      assume database rows are file
                                      storage

  Queue                               Track background processing jobs
                                      and retries

  Worker                              Extract, normalize, tokenize and
                                      index document content

  Retrieval service                   Parse queries, find candidates,
                                      rank and format results

  Observability                       Structured logs, metrics, health
                                      and failure diagnosis
  -----------------------------------------------------------------------

**Architecture rule:** Keep the first deployment as a modular monolith.
Separate API and worker processes when asynchronous processing is
introduced; do not split into microservices without measured need.

## 4. Suggested repository structure

``` text
SearchForge/
├── README.md
├── docs/
│   ├── requirements/
│   ├── research/
│   ├── architecture/
│   ├── api/
│   │   └── backend-api-roadmap.md
│   ├── algorithms/
│   ├── decisions/
│   └── interview-preparation/
├── backend/
│   ├── prisma/
│   │   └── schema.prisma
│   ├── src/
│   │   ├── config/
│   │   ├── controllers/
│   │   ├── routes/
│   │   ├── middlewares/
│   │   ├── validators/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── engines/
│   │   │   ├── tokenizer/
│   │   │   ├── indexing/
│   │   │   ├── query/
│   │   │   └── ranking/
│   │   ├── queues/
│   │   ├── workers/
│   │   ├── storage/
│   │   ├── utils/
│   │   ├── app.js
│   │   └── server.js
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── fixtures/
│   └── package.json
├── frontend/
├── benchmarks/
└── docker-compose.yml
```

Create directories as the corresponding milestone begins. This is a
target structure, not a requirement to create empty folders immediately.

------------------------------------------------------------------------

# Part I --- Backend implementation roadmap

## 5. Milestone 0: Product and engineering foundation

**Goal:** Agree on what is being built before implementation.

Research and document:

-   Problem statement and target users
-   Functional and non-functional requirements
-   User stories and acceptance criteria
-   Supported document formats and initial size limits
-   Workspace and permission model
-   Data lifecycle: upload, process, search, update, delete
-   Initial performance measurement plan
-   Explicit non-goals and known limitations

**Deliverables**

-   `docs/requirements/problem-statement.md`
-   `docs/requirements/functional-requirements.md`
-   `docs/requirements/non-functional-requirements.md`
-   `docs/architecture/system-context.md`
-   Initial architecture decision records (ADRs)

**Exit criteria:** A reviewer can explain who uses SearchForge, what
problem it solves and what the first release will not attempt to solve.

## 6. Milestone 1: Node.js and Express foundation

**Goal:** Create a reliable, testable HTTP application.

Implement:

-   Node.js CommonJS project setup
-   Express application and separate server startup
-   Environment-variable validation
-   Centralized error handling
-   404 handling
-   Request validation
-   Structured logging
-   Versioned route prefix: `/api/v1`
-   Liveness and readiness health checks
-   Jest and Supertest integration tests

Suggested endpoints:

-   `GET /api/v1/health/live`
-   `GET /api/v1/health/ready`

**Exit criteria:** The server starts from documented instructions,
invalid configuration fails clearly, unknown routes return a consistent
JSON error, and health tests pass.

## 7. Milestone 2: PostgreSQL and data model

**Goal:** Persist users, workspaces and knowledge documents safely.

Research before choosing final schema:

-   Relational modeling and normalization
-   Primary keys, foreign keys and unique constraints
-   Transactions and isolation
-   Index selection and query plans
-   Workspace tenancy boundaries
-   Document versioning
-   Permission inheritance and explicit grants

Initial conceptual entities:

  Entity               Purpose
  -------------------- -------------------------------------------
  User                 Account identity
  Workspace            Team or organization boundary
  Membership           User's role in a workspace
  Document             Logical knowledge resource
  DocumentVersion      Immutable or traceable revision
  DocumentChunk        Searchable text segment
  DocumentPermission   Optional document-specific access
  ProcessingJob        Ingestion status and retry metadata
  SearchHistory        Optional, privacy-conscious query history
  AuditEvent           Security-sensitive actions

Do not automatically persist every token occurrence as a relational row.
Compare database storage, in-memory structures and rebuild strategies
through experiments.

**Exit criteria:** Migrations work on a clean database, constraints are
tested, and workspace isolation is demonstrated by integration tests.

## 8. Milestone 3: Authentication and authorization

**Goal:** Protect every private engineering document.

Implement:

-   Registration and login
-   Password hashing using a suitable password-hashing algorithm
-   Session or token lifecycle and logout/revocation strategy
-   Authentication middleware
-   Workspace membership checks
-   Role-based permissions
-   Document-level access checks where required
-   Audit events for sensitive operations

Security requirements:

-   Never store plaintext passwords.
-   Never commit secrets or real `.env` files.
-   Validate authorization on the server, not only in React.
-   Do not reveal document existence, title, snippet or citation to
    unauthorized users.
-   Apply authorization to search results before returning content.
-   Re-check access when permissions or memberships change.

**Exit criteria:** Tests prove that a user in one workspace cannot
access another workspace's documents or search snippets.

## 9. Milestone 4: Document ingestion and storage

**Goal:** Accept documents and track their lifecycle.

Implement in small steps:

1.  Define supported formats for the first release.
2.  Validate file type, size and upload metadata.
3.  Generate safe server-side storage identifiers.
4.  Store the original file outside the public web root.
5.  Create document and version records.
6.  Track states such as `UPLOADED`, `PROCESSING`, `READY` and `FAILED`.
7.  Add duplicate detection using a content hash where appropriate.
8.  Define update, delete and reprocessing behavior.

Security considerations:

-   Treat filenames and MIME types supplied by clients as untrusted.
-   Enforce upload-size limits.
-   Do not execute uploaded content.
-   Consider malware scanning before production use.
-   Prevent path traversal and public access to private files.

**Exit criteria:** Valid files are stored and tracked; invalid files are
rejected; failures are visible and recoverable.

## 10. Milestone 5: Text extraction and normalization

**Goal:** Convert supported files into predictable text.

Research and implement:

-   Format-specific extraction adapters
-   UTF-8 and Unicode handling
-   Whitespace normalization
-   Case normalization
-   Punctuation policy
-   Token boundaries
-   Stop-word policy (do not remove terms without measuring their
    impact)
-   Chunking and source offsets
-   Extraction errors and empty documents

Keep the original document and extracted representation traceable to a
specific document version.

**Exit criteria:** A test corpus produces deterministic extracted text
and tokens, including edge cases such as empty files, punctuation,
Unicode and repeated whitespace.

## 11. Milestone 6: Search engine core

**Goal:** Build a correct keyword retrieval engine before optimizing it.

Implementation sequence:

1.  Tokenizer
2.  Normalizer
3.  In-memory inverted index
4.  Posting lists
5.  Positional information
6.  Boolean AND and OR
7.  Phrase search
8.  Query validation and parsing
9.  Ranking baseline
10. Index update and deletion behavior

Research and compare:

-   Linear scan vs inverted index
-   Term-document incidence matrix
-   Inverted index and postings
-   Positional indexes
-   TF, DF and IDF
-   TF-IDF
-   BM25
-   Index memory and build-time costs

**Important:** A basic inverted index is an initial retrieval
implementation, not automatically a production-scale search engine.
Benchmark before making performance claims.

**Exit criteria:** Unit tests cover exact matching, AND/OR, phrase
queries, empty queries, repeated terms, document updates and deletions.

## 12. Milestone 7: Asynchronous processing

**Goal:** Prevent large document processing tasks from blocking API
requests.

Introduce Redis and BullMQ only when the synchronous prototype
demonstrates the need.

Implement:

-   Queue producer
-   Worker process
-   Job status tracking
-   Retry policy with backoff
-   Idempotent processing
-   Failed-job handling and recovery
-   Concurrency limits
-   Graceful shutdown
-   Duplicate-job protection

**Exit criteria:** A worker restart or retry does not create duplicate
or inconsistent index entries, and failed jobs can be diagnosed.

## 13. Milestone 8: Search relevance and source traceability

**Goal:** Return useful results that engineers can verify.

Implement:

-   Candidate retrieval
-   Ranking strategy
-   Result snippets
-   Document title and version
-   Source location or chunk reference
-   Matched terms
-   Freshness metadata
-   Permission filtering
-   Stable pagination

Compare TF-IDF and BM25 using a labeled evaluation set. Measure
relevance rather than choosing a ranking method only because it is
popular.

**Exit criteria:** Search results show the correct source and version,
respect permissions, and can be evaluated against known relevant
results.

## 14. Milestone 9: Production quality

Implement and verify:

-   Unit, integration and end-to-end tests
-   API contract documentation
-   Rate limiting
-   Secure headers and CORS policy
-   Input validation
-   SQL and file-upload safety
-   Structured logs
-   Metrics and tracing where useful
-   Database backups and restore procedure
-   Docker-based local environment
-   CI workflow
-   Deployment and operational documentation

**Exit criteria:** A clean environment can be built from instructions,
tests run in CI, and operational failures have documented recovery
steps.

## 15. Milestone 10: Benchmarks and evaluation

Build repeatable datasets and record actual results.

Measure:

-   Index construction time
-   Index memory usage
-   Query latency: P50, P95 and P99
-   Throughput
-   Documents processed per second
-   Queue wait time and processing duration
-   Search relevance (precision/recall or an appropriate ranking metric)
-   Update and deletion cost

Compare linear scan and indexed retrieval on the same corpus, machine
and query set. Record dataset size, hardware, runtime version, warm-up
procedure and repetitions.

Never publish fabricated or unrepeatable performance claims.

------------------------------------------------------------------------

# Part II --- Proposed API reference

## 16. API conventions

**Base path:** `/api/v1`

**Content type:** `application/json` for JSON requests and responses.
Multipart upload endpoints use `multipart/form-data`.

**Authentication:** Protected endpoints require the chosen
authentication mechanism. The examples below use:

``` http
Authorization: Bearer <access_token>
```

This is a proposed contract; cookie-based sessions are also a valid
design and must be decided before implementation.

### Common response conventions

Successful single-resource response:

``` json
{
  "data": {
    "id": "doc_123",
    "status": "READY"
  }
}
```

List response:

``` json
{
  "data": [],
  "pagination": {
    "limit": 20,
    "nextCursor": null
  }
}
```

Error response:

``` json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request is invalid.",
    "details": []
  }
}
```

Use appropriate HTTP status codes:

  Status   Meaning
  -------- ---------------------------------------------------------
  200      Successful read or update
  201      Resource created
  202      Accepted for asynchronous processing
  204      Successful operation with no response body
  400      Malformed or invalid request
  401      Missing or invalid authentication
  403      Authenticated but not allowed
  404      Resource not found or intentionally concealed
  409      Conflict, such as duplicate or invalid state transition
  413      Upload too large
  415      Unsupported media type
  422      Valid syntax but unprocessable content
  429      Rate limit exceeded
  500      Unexpected server error

## 17. Health endpoints

  Method   Endpoint          Auth   Purpose
  -------- ----------------- ------ ---------------------------------
  GET      `/health/live`    No     Process is alive
  GET      `/health/ready`   No     Required dependencies are ready

Example:

``` http
GET /api/v1/health/live
```

``` json
{
  "data": {
    "status": "alive"
  }
}
```

Readiness may check PostgreSQL and other mandatory dependencies. Avoid
exposing secrets, connection strings or detailed infrastructure
information.

## 18. Authentication endpoints

  Method   Endpoint           Purpose
  -------- ------------------ --------------------------------------
  POST     `/auth/register`   Create an account
  POST     `/auth/login`      Authenticate
  POST     `/auth/logout`     Revoke session/token where supported
  GET      `/auth/me`         Get current user

### Register

``` http
POST /api/v1/auth/register
Content-Type: application/json
```

``` json
{
  "name": "Asha",
  "email": "asha@example.com",
  "password": "use-a-strong-secret"
}
```

Expected result: `201 Created`.

### Login

``` http
POST /api/v1/auth/login
```

``` json
{
  "email": "asha@example.com",
  "password": "use-a-strong-secret"
}
```

Return the chosen session/token response without exposing password
hashes.

## 19. Workspace endpoints

  --------------------------------------------------------------------------------------------
  Method                  Endpoint                                     Purpose
  ----------------------- -------------------------------------------- -----------------------
  POST                    `/workspaces`                                Create workspace

  GET                     `/workspaces`                                List user's workspaces

  GET                     `/workspaces/:workspaceId`                   Workspace details

  PATCH                   `/workspaces/:workspaceId`                   Update workspace

  GET                     `/workspaces/:workspaceId/members`           List members

  POST                    `/workspaces/:workspaceId/invitations`       Invite member

  PATCH                   `/workspaces/:workspaceId/members/:userId`   Change member role

  DELETE                  `/workspaces/:workspaceId/members/:userId`   Remove member
  --------------------------------------------------------------------------------------------

Every endpoint must enforce workspace membership and role requirements.

## 20. Document endpoints

  ----------------------------------------------------------------------------------------------
  Method                  Endpoint                                       Purpose
  ----------------------- ---------------------------------------------- -----------------------
  POST                    `/workspaces/:workspaceId/documents`           Upload document

  GET                     `/workspaces/:workspaceId/documents`           List accessible
                                                                         documents

  GET                     `/documents/:documentId`                       Get document metadata

  PATCH                   `/documents/:documentId`                       Update metadata

  DELETE                  `/documents/:documentId`                       Delete or archive
                                                                         document

  GET                     `/documents/:documentId/versions`              List versions

  GET                     `/documents/:documentId/versions/:versionId`   Version details

  POST                    `/documents/:documentId/reprocess`             Request reprocessing

  GET                     `/documents/:documentId/status`                Processing status
  ----------------------------------------------------------------------------------------------

### Upload document

``` http
POST /api/v1/workspaces/ws_123/documents
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

Multipart fields (proposed):

-   `file`: document binary
-   `title`: optional display title
-   `sourceType`: e.g. `INCIDENT`, `API_DOC`, `ADR`, `RUNBOOK`,
    `TECHNICAL_NOTE`
-   `visibility`: workspace or restricted, according to the permission
    model

If processing is asynchronous, return `202 Accepted`:

``` json
{
  "data": {
    "documentId": "doc_123",
    "versionId": "ver_001",
    "status": "UPLOADED",
    "jobId": "job_456"
  }
}
```

### Document lifecycle

``` text
UPLOADED
   |
   v
PROCESSING
   |
   +----> READY
   |
   +----> FAILED
```

Define explicit transitions for reprocessing, deletion and cancellation.
Do not treat a `202 Accepted` response as proof that indexing has
completed.

## 21. Search endpoints

  -----------------------------------------------------------------------------------------------
  Method                  Endpoint                                        Purpose
  ----------------------- ----------------------------------------------- -----------------------
  GET                     `/workspaces/:workspaceId/search`               Keyword search

  GET                     `/workspaces/:workspaceId/search/suggestions`   Optional query
                                                                          suggestions

  GET                     `/search/:searchId`                             Optional saved
                                                                          search/result retrieval
  -----------------------------------------------------------------------------------------------

### Keyword search

``` http
GET /api/v1/workspaces/ws_123/search?q=database%20connection%20timeout&limit=10
Authorization: Bearer <access_token>
```

Proposed query parameters:

  Parameter        Purpose
  ---------------- -----------------------------------------
  `q`              Search text; required
  `limit`          Page size with a server-defined maximum
  `cursor`         Cursor for pagination
  `sourceType`     Optional source-type filter
  `updatedAfter`   Optional freshness filter
  `sort`           Supported ranking or date ordering

Example response:

``` json
{
  "data": [
    {
      "documentId": "doc_123",
      "versionId": "ver_004",
      "title": "Database Connection Incident",
      "sourceType": "INCIDENT",
      "score": 0.87,
      "snippet": "The service experienced a database connection timeout...",
      "matchedTerms": [
        "database",
        "connection",
        "timeout"
      ],
      "source": {
        "chunkId": "chunk_009",
        "page": 2
      },
      "updatedAt": "2026-11-20T10:00:00.000Z"
    }
  ],
  "pagination": {
    "limit": 10,
    "nextCursor": null
  }
}
```

The score shown is illustrative, not a defined scoring formula. The
actual ranking algorithm must be documented and evaluated.

**Critical authorization rule:** Search must never return unauthorized
document titles, snippets, matched terms or source references. Decide
whether filtering occurs during candidate retrieval or through a secure
post-retrieval authorization stage, and measure the performance
implications.

## 22. Document permission endpoints

  ---------------------------------------------------------------------------------------------------
  Method                  Endpoint                                            Purpose
  ----------------------- --------------------------------------------------- -----------------------
  GET                     `/documents/:documentId/permissions`                View permissions

  PUT                     `/documents/:documentId/permissions`                Replace permission set

  POST                    `/documents/:documentId/permissions`                Grant access

  DELETE                  `/documents/:documentId/permissions/:principalId`   Revoke access
  ---------------------------------------------------------------------------------------------------

These endpoints are needed only if the selected design supports
document-specific ACLs. Workspace-only access may be sufficient for the
first prototype.

Permission changes must take effect immediately at the authorization
layer, even if an index update is still pending.

## 23. Processing and operational endpoints

  Method   Endpoint                             Purpose
  -------- ------------------------------------ ----------------------------
  GET      `/jobs/:jobId`                       Processing job status
  POST     `/documents/:documentId/reprocess`   Requeue document
  GET      `/admin/index/status`                Index health and metadata
  POST     `/admin/index/rebuild`               Trigger controlled rebuild
  GET      `/admin/metrics`                     Operational metrics

Admin endpoints require explicit authorization and audit logging. Do not
expose these endpoints to ordinary workspace members.

------------------------------------------------------------------------

# Part III --- API and engineering standards

## 24. Validation rules

Every endpoint should define:

-   Required and optional fields
-   Data types and maximum lengths
-   Allowed enum values
-   Authentication requirements
-   Authorization requirements
-   Success status and response schema
-   Possible error codes
-   Rate limits where applicable
-   Tests for invalid input and edge cases

Use a schema-validation library if it improves consistency. Validate
path parameters, query parameters, JSON bodies and uploaded files.

## 25. API versioning and compatibility

-   Keep routes under `/api/v1`.
-   Avoid breaking response contracts silently.
-   Document deprecated fields and endpoints.
-   Use consistent error codes.
-   Add pagination to list/search endpoints.
-   Set maximum page sizes.
-   Avoid returning internal database records directly.

## 26. Testing strategy

  -----------------------------------------------------------------------
  Test level                          Examples
  ----------------------------------- -----------------------------------
  Unit                                Tokenizer, postings intersection,
                                      query parser, ranking

  Integration                         API + PostgreSQL, authorization,
                                      upload lifecycle

  Worker                              Retries, idempotency, failed jobs

  Security                            Cross-workspace access, permission
                                      revocation, malicious uploads

  Performance                         Index build time, query latency,
                                      throughput

  Contract                            Request/response schema and
                                      status-code verification
  -----------------------------------------------------------------------

Every feature issue should include acceptance criteria and tests. Bugs
should become regression tests.

## 27. Suggested environment variables

Names are illustrative; add only those required by the current
milestone.

``` dotenv
NODE_ENV=development
PORT=5000

DATABASE_URL=
DIRECT_URL=

# Add only after selecting an authentication strategy.
# AUTH_SECRET=

# Add when asynchronous jobs are introduced.
# REDIS_URL=

# Add when file storage is configured.
# STORAGE_DRIVER=
# STORAGE_PATH=
```

Keep real values in an ignored `.env` file or a secret manager. Commit
an `.env.example` containing placeholders only.

## 28. Recommended development order

    Order Deliverable                                  Depends on
  ------- -------------------------------------------- ---------------------------
        1 Requirements and architecture                Product definition
        2 Express foundation and health checks         1
        3 PostgreSQL connection and initial schema     1--2
        4 Authentication and workspace authorization   3
        5 Document metadata and upload                 3--4
        6 Text extraction and normalization            5
        7 Tokenizer and in-memory inverted index       Research and 6
        8 Boolean and phrase search                    7
        9 Search ranking and source references         8
       10 Background workers and retries               5--6
       11 Persistent/rebuildable indexing strategy     7, 10
       12 Frontend integration                         Stable API contracts
       13 Security, observability and deployment       Core features
       14 Benchmarks and optimization                  Working end-to-end system

This order is flexible where dependencies allow parallel work, but avoid
introducing infrastructure before the problem requires it.

## 29. GitHub issue and pull-request workflow

Use this cycle for every meaningful deliverable:

1.  Create a GitHub Issue describing the problem.
2.  Add acceptance criteria and tests.
3.  Create a focused branch.
4.  Research and document important decisions.
5.  Implement a small change.
6.  Run tests and inspect the diff.
7.  Open a Pull Request linked to the issue.
8.  Review the code and documentation.
9.  Merge only when acceptance criteria pass.

Suggested branch names:

``` text
research/inverted-index
docs/api-contract
feat/health-endpoints
feat/document-upload
feat/keyword-search
test/search-integration
fix/permission-filtering
```

Suggested Conventional Commit messages:

``` text
docs: define backend architecture
docs: document document upload API
feat: add application health endpoints
feat: create workspace document metadata
test: cover cross-workspace authorization
perf: optimize postings list intersection
```

------------------------------------------------------------------------

# Part IV --- Definition of done

A backend feature is not complete merely because its happy path works.

Before closing an issue, verify:

-   [ ] The requirement and user value are clear.
-   [ ] The API contract is documented.
-   [ ] Authentication and authorization are enforced.
-   [ ] Inputs and failure states are validated.
-   [ ] Unit or integration tests are present.
-   [ ] Logs do not expose secrets or private document content.
-   [ ] Database and index consistency are considered.
-   [ ] Documentation is updated.
-   [ ] The PR is reviewed and merged.
-   [ ] Known limitations are recorded.

## 30. Important scope decisions to confirm

These decisions affect the database, API and infrastructure. Keep them
open until research supports a choice.

  -----------------------------------------------------------------------
  Decision                            Initial recommendation
  ----------------------------------- -----------------------------------
  Product scope                       Engineering knowledge retrieval

  First search mode                   Keyword

  Initial architecture                Modular monolith

  Database                            PostgreSQL

  ORM                                 Prisma

  Search index                        Implement an in-memory
                                      inverted-index prototype first

  Background processing               Add Redis/BullMQ after ingestion
                                      prototype

  File storage                        Local development adapter, then
                                      choose production storage

  Access control                      Workspace membership first; add
                                      document ACL if needed

  AI/RAG                              Defer until retrieval quality is
                                      measured

  Similarity detection                Separate future capability, not the
                                      first release blocker
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Final engineering principle

SearchForge should be built in layers:

**Understand the problem → research the technology → document the design
→ define the API → implement → test → benchmark → review.**

Treat this document as a living roadmap. When research changes a
decision, update the relevant section and record important architectural
changes in an ADR. The endpoint contracts above are proposed designs,
not claims that those APIs already exist.
