# SearchForge Complete System Overview

## Objective

Understand the problem SearchForge solves, its two primary workflows, its initial architecture, and the responsibilities of its major components.

## 1. Product Overview

SearchForge Enterprise is a permission-aware internal technical document search platform for engineering teams.

It helps employees find relevant information inside their organization's uploaded technical documents.

Initial document types:

- API documentation
- Engineering documentation
- Troubleshooting guides
- Incident reports
- Architecture Decision Records (ADRs)

Initial search approach: Keyword-based search.

## 2. Problem Statement

Organizations accumulate large numbers of internal documents. Employees may know that a solution or technical explanation exists but struggle to locate the correct document.

For example:

"How did our engineering team resolve the PostgreSQL connection timeout issue?"

Searching filenames or opening documents individually is inefficient.

SearchForge aims to make this information retrievable through a centralized, permission-aware search system.

## 3. Two Primary Workflows

### A. Document Indexing

Indexing prepares documents for efficient retrieval.

1. Employee uploads a document.
2. Backend authenticates and authorizes the request.
3. File type and size are validated.
4. Original file is stored.
5. Document metadata is stored in PostgreSQL.
6. Document processing job is added to BullMQ.
7. Worker extracts text.
8. Extracted text is normalized.
9. Text is tokenized into terms.
10. Search engine builds an inverted index.
11. Index entries are stored in PostgreSQL.
12. Document status changes to INDEXED.

### B. Document Searching

Searching retrieves relevant documents using the prepared index.

1. Employee submits a search query.
2. Backend authenticates the employee.
3. Organization membership and document permissions are checked.
4. Query is normalized and tokenized.
5. Inverted index is used to find matching documents.
6. Candidate documents are retrieved within the authorized scope.
7. TF-IDF is used to calculate relevance scores.
8. Documents are ranked.
9. Snippets are generated from authorized content.
10. Results are returned to the frontend.

## 4. Initial Architecture

React Frontend  
↓  
Node.js + Express API  
↓  
PostgreSQL + Redis/BullMQ  
↓  
Background Workers  
↓  
Document Extraction and Search Indexing  
↓  
Inverted Index and TF-IDF Search Engine

This is an initial conceptual architecture. Detailed component boundaries and deployment decisions will be refined during research.

## 5. Technology Responsibilities

| Technology | Responsibility |
|---|---|
| React | Frontend user interface |
| Node.js | JavaScript runtime |
| Express | Backend HTTP framework |
| PostgreSQL | Persistent relational database |
| Prisma | Database ORM |
| Redis | Background queue infrastructure |
| BullMQ | Background job management |
| Workers | Asynchronous document processing |

## 6. Important Engineering Concepts Introduced

- Client-server architecture
- REST APIs
- Authentication
- Authorization
- Multi-tenancy
- File storage
- Asynchronous processing
- Background workers
- Job queues
- Text extraction
- Text normalization
- Tokenization
- Inverted index
- TF-IDF ranking
- Document lifecycle management

## 7. Initial Design Decisions

1. Use manual document uploads for the initial version.
2. Start with keyword-based search.
3. Build our own inverted index and ranking implementation.
4. Use PostgreSQL for persistent application data and initial index storage.
5. Use Redis and BullMQ for background processing.
6. Keep document processing separate from HTTP request handling.
7. Enforce authorization on the backend before returning document content or snippets.
8. Make document processing retryable and safe against duplicate job execution.

## 8. Important Questions for Further Research

- Why is indexing necessary?
- How does an inverted index work internally?
- How does a background job queue work?
- Why should document processing be asynchronous?
- How does TF-IDF rank documents?
- How can multiple organizations safely share the same application?
- How do we prevent unauthorized document retrieval?
- What happens when a worker crashes during indexing?

## 9. Day 01 Learning Outcome

I understand that SearchForge has two major workflows:

1. Indexing: preparing documents for search.
2. Searching: retrieving and ranking relevant documents.

I have also identified the initial responsibilities of the frontend, API, database, queue, workers, and search engine.

**Next topic:** Deep dive into client-server architecture and the complete document upload lifecycle.

---

*Project: SearchForge Enterprise*  
*Learning approach: Research → Understand → Document → Implement → Test → Commit*