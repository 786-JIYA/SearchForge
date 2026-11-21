
erDiagram
    USER {
        uuid id PK
        string name
        string email UK
        string passwordHash
        datetime createdAt
        datetime updatedAt
    }

    SESSION {
        uuid id PK
        uuid userId FK
        string tokenHash
        datetime expiresAt
    }

    ORGANIZATION {
        uuid id PK
        string name
        string slug UK
        datetime createdAt
    }

    ORGANIZATION_MEMBERSHIP {
        uuid id PK
        uuid userId FK
        uuid organizationId FK
        string role
    }

    TEAM {
        uuid id PK
        uuid organizationId FK
        string name
    }

    TEAM_MEMBERSHIP {
        uuid id PK
        uuid teamId FK
        uuid userId FK
        string role
    }

    DOCUMENT {
        uuid id PK
        uuid organizationId FK
        uuid teamId FK
        uuid ownerId FK
        string title
        string documentType
        string visibility
        datetime deletedAt
    }

    DOCUMENT_VERSION {
        uuid id PK
        uuid documentId FK
        int versionNumber
        string storageKey UK
        string checksum
        uuid uploadedById FK
        datetime createdAt
    }

    DOCUMENT_SHARE {
        uuid id PK
        uuid documentId FK
        uuid userId FK
        string accessLevel
    }

    DOCUMENT_CHUNK {
        uuid id PK
        uuid versionId FK
        int chunkIndex
        text content
        int pageNumber
        string sectionTitle
    }

    PROCESSING_JOB {
        uuid id PK
        uuid versionId FK
        string jobType
        string status
        int attempts
    }

    SEARCH_TERM {
        uuid id PK
        string term UK
    }

    CHUNK_TERM {
        uuid id PK
        uuid chunkId FK
        uuid termId FK
        int termFrequency
    }

    AUDIT_LOG {
        uuid id PK
        uuid actorId FK
        uuid organizationId FK
        string action
        string resourceType
        uuid resourceId
        datetime createdAt
    }

    USER ||--o{ SESSION : has
    USER ||--o{ ORGANIZATION_MEMBERSHIP : joins
    ORGANIZATION ||--o{ ORGANIZATION_MEMBERSHIP : contains

    ORGANIZATION ||--o{ TEAM : owns
    TEAM ||--o{ TEAM_MEMBERSHIP : contains
    USER ||--o{ TEAM_MEMBERSHIP : joins

    ORGANIZATION ||--o{ DOCUMENT : owns
    USER ||--o{ DOCUMENT : creates
    TEAM o|--o{ DOCUMENT : groups

    DOCUMENT ||--|{ DOCUMENT_VERSION : has
    USER ||--o{ DOCUMENT_VERSION : uploads
    DOCUMENT ||--o{ DOCUMENT_SHARE : shares
    USER ||--o{ DOCUMENT_SHARE : receives

    DOCUMENT_VERSION ||--o{ DOCUMENT_CHUNK : produces
    DOCUMENT_VERSION ||--o{ PROCESSING_JOB : processes

    DOCUMENT_CHUNK ||--o{ CHUNK_TERM : indexed_by
    SEARCH_TERM ||--o{ CHUNK_TERM : appears_in

    USER ||--o{ AUDIT_LOG : performs
    ORGANIZATION ||--o{ AUDIT_LOG : records



## 2. Document Authorization Design

### Organization isolation

Every document belongs to exactly one organization.

A user must have an active membership in that organization to access its documents.

### Visibility levels

- PRIVATE: Accessible to the owner and explicitly shared users.
- TEAM: Accessible to members of the associated team.
- ORGANIZATION: Accessible to members of the owning organization.

### Additional rules

1. Organization administrators do not automatically bypass PRIVATE document permissions.
2. TEAM documents must have an associated team.
3. A document's team must belong to its organization.
4. Document owners must be members of the owning organization.
5. Directly shared users must belong to the same organization.
6. Revoked membership or document sharing must immediately affect access.
7. Backend authorization is mandatory for document access, downloads, and search.
8. Authorization must be enforced before passages enter the RAG context.

### Database integrity

Use composite unique constraints and foreign keys where appropriate
to enforce organization consistency across memberships, teams,
documents, and document shares.

Backend authorization checks remain mandatory.



## 3. Document Lifecycle and Processing

### Document and DocumentVersion

- Document represents the logical document.
- DocumentVersion represents an immutable uploaded revision.
- Each version references its original file in object storage.
- Previous versions are retained when a new version is uploaded.
- Each version is processed independently.

### Upload workflow

1. Authenticate the user.
2. Validate organization membership and document permissions.
3. Validate file type, size and content.
4. Upload the original file to object storage.
5. Create Document and DocumentVersion records.
6. Enqueue a background processing job.
7. Extract text and metadata.
8. Generate DocumentChunk records.
9. Build the keyword search index.
10. Publish the version as searchable after successful indexing.

### Document statuses

- ACTIVE
- ARCHIVED
- DELETED

### Processing job statuses

- PENDING
- PROCESSING
- COMPLETED
- FAILED

### Version publication

- Preserve the previous published version while a new version is processing.
- Update the current published version only after successful processing and indexing.
- Failed processing must not destroy previously searchable content.

### Failure handling

- Use retry policies for transient processing failures.
- Keep the original uploaded file when extraction fails.
- Handle orphaned object storage files.
- Make processing idempotent.
- Record errors for debugging and auditing.
- Hide deleted documents from search immediately.



## 4. Document Version Publication Strategy

### Version tracking

- Every uploaded revision creates a new DocumentVersion.
- Version numbers are unique within a document.
- Document stores currentPublishedVersionId.
- The published version pointer is nullable until the first successful publication.
- DocumentVersion stores its processing status.
- ProcessingJob tracks individual processing attempts.

### Publication rules

1. New versions do not replace the currently published version during processing.
2. Search continues using the last successfully published version.
3. A version becomes eligible for publication only after extraction, chunking and indexing succeed.
4. Publication updates currentPublishedVersionId transactionally.
5. A slower older job must never downgrade the published version.
6. Previous versions and their chunks remain available for history.
7. If processing fails, the previous published version remains searchable.
8. A document with no successfully published version is not searchable.

### Search behavior

- Search only chunks belonging to currentPublishedVersionId.
- Apply document authorization before returning chunks or snippets.
- Exclude ARCHIVED and DELETED documents from normal search.
- A permission revocation must take effect without requiring index rebuilding.

### Retry behavior

- Processing jobs must be idempotent.
- Failed jobs may be retried.
- Duplicate chunks and index entries must be prevented using
  unique constraints and deterministic processing identifiers.



## 5. Database Integrity and Constraints

### Unique constraints

- User.email
- Organization.slug
- OrganizationMembership(userId, organizationId)
- TeamMembership(teamId, userId)
- DocumentVersion(documentId, versionNumber)
- DocumentShare(documentId, userId)
- DocumentChunk(versionId, chunkIndex)
- SearchTerm(normalizedTerm)
- ChunkTerm(chunkId, termId)

### Referential integrity

- Every membership references an existing user and organization.
- Every team belongs to an organization.
- Every team membership references an existing team and user.
- Every document belongs to an organization and has an owner.
- Every document team must belong to the same organization.
- Every document version belongs to one document.
- Every document chunk belongs to one document version.
- Every chunk-term mapping references an existing chunk and term.

### Published version integrity

- Document.currentPublishedVersionId is nullable.
- A published version must belong to the same document.
- Enforce this relationship using a composite foreign key
  referencing DocumentVersion(documentId, id).
- A document cannot publish a version before successful processing
  and indexing.

### Indexing strategy

Create indexes based on authorization, document listing,
version lookup, processing queues and keyword retrieval queries.

Avoid duplicate indexes already created by unique constraints.

### Deletion strategy

- Use soft deletion for documents.
- Preserve document version history.
- Exclude archived and deleted documents from normal search.
- Use controlled cleanup for permanent deletion and object storage removal.


## 6. Identity and Authentication Module

### User

| Field | Type | Nullable |
|---|---|---|
| id | UUID | No |
| name | VARCHAR(100) | No |
| email | VARCHAR(255) | No |
| passwordHash | TEXT | No |
| emailVerifiedAt | TIMESTAMP | Yes |
| isActive | BOOLEAN | No |
| createdAt | TIMESTAMP | No |
| updatedAt | TIMESTAMP | No |

Constraints:
- Primary key: id
- Unique: email
- Normalize email to lowercase before persistence.
- Store Argon2id password hashes, never plaintext passwords.

### Session

| Field | Type | Nullable |
|---|---|---|
| id | UUID | No |
| userId | UUID | No |
| refreshTokenHash | TEXT | No |
| expiresAt | TIMESTAMP | No |
| revokedAt | TIMESTAMP | Yes |
| lastUsedAt | TIMESTAMP | Yes |
| userAgent | TEXT | Yes |
| createdAt | TIMESTAMP | No |

Constraints:
- Primary key: id
- Foreign key: userId references User.id
- Unique: refreshTokenHash
- Index: (userId, revokedAt)
- Index: expiresAt

### Session rules

- A user can have multiple active sessions.
- Each session belongs to exactly one user.
- Refresh tokens are stored as cryptographic hashes.
- Logout revokes the relevant session.
- Expired and revoked sessions cannot be used.
- Session cleanup must not delete the associated user account.


## 7. Organization and Team Management

### Organization

Fields:
- id: UUID, primary key
- name: VARCHAR(150), required
- slug: VARCHAR(100), required and unique
- createdAt: TIMESTAMP, required
- updatedAt: TIMESTAMP, required

### OrganizationMembership

Fields:
- id: UUID, primary key
- userId: UUID, foreign key
- organizationId: UUID, foreign key
- role: ADMIN or MEMBER
- isActive: BOOLEAN, required
- joinedAt: TIMESTAMP, required

Constraints:
- Unique(userId, organizationId)
- Index(organizationId, isActive)
- Index(userId, isActive)

### Team

Fields:
- id: UUID, primary key
- organizationId: UUID, foreign key
- name: VARCHAR(100), required
- description: TEXT, nullable
- createdAt: TIMESTAMP, required
- updatedAt: TIMESTAMP, required

Constraints:
- Unique(organizationId, name)

### TeamMembership

Fields:
- id: UUID, primary key
- teamId: UUID, foreign key
- userId: UUID, foreign key
- role: LEAD or MEMBER
- joinedAt: TIMESTAMP, required

Constraints:
- Unique(teamId, userId)
- Index(userId, teamId)

### Authorization and integrity

- A user can belong to multiple organizations.
- A user can have different roles in different organizations.
- A team belongs to exactly one organization.
- A user can belong to multiple teams.
- Team membership requires active membership in the
  team's organization.
- Organization and team roles are independent.
- Organization ADMIN does not automatically bypass
  PRIVATE document permissions.

## 8. Document Management

### 8.1 Document

The `Document` entity represents a logical engineering document, independent of its uploaded file versions.

**Fields:**

| Field | PostgreSQL Type | Nullable |
|---|---|---|
| id | UUID | No |
| organizationId | UUID | No |
| teamId | UUID | Yes |
| ownerId | UUID | No |
| title | VARCHAR(255) | No |
| description | TEXT | Yes |
| documentType | ENUM | No |
| visibility | ENUM | No |
| status | ENUM | No |
| currentPublishedVersionId | UUID | Yes |
| deletedAt | TIMESTAMP | Yes |
| createdAt | TIMESTAMP | No |
| updatedAt | TIMESTAMP | No |

**Enums:**

- `DocumentType`: INCIDENT, TECHNICAL_SOLUTION, API_DOCUMENTATION, ARCHITECTURE_DECISION, DEPLOYMENT_GUIDE, OTHER
- `DocumentVisibility`: PRIVATE, TEAM, ORGANIZATION
- `DocumentStatus`: ACTIVE, ARCHIVED, DELETED

**Constraints and indexes:**

- Primary key: `id`
- Foreign key: `organizationId` references Organization.id
- Foreign key: `ownerId` references User.id
- Foreign key: `teamId` references Team.id
- Index: `(organizationId, status)`
- Index: `(ownerId, status)`
- Index: `(teamId, status)`
- `currentPublishedVersionId` must reference a version belonging to the same document.

### 8.2 DocumentVersion

The `DocumentVersion` entity represents an immutable uploaded revision of a logical document.

Every new upload creates a new version instead of replacing the previous file.

**Fields:**

| Field | PostgreSQL Type | Nullable |
|---|---|---|
| id | UUID | No |
| documentId | UUID | No |
| versionNumber | INTEGER | No |
| originalFileName | VARCHAR(255) | No |
| storageKey | TEXT | No |
| mimeType | VARCHAR(150) | No |
| fileSizeBytes | BIGINT | No |
| checksum | VARCHAR(64) | No |
| processingStatus | ENUM | No |
| uploadedById | UUID | No |
| createdAt | TIMESTAMP | No |
| publishedAt | TIMESTAMP | Yes |

**Processing status enum:**

- UPLOADED
- PENDING
- PROCESSING
- COMPLETED
- FAILED

**Constraints and indexes:**

- Primary key: `id`
- Foreign key: `documentId` references Document.id
- Foreign key: `uploadedById` references User.id
- Unique: `(documentId, versionNumber)`
- Unique: `storageKey`
- Index: `(documentId, createdAt)`
- Index: `(documentId, processingStatus)`

**Version rules:**

1. Each upload creates a new version.
2. Version numbers are sequential within a document.
3. Previously uploaded versions remain immutable.
4. Each version references its original file in object storage.
5. Each version is processed independently.
6. Failed versions remain available for troubleshooting and retry.
7. A new version does not replace the currently published version until processing and indexing succeed.

### 8.3 DocumentShare

The `DocumentShare` entity manages explicit user-level access to PRIVATE documents.

**Fields:**

| Field | PostgreSQL Type | Nullable |
|---|---|---|
| id | UUID | No |
| documentId | UUID | No |
| userId | UUID | No |
| sharedById | UUID | No |
| accessLevel | ENUM | No |
| createdAt | TIMESTAMP | No |

**Access level enum:**

- READ

The initial implementation supports read-only sharing. Additional permissions can be introduced if required later.

**Constraints and indexes:**

- Primary key: `id`
- Foreign key: `documentId` references Document.id
- Foreign key: `userId` references User.id
- Foreign key: `sharedById` references User.id
- Unique: `(documentId, userId)`
- Index: `(userId, documentId)`

**Sharing rules:**

1. Only authorized users can share a document.
2. The recipient must have an active membership in the document's organization.
3. Explicit sharing grants read access to a PRIVATE document.
4. Revoking a share immediately removes the recipient's access.
5. Document sharing does not bypass organization isolation.

### 8.4 Document Visibility Rules

| Visibility | Access |
|---|---|
| PRIVATE | Owner and explicitly shared users |
| TEAM | Active members of the associated team |
| ORGANIZATION | Active members of the owning organization |

Additional rules:

- Every document belongs to exactly one organization.
- TEAM documents must have an associated team.
- The associated team must belong to the same organization.
- The document owner must be an active member of its organization.
- Explicitly shared users must belong to the same organization.
- Organization administrators do not automatically bypass PRIVATE document permissions.
- Archived and deleted documents are excluded from normal search.

### 8.5 Document Storage Design

Original files are stored in external object storage, not directly inside PostgreSQL.

PostgreSQL stores the metadata and storage reference.

**Storage responsibilities:**

- Store original uploaded files.
- Generate unique storage keys.
- Prevent filename collisions.
- Keep different document versions as separate objects.
- Retrieve files only through backend-authorized operations.
- Handle failed uploads and orphaned storage objects.
- Support controlled deletion and retention policies.

Example storage key:

`organizations/{organizationId}/documents/{documentId}/versions/{versionId}/original.pdf`

The original filename is stored separately as metadata.

### 8.6 Document Version Publication

The currently published version is identified through:

`Document.currentPublishedVersionId`

**Publication rules:**

1. A document can initially have a NULL published-version pointer.
2. A newly uploaded version begins processing without replacing the existing published version.
3. Search continues using the last successfully published version.
4. A version becomes eligible for publication only after extraction, chunking, and indexing succeed.
5. Publication updates `currentPublishedVersionId` transactionally.
6. A slower older processing job must never downgrade the published version.
7. Previous versions and their chunks are retained for version history.
8. If processing fails, the previous published version remains searchable.

No separate `isPublished` Boolean is required because the document's published-version pointer is authoritative.

### 8.7 Referential Integrity

- Every version must belong to an existing document.
- Every document share must reference an existing document and user.
- A document's owner and explicitly shared users must satisfy organization membership rules.
- A document's team must belong to its organization.
- `currentPublishedVersionId` must reference a `DocumentVersion` belonging to the same document.
- Enforce the published-version relationship through a composite foreign key referencing `DocumentVersion(documentId, id)`.
- Enforce organization consistency using appropriate composite foreign keys and backend transactions.

### 8.8 Design Decisions

- Separate logical documents from uploaded versions.
- Preserve historical versions instead of overwriting files.
- Store original files in object storage.
- Store file metadata and processing state in PostgreSQL.
- Use a published-version pointer to support uninterrupted search during new version processing.
- Keep authorization independent of search indexing.
- Enforce permissions in the backend before document content or passages are returned.
- Use soft deletion and controlled permanent cleanup.
