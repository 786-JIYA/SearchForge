# SearchForge: Feature List per Model

*A plain-language guide to all 59 models in `schema.prisma`: what each one is for, which features it enables, and the rules you must remember. Read it with the schema open.*

---

## How to read this guide

- **Purpose** is one sentence on why the model exists.
- **Features** are things a user or the system can do because it exists.
- **Remember** is the rule most likely to cause a bug if you forget it.
- A model describes **data only**. Logic such as login checks, ranking and retries lives in your Express services and BullMQ workers.

**Vocabulary:** *soft delete* = mark as deleted but keep for recovery. *Purge* = permanent deletion. *Tombstone* = anonymized leftover row. *Generation* = one complete build of an index.

---

## 1. Identity and authentication (5 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **User** | One account | Register, log in, profile (name, job title, bio), platform role, suspend, request deletion, recover within 30 days | Status flow: `ACTIVE` → `DELETION_PENDING` → `PURGING` → `DELETED`. Row is kept as a tombstone; email becomes `deleted+<id>@deleted.invalid`; `passwordHash` becomes null. Email is always lowercase |
| **Session** | One refresh-token generation | Stay logged in, many devices, log out one device or all, detect stolen tokens | Store only the token's SHA-256 hash. Rotation copies the old `expiresAt` (30-day cap). `replacedById` links to the next generation. Reusing a replaced token revokes the whole `familyId` |
| **EmailVerificationToken** | Proof the user owns their email | Verify email after registration | 24 hours, single use, hashed |
| **PasswordResetToken** | Safe password recovery | Reset password by email link | 1 hour, single use, hashed. Revoke on password change |
| **EmailChangeRequest** | Verified email change | Change email only after confirming the new address | One live request per user and per target email. Re-check availability when confirming. Keep current session, revoke the others |

---

## 2. Workspace and access (4 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **Workspace** | A team's knowledge space | Create, edit, soft delete, restore within 30 days | `status`: `ACTIVE`, `SOFT_DELETED`, `PURGING`. `createdById` is the original creator, **not** the current owner. A deleted workspace is unreachable through API, search and invitations |
| **WorkspaceMember** | Links a user to a workspace with a role | Roles `OWNER`, `ADMIN`, `MEMBER`; join, leave, be removed; ownership transfer; membership history | History rows are kept; rejoining makes a new row. Database allows at most one active membership and one active OWNER. "At least one OWNER" is **your** transaction logic |
| **WorkspaceInvitation** | Invite people by email | Send, resend, revoke, accept | 7 days. Token hashed. Only the verified-email owner accepts. One pending invite per email per workspace |
| **WorkspaceAuditLog** | Record of important workspace actions | OWNER/ADMIN view and export (CSV/JSON) | Append-only. Never put tokens, passwords or document content in `metadata`. Deleted with the workspace at purge |

---

## 3. Document management (5 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **Document** | Permanent identity of a logical document | Title, description, soft delete, 30-day recovery | Holds `activeGenerationId`: the pointer that says which index is searchable. No owner or workspace data here |
| **DocumentVersion** | One immutable uploaded revision | Version history, exact-version citations, view old versions | Never edited. Unique `(documentId, versionNumber)`. Stores the storage key and SHA-256, never file bytes. Only the latest completed version is searchable |
| **DocumentVersionText** | Extracted text of a version | Keeps text for every version | Separate table so normal queries never load big text |
| **WorkspaceDocument** | "This workspace sees this document" | Share across workspaces, remove and restore (30 days), one category, tags | Removing it from one workspace must not touch other workspaces. Restore clears `deletedAt`. Cannot be restored once the document is purging |
| **DocumentUploadAttempt** | Staging area for an upload | Duplicate rejection, scan and extraction tracking, version replacement, safe rejection messages | A `Document` and `DocumentVersion` are created only after extraction succeeds. Duplicates are checked inside one workspace only. `baseVersionNumber` catches two people replacing at once |

---

## 4. Document organization (4 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **WorkspaceCategory** | Workspace-specific category | OWNER/ADMIN create, rename, delete; filter by category | Names unique per workspace after normalizing. Deleting removes the association, not the documents |
| **WorkspaceTag** | Workspace-specific tag | Members create and apply; OWNER/ADMIN rename and delete | Same uniqueness rule |
| **WorkspaceDocumentTag** | Connects a tag to a workspace-document | Many tags per document, filter by tag | A tag must belong to the same workspace (trigger enforces it) |
| **DocumentRelationship** | Typed link between two documents | `RESOLVES`, `REFERENCES`, `DOCUMENTS`, `RELATED_TO`; "how was this incident fixed?" navigation | Pins the exact versions at creation. No self-links. `RELATED_TO` stored as an ordered pair. Both documents must be in the workspace. When showing a related document, check access to **both** documents |

---

## 5. Processing and indexing (9 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **MalwareScanJob** | Scan work for one target | Quarantine bad or unscanned files | Targets **exactly one** of an upload attempt or an accepted version (database check). Scanner down means the file stays quarantined |
| **MalwareScanAttempt** | One scan try | Failure investigation | 3 automatic attempts, then admins can rescan |
| **DocumentProcessingJob** | Text extraction work | Async extraction | Points to the **upload attempt**, because no version exists yet |
| **DocumentProcessingAttempt** | One extraction try | Retry history, safe errors | 3 attempts total, then reject. User re-uploads a fixed file |
| **IndexingJob** | Build one index generation | Async indexing, manual retry by workspace OWNER/ADMIN | Its generation must belong to the version being indexed |
| **IndexingAttempt** | One indexing try | Retry history | 3 automatic attempts, then `FAILED` |
| **DocumentPassage** | A piece of a version's text | Snippets, page and section citations, RAG evidence | The unit of **everything**: ranking, citing, prompting. Stores `tokenCount` for TF and `pageNumber` and `sectionTitle` for traceability |
| **DocumentIndexGeneration** | One complete build of a version's index | Rebuild without downtime, safe switch | Status is `BUILDING`, `READY`, `RETIRED`, `FAILED`; **never** "active". The active one is whatever `Document.activeGenerationId` points to |
| **InvertedIndexEntry** | One term in one passage | Fast keyword lookup, TF-IDF scoring | Biggest table. Composite key `(generationId, term, passageId)`. Stores positions for future phrase search. Build with one `INSERT ... SELECT` |

---

## 6. Search and personalization (5 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **SearchHistory** | A user's recent searches | See and delete own searches | Private. 90 days. Visible only to the owner |
| **SavedSearch** | Reusable query plus filters | Save, rename, reuse | Personal. Auto-named from the query. Kept until the user deletes it. Must handle purged results gracefully |
| **Bookmark** | Personal document bookmark | Bookmark a document, open the latest completed version | Points to the logical document. Deleted when the document is purged |
| **SearchFeedback** | Helpful / not helpful on a result | One changeable vote per user per exact version | OWNER/ADMIN see counts only, not voters |
| **FeedbackAggregate** | Anonymous long-term statistics | Measure ranking and answer quality over time | No user, document or workspace column. Individual rows are folded in just before they are deleted |

---

## 7. Answers: RAG and agent (6 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **GeneratedAnswer** | One question and its answer | Grounded answers, "not enough evidence", graceful degradation | Private to the user. 90 days. `status` covers `NO_RESULTS` (LLM never called) and `LLM_UNAVAILABLE` (show sources only). Stores `promptVersion` and `rankingVersion` for evaluation |
| **AnswerSource** | A passage used as evidence | Clickable citations, retrieval-versus-generation evaluation | Stores passage IDs, not text. `label` (`S1`, `S2`) is what the LLM cites; a label not in this table is a fake citation. Re-check access when an old answer is opened |
| **AnswerFeedback** | Vote on an answer | Helpful / not helpful, reason (wrong, unsupported, bad citation, outdated) | One vote per user per answer; only the owner can vote |
| **LlmCall** | One call to the model provider | Cost, latency and failure tracking | Never store API keys or raw provider payloads |
| **AgentRun** | One agent investigation | Multi-step research, hard limits | `maxIterations` and token and time budgets. Stop reason is recorded |
| **AgentStep** | One tool call inside a run | "How I found this" trace, debugging | Stores validated arguments and **passage IDs only**. User identity never comes from the model |

---

## 8. Notifications and email preparation (3 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **Notification** | In-app message | Upload, new-version, removal and restore alerts | Sent to all active members except the actor. 90 days, or earlier when the resource is purged. No foreign key to the resource, so purge finds them by `(resourceType, resourceId)` |
| **EmailNotification** | Prepared workspace-deletion email | Gmail Compose with BCC, mark as sent, cancel | Opening Compose does not prove it was sent (`SENT_REPORTED` is the owner's report). Re-check recipient eligibility when preparing |
| **EmailNotificationRecipient** | Recipient snapshot | Records who was in BCC | Email address set to null 90 days after workspace purge |

---

## 9. Purge workflows (15 models)

All purge jobs are **resumable and repeatable**. A worker re-checks state before every destructive step, and a file that is already gone counts as success.

| Model | Purpose | Remember |
|---|---|---|
| **DocumentPurgeJob** | Permanent deletion of one document | No foreign key to the document. `pointOfNoReturnAt` set means restore is no longer possible |
| **DocumentPurgeStage** / **DocumentPurgeTask** | Steps and sub-steps | Stages: revalidate, remove index, delete dependents, delete files, delete rows, write summary. Task types are enums; each task is unique per stage |
| **DocumentPurgeAttempt** | One run of the job | Bounded retries, then `FAILED` and an admin alert |
| **DocumentPurgeSummary** | What remains afterwards | Counts and status only; no title, no content |
| **WorkspacePurgeJob** | Permanent deletion of a workspace | Has `pointOfNoReturnAt`. Never deletes a document still linked elsewhere |
| **WorkspacePurgeStage** / **WorkspacePurgeTask** | Steps and sub-steps | Includes detaching documents and scheduling purges of orphaned ones |
| **WorkspacePurgeAttempt** | One run | Admin resume gets a fresh retry budget |
| **WorkspacePurgeSummary** | What remains afterwards | No workspace name |
| **UserPurgeJob** | Finalizing account deletion | The user row stays (tombstone), so a normal Restrict foreign key is used |
| **UserPurgeAttempt** | One run | Starts only after the atomic move to `PURGING` |
| **PurgeIntervention** | Admin manual resume | Records admin, time and mandatory reason; kept indefinitely |

---

## 10. Reliability (1 model)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **OutboxEvent** | Safe hand-off from PostgreSQL to BullMQ | No lost jobs if the API crashes between the two | Insert it in the **same transaction** as the business change. A dispatcher publishes unpublished events (`FOR UPDATE SKIP LOCKED`) and marks them published. Delivery is at-least-once, so workers must be idempotent |

---

## 11. Platform administration (4 models)

| Model | Purpose | Features | Remember |
|---|---|---|---|
| **PlatformAuditLog** | Record of admin actions | Review admin activity, audited exports | Append-only; the API database role cannot delete from it |
| **PlatformSecurityEvent** | Security signals | Failed logins, authorization denials, token reuse | Includes IP address, so it has a retention period |
| **RetentionPolicy** | How long each data type is kept | Configurable cleanup | `null` days means keep forever. One cleanup worker reads it |
| **PlatformSetting** | Tunable values | Page sizes, upload limit, attempt limits, agent limit | Validate every value in code before saving, and audit every change |

---

## 12. How the models work together

**Upload:** `DocumentUploadAttempt` → `MalwareScanJob` → `DocumentProcessingJob` → (transaction) `Document` + `DocumentVersion` + `DocumentVersionText` + `DocumentPassage` + `WorkspaceDocument` → `IndexingJob` → `DocumentIndexGeneration` + `InvertedIndexEntry` → switch `Document.activeGenerationId`.

**Search:** `WorkspaceMember` → `Workspace` → `WorkspaceDocument` → `Document.activeGenerationId` → `InvertedIndexEntry` → `DocumentPassage`. Permissions are part of the query, never applied afterwards.

**Answer:** search → `GeneratedAnswer` + `AnswerSource` (+ `LlmCall`, optionally `AgentRun` and `AgentStep`) → `AnswerFeedback`.

**Delete a document:** last `WorkspaceDocument` removed → 30-day recovery → `DocumentPurgeJob` stages → `DocumentPurgeSummary`.

**Delete an account:** `DELETION_PENDING` → 30 days → `PURGING` → `UserPurgeJob` → tombstone.

---

## 13. Quick map: feature to models

| Feature | Models |
|---|---|
| Register, log in, stay logged in | User, Session |
| Verify email, reset password, change email | EmailVerificationToken, PasswordResetToken, EmailChangeRequest |
| Create and manage workspaces | Workspace, WorkspaceMember, WorkspaceInvitation, WorkspaceAuditLog |
| Upload documents safely | DocumentUploadAttempt, MalwareScanJob, DocumentProcessingJob and their attempts |
| Version history and sharing | Document, DocumentVersion, DocumentVersionText, WorkspaceDocument |
| Organize and connect documents | WorkspaceCategory, WorkspaceTag, WorkspaceDocumentTag, DocumentRelationship |
| Keyword search with ranking | DocumentPassage, DocumentIndexGeneration, InvertedIndexEntry, IndexingJob and attempts |
| Personal search features | SearchHistory, SavedSearch, Bookmark, SearchFeedback, FeedbackAggregate |
| AI answers with citations | GeneratedAnswer, AnswerSource, AnswerFeedback, LlmCall |
| Agent investigation | AgentRun, AgentStep |
| Notifications and deletion emails | Notification, EmailNotification, EmailNotificationRecipient |
| Safe permanent deletion | Document, Workspace and User purge models, PurgeIntervention |
| Reliable background jobs | OutboxEvent |
| Administration and compliance | PlatformAuditLog, PlatformSecurityEvent, RetentionPolicy, PlatformSetting |

---

