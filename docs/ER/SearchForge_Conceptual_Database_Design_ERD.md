# SearchForge --- Conceptual Database Design & ERD

> **Status:** Conceptual ERD draft\
> **Purpose:** Document the proposed database entities, relationships,
> cardinalities, constraints, and lifecycle design before creating
> Prisma models or migrations.\
> **Database:** PostgreSQL\
> **ORM:** Prisma 7\
> **Backend:** Node.js (CommonJS) + Express\
> **Frontend:** React\
> **Background processing:** Redis + BullMQ\
> **File storage:** Local filesystem in development; Cloudflare R2 in
> production

------------------------------------------------------------------------

## 1. Product context

SearchForge is an **Engineering Knowledge Retrieval Platform**. It helps
engineering teams retrieve and reuse:

-   Production incidents and postmortems
-   Technical solutions and debugging notes
-   API documentation
-   Architecture Decision Records (ADRs)
-   Runbooks
-   Engineering guides

The platform is designed around source traceability, document freshness,
and permission-aware retrieval.

The initial search experience is keyword-based and uses a custom Node.js
inverted index persisted in PostgreSQL. Grounded RAG/LLM and bounded
agentic capabilities are future stages; retrieval quality and evaluation
come first.

### Core architecture decisions

-   PostgreSQL is the primary relational database.
-   Prisma 7 is the ORM. Database connection configuration belongs in
    `prisma.config.ts`; do not assume the legacy Prisma datasource URL
    syntax.
-   Original uploaded files are stored in object storage: local
    filesystem in development and Cloudflare R2 in production.
-   Extracted text is stored in PostgreSQL for every document version.
-   Redis and BullMQ handle asynchronous processing.
-   One global document index is used, with search results filtered by
    active workspace membership and document-workspace relationships.
-   The ERD is designed and reviewed before writing Prisma models or
    migrations.

------------------------------------------------------------------------

## 2. Design principles

### 2.1 User and workspace membership are separate

A user can belong to multiple workspaces, and a workspace can have
multiple users. `WorkspaceMember` is the junction entity that stores
membership role and status.

### 2.2 Logical documents and versions are separate

A `Document` is the long-lived logical record. Each uploaded revision is
a separate `DocumentVersion`, with its own original file reference,
checksum, extracted text, and processing/indexing state.

### 2.3 Workspace relationships are separate from documents

`WorkspaceDocument` links a logical document to a workspace. Removing
the relationship from one workspace must not remove the document from
other workspaces.

### 2.4 Malware scanning, extraction, and indexing have independent lifecycles

A file can be malware-safe and readable even when indexing has failed.
Such a document can be available for authorized viewing/download after
scanning passes, while remaining excluded from search until indexing
succeeds.

### 2.5 Latest extracted version and active searchable version are different concepts

Extraction may succeed while indexing fails. Therefore, the design must
distinguish:

-   The latest successfully extracted version.
-   The currently active searchable version.

The previous active index remains available until a new index generation
is successfully built and atomically activated.

### 2.6 Deletion is a controlled lifecycle, not an immediate cascade

Workspace and document deletion use soft deletion, recovery windows,
background purge jobs, retries, and resumable cleanup. Purge workers
must re-check resource state before destructive operations.

------------------------------------------------------------------------

## 3. Entity inventory

The entities are grouped into eight subsystems. Some entities are
proposed supporting entities and will need validation during the
detailed ERD review.

  -----------------------------------------------------------------------
  Subsystem                           Entities
  ----------------------------------- -----------------------------------
  Identity and authentication         `User`, `Session`,
                                      `EmailVerificationToken`,
                                      `PasswordResetToken`,
                                      `EmailChangeRequest`

  Workspace and access                `Workspace`, `WorkspaceMember`,
                                      `WorkspaceInvitation`,
                                      `WorkspaceAuditLog`

  Document management                 `Document`, `DocumentVersion`,
                                      `WorkspaceDocument`,
                                      `DocumentUploadAttempt`

  Document organization               `WorkspaceCategory`,
                                      `WorkspaceTag`,
                                      `WorkspaceDocumentTag`,
                                      `DocumentRelationship`

  Processing and indexing             `MalwareScanJob`,
                                      `MalwareScanAttempt`,
                                      `DocumentProcessingJob`,
                                      `DocumentProcessingAttempt`,
                                      `IndexingJob`, `IndexingAttempt`,
                                      `DocumentIndexGeneration`,
                                      `InvertedIndexEntry`

  Search and personalization          `SearchHistory`, `SavedSearch`,
                                      `Bookmark`, `SearchFeedback`,
                                      proposed aggregate-feedback storage

  Deletion and notifications          `Notification`,
                                      `EmailNotification`,
                                      `EmailNotificationRecipient`,
                                      `DocumentPurgeJob`,
                                      `DocumentPurgeAttempt`,
                                      `WorkspacePurgeJob`,
                                      `WorkspacePurgeStage`,
                                      `WorkspacePurgeTask`,
                                      `WorkspacePurgeAttempt`,
                                      `WorkspacePurgeSummary`

  Platform administration             `PlatformAuditLog`,
                                      `PlatformSecurityEvent`, proposed
                                      `RetentionPolicy` and platform
                                      configuration
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 4. ERD --- Identity and workspace access

``` mermaid
erDiagram
    USER {
        uuid id PK
        string email UK
        string passwordHash
        string name
        string jobTitle
        string bio
        string platformRole
        string accountStatus
        datetime emailVerifiedAt
        datetime deletedAt
        datetime createdAt
    }

    SESSION {
        uuid id PK
        uuid userId FK
        string tokenHash
        string deviceInfo
        datetime expiresAt
        datetime revokedAt
        datetime createdAt
    }

    EMAIL_VERIFICATION_TOKEN {
        uuid id PK
        uuid userId FK
        string tokenHash
        datetime expiresAt
        datetime usedAt
    }

    PASSWORD_RESET_TOKEN {
        uuid id PK
        uuid userId FK
        string tokenHash
        datetime expiresAt
        datetime usedAt
    }

    EMAIL_CHANGE_REQUEST {
        uuid id PK
        uuid userId FK
        string newEmail
        string tokenHash
        datetime expiresAt
        datetime usedAt
    }

    WORKSPACE {
        uuid id PK
        string name
        string description
        uuid createdById FK
        datetime deletedAt
        uuid deletedById FK
        datetime scheduledPurgeAt
        datetime createdAt
    }

    WORKSPACE_MEMBER {
        uuid id PK
        uuid workspaceId FK
        uuid userId FK
        string role
        string status
        datetime joinedAt
        datetime leftAt
        datetime removedAt
        uuid removedById FK
    }

    WORKSPACE_INVITATION {
        uuid id PK
        uuid workspaceId FK
        string email
        uuid invitedById FK
        string tokenHash
        string status
        datetime expiresAt
        datetime acceptedAt
        datetime revokedAt
    }

    WORKSPACE_AUDIT_LOG {
        uuid id PK
        uuid workspaceId FK
        uuid actorId FK
        string action
        string outcome
        string targetType
        uuid targetId
        json metadata
        datetime createdAt
    }

    USER ||--o{ SESSION : owns
    USER ||--o{ EMAIL_VERIFICATION_TOKEN : verifies
    USER ||--o{ PASSWORD_RESET_TOKEN : resets
    USER ||--o{ EMAIL_CHANGE_REQUEST : requests
    USER ||--o{ WORKSPACE : creates
    USER ||--o{ WORKSPACE : deletes
    USER ||--o{ WORKSPACE_MEMBER : joins
    WORKSPACE ||--o{ WORKSPACE_MEMBER : has_members
    USER ||--o{ WORKSPACE_INVITATION : sends
    WORKSPACE ||--o{ WORKSPACE_INVITATION : receives
    USER o|--o{ WORKSPACE_AUDIT_LOG : performs
    WORKSPACE ||--o{ WORKSPACE_AUDIT_LOG : records
```

### Responsibilities and constraints

-   `User.email` is globally unique and normalized to lowercase.
-   Email reuse is permitted only after permanent account purge.
-   Store password hashes only; use Argon2id.
-   Email verification tokens and password reset tokens store hashes,
    not raw tokens.
-   Email verification token validity: 24 hours.
-   Password reset token validity: 1 hour.
-   Refresh-token session maximum lifetime: 30 days; rotate on each use.
-   Password change requires the current password and revokes other
    active sessions.
-   Email change is verified before replacing the old email. Keep the
    current session and revoke other sessions.
-   A user may have multiple independent active sessions.
-   Platform role is separate from workspace role. Public signup creates
    a normal user, not a platform admin.
-   Workspace roles: `OWNER`, `ADMIN`, `MEMBER`.
-   Workspace membership statuses: `ACTIVE`, `REMOVED`, `LEFT`.
-   Membership history is retained; rejoining creates a new membership
    period.
-   At most one active membership per user/workspace.
-   At most one active OWNER per workspace.
-   The OWNER role is held in `WorkspaceMember`; the workspace creator
    field preserves the original creator and is not the current-owner
    pointer.
-   Only active members have workspace access.
-   Invitations are accepted only by an authenticated user whose
    verified email matches the invitation email.
-   Invitation statuses: `PENDING`, `ACCEPTED`, `EXPIRED`, `REVOKED`.
-   Invitation validity: 7 days. Resending revokes the old invitation
    and creates a new record/token.
-   Only OWNER/ADMIN can invite and revoke pending invitations; only
    OWNER manages workspace admins and ownership transfer.
-   Workspace audit actor may be nullable because personal data can
    later be anonymized or erased.

### Workspace audit log

`WorkspaceAuditLog` is append-only and records important successful
actions and security-sensitive denied attempts, not ordinary validation
errors. It includes actor, action, outcome, target, metadata, and
timestamp. Metadata must not contain secrets, tokens, full document
content, or unnecessary snapshots.

OWNER/ADMIN can view and export workspace audit logs. Logs are
paginated; exports support CSV/JSON with streaming, bounded result
sizes, stable ordering, and CSV formula-injection protection. Export
actions themselves are audited.

------------------------------------------------------------------------

## 5. ERD --- Documents, versions, and organization

``` mermaid
erDiagram
    USER {
        uuid id PK
        string name
    }

    WORKSPACE {
        uuid id PK
        string name
    }

    DOCUMENT {
        uuid id PK
        string title
        string description
        string status
        uuid createdById FK
        datetime deletedAt
        datetime purgeAfter
        datetime createdAt
    }

    DOCUMENT_VERSION {
        uuid id PK
        uuid documentId FK
        int versionNumber
        string versionLabel
        string fileKey
        string sha256
        string format
        bigint fileSize
        text extractedText
        string extractionStatus
        string indexingStatus
        uuid uploadedById FK
        datetime createdAt
    }

    WORKSPACE_DOCUMENT {
        uuid id PK
        uuid workspaceId FK
        uuid documentId FK
        uuid categoryId FK
        uuid deletedById FK
        datetime deletedAt
        datetime purgeAfter
        datetime createdAt
    }

    DOCUMENT_UPLOAD_ATTEMPT {
        uuid id PK
        uuid workspaceId FK
        uuid uploadedById FK
        string sha256
        string originalFileKey
        string status
        datetime retentionUntil
        datetime createdAt
    }

    WORKSPACE_CATEGORY {
        uuid id PK
        uuid workspaceId FK
        string name
        datetime createdAt
    }

    WORKSPACE_TAG {
        uuid id PK
        uuid workspaceId FK
        string name
        datetime createdAt
    }

    WORKSPACE_DOCUMENT_TAG {
        uuid id PK
        uuid workspaceDocumentId FK
        uuid tagId FK
    }

    DOCUMENT_RELATIONSHIP {
        uuid id PK
        uuid workspaceId FK
        uuid sourceDocumentId FK
        uuid targetDocumentId FK
        uuid sourceVersionId FK
        uuid targetVersionId FK
        string relationshipType
        uuid createdById FK
        datetime createdAt
    }

    WORKSPACE ||--o{ WORKSPACE_DOCUMENT : links
    DOCUMENT ||--o{ WORKSPACE_DOCUMENT : appears_in
    DOCUMENT ||--|{ DOCUMENT_VERSION : has_versions
    USER o|--o{ DOCUMENT : creates
    USER o|--o{ DOCUMENT_VERSION : uploads
    WORKSPACE ||--o{ DOCUMENT_UPLOAD_ATTEMPT : receives
    USER o|--o{ DOCUMENT_UPLOAD_ATTEMPT : uploads
    WORKSPACE ||--o{ WORKSPACE_CATEGORY : defines
    WORKSPACE_CATEGORY o|--o{ WORKSPACE_DOCUMENT : classifies
    WORKSPACE ||--o{ WORKSPACE_TAG : defines
    WORKSPACE_DOCUMENT ||--o{ WORKSPACE_DOCUMENT_TAG : has
    WORKSPACE_TAG ||--o{ WORKSPACE_DOCUMENT_TAG : applied_to
    WORKSPACE ||--o{ DOCUMENT_RELATIONSHIP : contains
    DOCUMENT ||--o{ DOCUMENT_RELATIONSHIP : source
    DOCUMENT ||--o{ DOCUMENT_RELATIONSHIP : target
    DOCUMENT_VERSION ||--o{ DOCUMENT_RELATIONSHIP : source_version
    DOCUMENT_VERSION ||--o{ DOCUMENT_RELATIONSHIP : target_version
    USER o|--o{ DOCUMENT_RELATIONSHIP : creates
```

### Document model

-   Supported formats: PDF, DOCX, TXT, MD, HTML, PPTX.
-   Every replacement creates a new immutable `DocumentVersion`.
-   Version numbers are sequential; an optional user-facing label is
    allowed.
-   Concurrent version uploads must be serialized so only one upload
    becomes the next version; another upload must retry against the
    latest version.
-   Original files are stored in R2 in production and local filesystem
    in development.
-   Extracted text is stored in PostgreSQL for every version.
-   SHA-256 duplicate detection is scoped to the same workspace.
-   Duplicate content upload is rejected, even if filename/title
    differs; show that an identical document exists.
-   A version replacement is a distinct operation and is allowed only to
    OWNER/ADMIN of a linked workspace.
-   A new version is shared across all workspaces linked to the logical
    document.
-   Only the latest completed version is searchable. Historical versions
    remain read-only and are not searchable.
-   All active members of linked workspaces can view documents; there
    are no per-document access restrictions.
-   All active members can edit document metadata.
-   Any active member can remove the document's relationship from a
    workspace.
-   A logical document is soft-deleted when its last active workspace
    relationship is removed; users do not directly delete the underlying
    logical document.
-   Orphaned logical documents are recoverable for 30 days.
-   If a user leaves/is removed from a workspace, documents and uploader
    attribution remain; the workspace retains access.
-   Account deletion preserves shared documents and replaces uploader
    attribution with an anonymized "Deleted user" reference.

### WorkspaceDocument

`WorkspaceDocument` is the workspace-specific association. It supports:

-   Soft removal and restoration within a 30-day recovery period.
-   Removal by an active member.
-   Restoration by the remover, OWNER, or ADMIN.
-   One optional category per workspace relationship.
-   Workspace-specific tags.
-   Immediate search visibility removal in the affected workspace when
    the relationship is removed, while preserving visibility in other
    workspaces.

### Categories and tags

-   Categories are workspace-specific; OWNER/ADMIN manage them.
-   Tags are workspace-specific; all members can create/apply tags,
    while OWNER/ADMIN can rename/delete them.
-   Deleting a category or tag removes its associations immediately but
    does not delete documents.
-   Enforce normalized unique names within a workspace and unique tag
    associations.

### Document relationships

Supported types: `RESOLVES`, `REFERENCES`, `DOCUMENTS`, `RELATED_TO`.

-   Relationships are workspace-scoped.
-   They can be directional or mutual.
-   Multiple distinct relationship types may exist between the same pair
    of documents.
-   The relationship preserves the exact document versions used when it
    was created.
-   Store creator and timestamp.
-   Prevent self-referencing relationships.
-   Enforce uniqueness for the same type and direction.
-   For mutual relationships, normalize document ordering to prevent
    storing both A→B and B→A as duplicate mutual links.

### Upload failures

`DocumentUploadAttempt` is proposed to retain a failed/rejected upload
without creating a searchable logical `Document`. This supports the rule
that if all extraction attempts fail, the original file may be retained
for troubleshooting for 30 days, then purged. The exact boundary between
an upload attempt and a created document version must be finalized in
the detailed ERD.

------------------------------------------------------------------------

## 6. ERD --- Malware scanning, processing, and indexing

``` mermaid
erDiagram
    DOCUMENT_VERSION {
        uuid id PK
        uuid documentId FK
        string extractionStatus
        string indexingStatus
    }

    DOCUMENT_UPLOAD_ATTEMPT {
        uuid id PK
        uuid workspaceId FK
        string sha256
        string status
        string originalFileKey
    }

    MALWARE_SCAN_JOB {
        uuid id PK
        uuid uploadAttemptId FK
        uuid documentVersionId FK
        string status
        int attemptCount
        datetime createdAt
    }

    MALWARE_SCAN_ATTEMPT {
        uuid id PK
        uuid malwareScanJobId FK
        int attemptNumber
        string status
        string safeError
        datetime startedAt
        datetime completedAt
    }

    DOCUMENT_PROCESSING_JOB {
        uuid id PK
        uuid documentVersionId FK
        string status
        int attemptCount
        datetime createdAt
    }

    DOCUMENT_PROCESSING_ATTEMPT {
        uuid id PK
        uuid processingJobId FK
        int attemptNumber
        string status
        string safeError
        datetime startedAt
        datetime completedAt
    }

    INDEXING_JOB {
        uuid id PK
        uuid documentVersionId FK
        uuid generationId FK
        string status
        int attemptCount
        datetime createdAt
    }

    INDEXING_ATTEMPT {
        uuid id PK
        uuid indexingJobId FK
        int attemptNumber
        string status
        string safeError
        datetime startedAt
        datetime completedAt
    }

    DOCUMENT_INDEX_GENERATION {
        uuid id PK
        uuid documentVersionId FK
        string status
        datetime createdAt
        datetime activatedAt
        datetime retiredAt
    }

    INVERTED_INDEX_ENTRY {
        uuid id PK
        uuid generationId FK
        uuid documentVersionId FK
        string term
        int termFrequency
        json positions
    }

    DOCUMENT_UPLOAD_ATTEMPT ||--o{ MALWARE_SCAN_JOB : scanned_by
    DOCUMENT_VERSION o|--o{ MALWARE_SCAN_JOB : scan_result_for
    MALWARE_SCAN_JOB ||--o{ MALWARE_SCAN_ATTEMPT : records
    DOCUMENT_VERSION ||--o{ DOCUMENT_PROCESSING_JOB : processed_by
    DOCUMENT_PROCESSING_JOB ||--o{ DOCUMENT_PROCESSING_ATTEMPT : records
    DOCUMENT_VERSION ||--o{ INDEXING_JOB : indexed_by
    INDEXING_JOB ||--o{ INDEXING_ATTEMPT : records
    DOCUMENT_VERSION ||--o{ DOCUMENT_INDEX_GENERATION : has
    DOCUMENT_INDEX_GENERATION ||--o{ INVERTED_INDEX_ENTRY : contains
    DOCUMENT_VERSION ||--o{ INVERTED_INDEX_ENTRY : indexed_for
    DOCUMENT_INDEX_GENERATION o|--o{ INDEXING_JOB : built_by
```

### Processing lifecycle

1.  Upload and validate format, MIME type, extension, and configured
    size limit.
2.  Perform malware scanning.
3.  If malicious, quarantine the file. Platform admins alone can inspect
    quarantined files.
4.  If the scanner is unavailable, keep the file quarantined until
    scanning succeeds.
5.  Scanner retries are bounded; after automatic attempts, platform
    admins can manually initiate another scan.
6.  If scan passes, extract text asynchronously.
7.  Extraction has three total automatic attempts, including the initial
    attempt.
8.  If extraction exhausts its attempts, reject the upload. Keep the
    original file for 30 days for troubleshooting, then purge. No
    searchable document is created.
9.  Store extracted text in PostgreSQL.
10. Build an index generation separately.
11. Indexing has three total automatic attempts.
12. If indexing fails after automatic retries, mark it `FAILED`;
    OWNER/ADMIN of a linked workspace can manually retry indexing.
13. A document whose extraction succeeded but indexing failed can be
    viewed/downloaded after malware scanning passes, but remains
    excluded from search.
14. Activate the new index generation atomically only after it is
    complete. Keep the previous active index available until activation
    succeeds.

### Index generation model

-   Index entries belong to a specific `DocumentVersion`.
-   Only the active generation for the currently searchable version is
    used for retrieval.
-   Full index rebuilds construct a separate generation; the old active
    generation remains available until the replacement is ready.
-   A single global document index is used. Search results are filtered
    through active workspace membership and active document-workspace
    relationships.
-   Metadata changes update searchable metadata immediately. Content is
    re-indexed only when content changes.
-   Removing a workspace relationship immediately removes search
    visibility in that workspace, without affecting other workspaces.
-   The initial tokenizer lowercases and tokenizes words and removes
    punctuation. It does not remove stop words, stem, or perform Unicode
    normalization in the initial baseline.
-   Default matching is OR, with an option to require all query terms.
-   Repeated query terms are ignored initially.
-   Ranking baseline: TF-IDF with a fixed title-match boost. BM25 is a
    later experiment.
-   Relevance descending, then newest upload as tie-breaker.
-   Filters use AND across different filter types and OR among multiple
    values of the same filter.
-   Search supports type, category, tags, and uploader filters.
-   Freshness is displayed but not used in ranking.
-   One short snippet is shown, expandable, with matching terms
    highlighted.
-   No-results state is a simple no-results message; do not expose
    global vocabulary suggestions.
-   User chooses current workspace or all accessible workspaces; default
    is current workspace.
-   Sort choices: relevance, newest upload, oldest upload.
-   Default page size: 10. A maximum page size and maximum total result
    window must be configured; exact values remain open.
-   Search result traceability includes title, workspace, uploader,
    upload date, exact version, and page/section where available.
-   Search explains matching keywords and title matches.

------------------------------------------------------------------------

## 7. ERD --- Search and personalization

``` mermaid
erDiagram
    USER {
        uuid id PK
        string email
    }

    WORKSPACE {
        uuid id PK
        string name
    }

    DOCUMENT {
        uuid id PK
        string title
    }

    DOCUMENT_VERSION {
        uuid id PK
        uuid documentId FK
        int versionNumber
    }

    SEARCH_HISTORY {
        uuid id PK
        uuid userId FK
        string query
        string matchMode
        uuid workspaceId FK
        datetime createdAt
    }

    SAVED_SEARCH {
        uuid id PK
        uuid userId FK
        string name
        string query
        json filters
        datetime createdAt
    }

    BOOKMARK {
        uuid id PK
        uuid userId FK
        uuid documentId FK
        datetime createdAt
    }

    SEARCH_FEEDBACK {
        uuid id PK
        uuid userId FK
        uuid documentVersionId FK
        string feedback
        datetime createdAt
        datetime updatedAt
    }

    USER ||--o{ SEARCH_HISTORY : performs
    WORKSPACE o|--o{ SEARCH_HISTORY : scopes
    USER ||--o{ SAVED_SEARCH : saves
    USER ||--o{ BOOKMARK : creates
    DOCUMENT ||--o{ BOOKMARK : bookmarked
    USER ||--o{ SEARCH_FEEDBACK : submits
    DOCUMENT_VERSION ||--o{ SEARCH_FEEDBACK : receives
```

### Search personalization rules

-   Search history is private to the user, retained for 90 days, and can
    be deleted individually or entirely.
-   Saved searches are personal. Their names are generated from the
    query and can be edited.
-   Saved-search retention is not yet specified.
-   Bookmarks reference the logical document and open the latest
    completed version.
-   When a document is permanently purged, its bookmarks are deleted.
    Saved searches remain and must handle unavailable results
    gracefully.
-   A user has one feedback vote per exact document version and can
    change the vote.
-   Workspace OWNER/ADMIN and platform admins see aggregate feedback
    counts only, not individual voters.
-   Individual feedback rows are deleted when the document is
    permanently purged.
-   Anonymized aggregate feedback is retained indefinitely. The
    aggregate storage mechanism and grouping dimensions still need final
    design.

------------------------------------------------------------------------

## 8. ERD --- Notifications, email, and deletion workflows

``` mermaid
erDiagram
    USER {
        uuid id PK
        string email
    }

    WORKSPACE {
        uuid id PK
        string name
        datetime deletedAt
        datetime scheduledPurgeAt
    }

    DOCUMENT {
        uuid id PK
        string title
        datetime deletedAt
        datetime purgeAfter
    }

    NOTIFICATION {
        uuid id PK
        uuid userId FK
        string type
        string title
        string message
        string resourceType
        uuid resourceId
        json metadata
        datetime readAt
        datetime createdAt
    }

    EMAIL_NOTIFICATION {
        uuid id PK
        uuid workspaceId FK
        uuid preparedById FK
        string eventType
        string subject
        string status
        int recipientCount
        datetime firstOpenedAt
        datetime sentReportedAt
    }

    EMAIL_NOTIFICATION_RECIPIENT {
        uuid id PK
        uuid emailNotificationId FK
        uuid userId FK
        string recipientEmail
        datetime retentionUntil
    }

    DOCUMENT_PURGE_JOB {
        uuid id PK
        uuid documentId FK
        string status
        datetime scheduledAt
        datetime completedAt
    }

    DOCUMENT_PURGE_ATTEMPT {
        uuid id PK
        uuid purgeJobId FK
        int attemptNumber
        string status
        string safeError
        datetime startedAt
        datetime completedAt
    }

    WORKSPACE_PURGE_JOB {
        uuid id PK
        uuid workspaceId FK
        string status
        int currentRun
        datetime scheduledAt
        datetime completedAt
    }

    WORKSPACE_PURGE_STAGE {
        uuid id PK
        uuid purgeJobId FK
        string stageName
        string status
    }

    WORKSPACE_PURGE_TASK {
        uuid id PK
        uuid stageId FK
        string taskType
        string status
    }

    WORKSPACE_PURGE_ATTEMPT {
        uuid id PK
        uuid purgeJobId FK
        int attemptNumber
        int runNumber
        string status
        datetime startedAt
        datetime completedAt
    }

    WORKSPACE_PURGE_SUMMARY {
        uuid id PK
        uuid originalWorkspaceId
        string workspaceName
        string finalStatus
        int totalAttempts
        datetime deletedAt
        datetime completedAt
    }

    USER ||--o{ NOTIFICATION : receives
    WORKSPACE ||--o{ EMAIL_NOTIFICATION : prepares
    USER ||--o{ EMAIL_NOTIFICATION : prepares
    EMAIL_NOTIFICATION ||--o{ EMAIL_NOTIFICATION_RECIPIENT : contains
    USER o|--o{ EMAIL_NOTIFICATION_RECIPIENT : recipient
    DOCUMENT ||--o| DOCUMENT_PURGE_JOB : has
    DOCUMENT_PURGE_JOB ||--o{ DOCUMENT_PURGE_ATTEMPT : records
    WORKSPACE ||--o| WORKSPACE_PURGE_JOB : has
    WORKSPACE_PURGE_JOB ||--o{ WORKSPACE_PURGE_STAGE : contains
    WORKSPACE_PURGE_STAGE ||--o{ WORKSPACE_PURGE_TASK : contains
    WORKSPACE_PURGE_JOB ||--o{ WORKSPACE_PURGE_ATTEMPT : records
    WORKSPACE_PURGE_JOB ||--o| WORKSPACE_PURGE_SUMMARY : produces
```

### Notifications

-   In-app notifications are generated for uploads, new versions, and
    workspace-document relationship removal/restoration.
-   All active members of the affected workspace except the actor
    receive the notification.
-   Processing failures are not broadcast to all members.
-   Notifications are retained for 90 days, but are deleted earlier when
    the related resource is permanently purged.
-   Notification records must not block resource purge; use a deliberate
    cleanup/anonymization strategy for resource references.

### Gmail Compose email workflow

The platform does not send email automatically through SMTP/Nodemailer.
It prepares a Gmail Compose URL for the deleting OWNER to send manually.

-   One `EmailNotification` is created for a workspace deletion event.
-   All active workspace members are included in BCC.
-   The recipient snapshot is immutable.
-   Only the deleting OWNER can access the recovery dashboard and
    initiate this deletion email workflow.
-   Opening Gmail Compose is an explicit user action; opening does not
    prove the email was sent.
-   The OWNER manually marks the preparation as sent.
-   The OWNER can reopen Compose multiple times using the same
    preparation and can cancel an unsent preparation with confirmation.
-   Only the first Compose opening is recorded.
-   Unsent `PREPARED` records remain indefinitely.
-   Statuses: `PREPARED`, `OPENED`, `SENT_REPORTED`, `CANCELLED`.
-   Recipient email snapshots are deleted 90 days after workspace purge.
-   Anonymized preparation history is retained indefinitely.
-   If the original deleting OWNER is suspended/deleted, another active
    OWNER or platform admin can restore the workspace, but the current
    email rule still restricts initiation to the original deleting
    OWNER. This can make the email workflow unavailable in that case and
    remains a policy consideration.

### Document purge

-   A logical document is soft-deleted when its last active workspace
    relationship is removed.
-   Recovery period: 30 days.
-   Permanent purge removes all versions, original files, extracted
    text, and index entries.
-   Purge uses bounded automatic retries. Platform admins can manually
    resume with a required reason and preserved attempt history.
-   Purge must coordinate with workspace purge and must not remove a
    logical document that remains linked to another workspace.
-   The detailed stage/task structure for document purge still needs to
    be added.

### Workspace purge

-   Workspace soft-delete is allowed immediately, regardless of active
    member count.
-   Recovery period: 30 days.
-   The workspace becomes inaccessible immediately, including documents,
    messages, attachments, audit views, and sockets.
-   Existing invitations retain their original token and expiry clock;
    they cannot be accepted while the workspace is deleted. After
    restoration, they are usable only if still unexpired.
-   Restore preserves membership/roles, messages, documents, extracted
    text, index, invitation states/expiry, and files.
-   Original deleting OWNER can restore under the original rule; if that
    OWNER is suspended/deleted, platform admin or another active OWNER
    may restore.
-   Restoration cancels pending purge and restores access
    transactionally.
-   Purge job is created transactionally with deletion. A periodic
    database scanner recovers missing queue jobs.
-   Worker revalidates workspace state and deadline before every
    destructive step.
-   Restoration must fail clearly after the defined irreversible point,
    such as deletion of original files.
-   Workspace purge is stage/task based and resumable. R2 objects
    already absent count as successful cleanup.
-   Parent purge job completes only when all required stages/tasks
    complete.
-   Automatic retries use bounded exponential backoff. When exhausted,
    mark `FAILED` and alert platform admins.
-   Manual admin resume requires admin ID, timestamp, mandatory reason,
    a fresh retry budget, preserved prior attempts, and protection
    against concurrent resumes.
-   Detailed attempt history is retained for 90 days after successful
    cleanup; compact summaries and admin interventions are retained
    indefinitely. Failed jobs are not automatically removed.
-   `WorkspacePurgeSummary` has no cascading FK to the deleted
    workspace. It stores a compact anonymized summary, not document
    content, credentials, tokens, signed URLs, or raw sensitive errors.
-   Purge summaries are platform-admin-only and support filtered
    CSV/JSON exports. Export actions are audited.

------------------------------------------------------------------------

## 9. ERD --- Platform administration and security

``` mermaid
erDiagram
    USER {
        uuid id PK
        string platformRole
    }

    PLATFORM_AUDIT_LOG {
        uuid id PK
        uuid actorId FK
        string action
        string outcome
        string targetType
        uuid targetId
        json metadata
        datetime createdAt
    }

    PLATFORM_SECURITY_EVENT {
        uuid id PK
        uuid userId FK
        string eventType
        string outcome
        string ipAddress
        json metadata
        datetime createdAt
    }

    USER o|--o{ PLATFORM_AUDIT_LOG : performs
    USER o|--o{ PLATFORM_SECURITY_EVENT : involved_in
```

### Platform administration

-   Platform admins are separate from workspace OWNER/ADMIN roles.
-   Only platform admins can globally suspend/reactivate users.
-   Platform admins can inspect quarantined files and manage malware
    rescans.
-   Platform admins can inspect extraction/indexing failures and perform
    allowed recovery actions.
-   Platform admins can rebuild the full index or selected documents.
-   Platform-wide metrics include users, workspaces, documents, uploads,
    searches, storage, processing/indexing failures, and job statistics.
-   Platform-wide audit records important administrative/security
    actions.
-   Platform security events include authentication events,
    authorization denials, and suspicious access.
-   Endpoint-specific rate limits and user/IP protections are required.
-   Platform audit retention is configurable by event type; MVP default
    values and legal requirements still need confirmation.

------------------------------------------------------------------------

## 10. Key cardinalities

  ------------------------------------------------------------------------
  Relationship            Cardinality             Meaning
  ----------------------- ----------------------- ------------------------
  User --- Workspace      Many-to-many            Through
                                                  `WorkspaceMember`

  Workspace ---           1-to-many               A workspace can have
  Invitation                                      many invitations

  Workspace --- Document  Many-to-many            Through
                                                  `WorkspaceDocument`

  Document ---            1-to-many               One logical document has
  DocumentVersion                                 multiple versions

  WorkspaceDocument ---   Many-to-one, optional   At most one category per
  Category                                        workspace relationship

  WorkspaceDocument ---   Many-to-many            Through
  Tag                                             `WorkspaceDocumentTag`

  Document --- Document   Many-to-many            Through
                                                  `DocumentRelationship`

  DocumentVersion ---     1-to-many               Each version can have
  IndexGeneration                                 index generations

  IndexGeneration ---     1-to-many               Each generation contains
  IndexEntry                                      many terms

  User --- SearchHistory  1-to-many               Private search history

  User --- SavedSearch    1-to-many               Personal saved searches

  User --- Bookmark       1-to-many               Bookmarks of logical
                                                  documents

  DocumentVersion ---     1-to-many               Feedback tied to an
  Feedback                                        exact version
  ------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Important database constraints and invariants

These rules must be represented in the detailed ERD and enforced through
database constraints, transactions, or backend authorization as
appropriate.

1.  `User.email` is globally unique and normalized.
2.  Email can be reused only after permanent account purge.
3.  At most one active membership per user/workspace.
4.  At most one active OWNER per workspace.
5.  Exactly one active OWNER must remain for a live workspace; ownership
    transfer is transactional.
6.  One version number per logical document.
7.  Concurrent version creation must be serialized.
8.  At most one active searchable index generation per logical document.
9.  A new index generation is activated only after it is complete.
10. No duplicate category names within a workspace after normalization.
11. No duplicate tag names within a workspace after normalization.
12. No duplicate workspace-document-tag association.
13. No self-referencing document relationship.
14. No duplicate relationship of the same type and direction.
15. One feedback vote per user per exact document version.
16. No access through deleted workspace or inactive membership.
17. Search visibility requires both active workspace membership and an
    active workspace-document relationship.
18. A document purge cannot delete a document still linked to another
    workspace.
19. Workspace restoration and purge must be coordinated to prevent
    races.
20. Purge operations must be idempotent and resumable.
21. Original file deletion defines an irreversible restoration boundary;
    the exact stage must be finalized.
22. An absent R2 object during retry is treated as successful deletion.
23. Audit metadata must not contain secrets or full document content.
24. User deletion must anonymize shared-document attribution and
    preserve legally required records.

Some of these require PostgreSQL partial unique indexes or transactional
safeguards rather than ordinary Prisma `@@unique` declarations.

------------------------------------------------------------------------

## 12. Outstanding design decisions

The conceptual ERD is not yet the final implementation schema. The
following items must be resolved or explicitly documented before Prisma
model design.

  --------------------------------------------------------------------------
  \#                      Open item               Why it matters
  ----------------------- ----------------------- --------------------------
  1                       Maximum search page     Bounds query cost and
                          size and maximum total  permission-filter work. A
                          result window           maximum total window
                                                  around 200 was suggested
                                                  but not formally
                                                  confirmed.

  2                       Malware quarantine      Determines when
                          retention duration      quarantined original files
                                                  are automatically purged
                                                  if not investigated.

  3                       Malware scan retry      Defines scanner recovery
                          count/backoff and       behavior.
                          manual rescan policy    

  4                       Document purge retry    Needed for bounded retries
                          count/backoff           and admin resume behavior.

  5                       Platform audit          Retention is configurable,
                          retention defaults by   but MVP defaults and legal
                          event type              requirements need
                                                  confirmation.

  6                       Failed extraction       `DocumentUploadAttempt` is
                          upload representation   proposed so the retained
                                                  original file does not
                                                  create a searchable
                                                  document/version.

  7                       Exact irreversible      Restore must fail after
                          purge point             original files have been
                                                  deleted.

  8                       Saved-search retention  Search history is 90 days,
                                                  but saved-search retention
                                                  is unspecified.

  9                       Email preparation       Determine which fields
                          anonymization details   remain indefinitely and
                                                  which identifying fields
                                                  are scrubbed.

  10                      Deletion email access   Restoration permission is
                          if original deleting    broader, but email
                          OWNER is                initiation remains
                          suspended/deleted       restricted to the original
                                                  deleting OWNER.

  11                      Document relationship   The design proposes
                          version pinning         workspace-scoped
                                                  relationships that retain
                                                  exact source/target
                                                  versions; confirm this
                                                  interpretation.

  12                      Duplicate upload versus Ensure duplicate content
                          version replacement     rejection does not block
                                                  the explicit authorized
                                                  version-replacement flow.

  13                      Coordination between    A workspace purge must not
                          document purge and      delete a logical document
                          workspace purge         still used by another
                                                  workspace.

  14                      Notification references Notifications are deleted
                          during resource purge   when their related
                                                  resource is permanently
                                                  purged; cleanup must avoid
                                                  blocking purge.

  15                      Aggregate feedback      Decide grouping dimensions
                          storage                 and ensure no
                                                  user/document identifiers
                                                  remain in indefinite
                                                  aggregate data.

  16                      Document purge          Add resumable stage/task
                          stages/tasks            entities analogous to
                                                  workspace purge.

  17                      Retention and           Add a `RetentionPolicy`
                          upload-size             and/or platform
                          configuration           configuration
                                                  representation if
                                                  administrators must change
                                                  these values.

  18                      Extraction manual retry Resolve the earlier admin
                          policy                  retry wording against the
                                                  later
                                                  three-attempt-and-reject
                                                  rule.
  --------------------------------------------------------------------------

### Extraction retry clarification

The current interpretation is:

-   Extraction gets three automatic attempts total, including the
    initial attempt.
-   After exhaustion, the upload is rejected.
-   Indexing failure has a separate manual retry path for workspace
    OWNER/ADMIN.
-   An earlier requirement allowed platform admins to inspect processing
    failures and retry them, while the later extraction decision says
    rejected uploads do not have manual extraction retry.

This conflict must be explicitly resolved before implementation. Do not
silently assume the earlier rule overrides the later one.

------------------------------------------------------------------------

## 13. Recommended next steps

The database work should proceed in this order:

1.  Confirm the outstanding policy decisions that affect entity
    structure.
2.  Refine the entity inventory and add missing purge/configuration
    entities.
3.  Validate every relationship and cardinality against the
    requirements.
4.  Add field-level attributes, nullability, uniqueness, indexes, and
    deletion behavior.
5.  Produce the consolidated final ERD.
6.  Perform a requirements-to-ERD traceability review.
7.  Only after approval, translate the ERD into Prisma 7 models.
8.  Review migrations and database constraints before implementing
    application services.

**Do not start Prisma schema or migrations until the ERD and its
outstanding decisions have been reviewed and approved.**

------------------------------------------------------------------------

## 14. Status

**Current status: Conceptual ERD draft completed.**

This document captures the proposed entity groups, core relationships,
major lifecycle workflows, constraints, and unresolved design questions.
It is a design artifact, not an executable database schema. The next
milestone is to refine and validate the ERD before implementing Prisma
models.
