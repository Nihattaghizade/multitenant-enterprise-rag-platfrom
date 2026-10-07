# Enterprise RAG Platform — Architecture Guide

> **Status:** v2 baseline
>
> **Scope:** Text-only, multi-tenant enterprise RAG application with secure isolation, MinIO object storage, hybrid retrieval, reranking, citations, evaluation, and observability.

## 1. Purpose

This document is the developer-facing architecture reference for the repository. It explains the folder layout, ownership of each layer, important functions, and the upload/query flows.

The formal Word specification remains the source for product-level decisions. This file is a living technical guide and must be updated in the same pull request as any architecture or file-responsibility change.

## 2. Core Decisions

| Area | Decision |
|---|---|
| Backend | FastAPI, async-first Python |
| Frontend | Next.js App Router, TypeScript, Tailwind CSS, shadcn/ui |
| AI orchestration | LangChain |
| AI tracing/evaluation | LangSmith |
| Relational database | PostgreSQL |
| Vector database | Qdrant |
| Cache, rate limits, queue support | Redis |
| Object storage | MinIO, self-hosted and S3-compatible |
| Embeddings | `BAAI/bge-m3`, 1024 dimensions |
| Reranker | `BAAI/bge-reranker-v2-m3` |
| Retrieval | Hybrid dense + sparse (BM25-style), fused with RRF |
| Deployment | Docker and Docker Compose first |
| Input scope | Text extracted from TXT, MD, DOCX, PPTX, PDF, HTML; no OCR or visual understanding in v1 |

## 3. Architecture Overview

```text
Browser / Next.js
       |
       v
FastAPI API  <--------------------->  Redis
       |                                  |-- query cache
       |                                  |-- embedding cache
       |                                  |-- rate limits / queue support
       |
       +----------------------------> PostgreSQL
       |                                  |-- users, tenants, memberships
       |                                  |-- documents, parent contexts
       |                                  |-- audit logs, query logs
       |
       +----------------------------> MinIO
       |                                  |-- original uploaded files
       |                                  |-- optional large parent contexts
       |
       +----------------------------> Qdrant
       |                                  |-- child retrieval chunks
       |                                  |-- dense + sparse vectors
       |
       +----------------------------> Embedding / Reranker / LLM services
       |
       +----------------------------> LangSmith
                                          |-- ingestion traces
                                          |-- retrieval/reranking traces
                                          |-- answer-generation traces/evaluations
```

### Non-negotiable isolation rule

Every database operation and Qdrant search is scoped by `tenant_id`. The backend derives tenant context from the authenticated membership. Client-provided tenant identifiers never grant access on their own.

## 4. Repository Tree

```text
rag-platform/
├── docker-compose.yml
├── .env.example
├── README.md
├── ARCHITECTURE.md
├── docs/
│   └── runbooks/
│       ├── backup-restore.md
│       └── reindexing.md
│
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   │
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── deps.py
│   │   │   ├── auth.py
│   │   │   ├── documents.py
│   │   │   ├── chat.py
│   │   │   ├── tenants.py
│   │   │   └── health.py
│   │   │
│   │   ├── core/
│   │   │   ├── __init__.py
│   │   │   ├── security.py
│   │   │   ├── auth.py
│   │   │   ├── permissions.py
│   │   │   ├── rate_limit.py
│   │   │   └── errors.py
│   │   │
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   ├── session.py
│   │   │   ├── models.py
│   │   │   ├── repositories/
│   │   │   │   ├── documents.py
│   │   │   │   ├── tenants.py
│   │   │   │   ├── users.py
│   │   │   │   └── audit.py
│   │   │   └── migrations/
│   │   │
│   │   ├── storage/
│   │   │   ├── __init__.py
│   │   │   ├── s3_adapter.py
│   │   │   └── minio_client.py
│   │   │
│   │   ├── ingestion/
│   │   │   ├── __init__.py
│   │   │   ├── types.py
│   │   │   ├── parser.py
│   │   │   ├── normalize.py
│   │   │   ├── sections.py
│   │   │   ├── chunking.py
│   │   │   ├── embed.py
│   │   │   ├── index.py
│   │   │   └── worker.py
│   │   │
│   │   ├── retrieval/
│   │   │   ├── __init__.py
│   │   │   ├── types.py
│   │   │   ├── retriever.py
│   │   │   ├── qdrant_search.py
│   │   │   ├── rerank.py
│   │   │   └── context_assembly.py
│   │   │
│   │   ├── generation/
│   │   │   ├── __init__.py
│   │   │   ├── prompts.py
│   │   │   ├── llm.py
│   │   │   └── grounding.py
│   │   │
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── document_service.py
│   │   │   ├── chat_service.py
│   │   │   ├── tenant_service.py
│   │   │   ├── audit_service.py
│   │   │   └── usage_service.py
│   │   │
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── documents.py
│   │   │   ├── chat.py
│   │   │   └── tenants.py
│   │   │
│   │   ├── observability/
│   │   │   ├── __init__.py
│   │   │   ├── logging.py
│   │   │   ├── metrics.py
│   │   │   └── tracing.py
│   │   │
│   │   └── tasks/
│   │       ├── __init__.py
│   │       └── queue.py
│   │
│   └── tests/
│       ├── unit/
│       ├── integration/
│       └── e2e/
│
└── frontend/
    ├── package.json
    ├── next.config.ts
    ├── tsconfig.json
    ├── tailwind.config.ts
    ├── components.json
    ├── Dockerfile
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx
    │   ├── globals.css
    │   ├── login/page.tsx
    │   ├── chat/page.tsx
    │   ├── documents/page.tsx
    │   ├── documents/[id]/page.tsx
    │   └── admin/tenants/page.tsx
    ├── components/
    │   ├── ui/
    │   ├── chat/
    │   │   ├── chat-input.tsx
    │   │   ├── chat-messages.tsx
    │   │   └── citation-card.tsx
    │   ├── documents/
    │   │   ├── document-list.tsx
    │   │   ├── document-upload.tsx
    │   │   └── document-status.tsx
    │   └── layout/
    │       ├── navbar.tsx
    │       └── sidebar.tsx
    ├── lib/
    │   ├── api.ts
    │   ├── auth.ts
    │   ├── sse.ts
    │   └── utils.ts
    └── types/
        ├── api.ts
        ├── document.ts
        └── chat.ts
```

## 5. Backend Responsibilities

### 5.1 Application entry and configuration

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `backend/app/main.py` | Build the FastAPI app, register API routers, configure CORS/middleware, lifespan startup/shutdown, and global exception handling. | `create_app()`, `lifespan(app)`, `register_routers(app)` |
| `backend/app/config.py` | Read environment variables and define validated application settings. Keep all infrastructure URLs, model settings, security settings, and chunking settings here. | `Settings`, `get_settings()` |

`config.py` must contain these approved chunking defaults:

```python
RECURSIVE_PARENT_TARGET_TOKENS = 1200
RECURSIVE_PARENT_MIN_TOKENS = 1000
RECURSIVE_PARENT_MAX_TOKENS = 1400
RECURSIVE_CHILD_TARGET_TOKENS = 350
RECURSIVE_CHILD_MIN_TOKENS = 300
RECURSIVE_CHILD_MAX_TOKENS = 400
RECURSIVE_CHILD_OVERLAP_TOKENS = 50
SEMANTIC_SECTION_TOKEN_THRESHOLD = 2500
MAX_PARENT_CONTEXTS_PER_ANSWER = 3
HYBRID_CANDIDATE_LIMIT = 60
RERANKED_CHILD_LIMIT = 8
```

### 5.2 API layer

API files are thin controllers. They validate request schemas, authenticate/authorize, call a service, and serialize a response. They must not contain ingestion/retrieval business logic.

| File | Route ownership | Responsibilities |
|---|---|---|
| `api/deps.py` | Dependencies | Inject DB/Redis/Qdrant/MinIO clients; resolve current user, tenant membership, and request context. |
| `api/auth.py` | `/api/v1/auth/*` | Login, refresh, logout. Calls `core/auth.py`. |
| `api/documents.py` | `/api/v1/documents/*` | Upload, list, detail, delete, reindex, status. Calls `document_service.py`. |
| `api/chat.py` | `/api/v1/chat/stream` | Accepts a query and streams answer/citation SSE events. Calls `chat_service.py`. |
| `api/tenants.py` | `/api/v1/tenants/*` | Tenant and membership administration. Requires admin role. |
| `api/health.py` | `/health`, `/ready`, `/metrics` | Liveness/readiness and protected metrics endpoint. |

### 5.3 Core security and permissions

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `core/security.py` | Password hashing, password validation, generic token primitives. | `hash_password()`, `verify_password()` |
| `core/auth.py` | Authenticate users; issue, refresh, decode, and revoke JWTs. | `authenticate_user()`, `create_access_token()`, `create_refresh_token()`, `decode_token()` |
| `core/permissions.py` | Role checks and tenant-scoped authorization helpers. | `require_role()`, `assert_tenant_access()` |
| `core/rate_limit.py` | Redis-backed per-user and per-tenant rate limiting. | `check_rate_limit()` |
| `core/errors.py` | Domain exceptions mapped to the standard API error response. | `NotFoundError`, `ForbiddenError`, `ValidationError` |

### 5.4 Database layer

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `db/session.py` | Configure SQLAlchemy async engine and sessions. | `create_engine()`, `get_db_session()` |
| `db/models.py` | Define ORM entities and indexes. | `User`, `Tenant`, `Membership`, `Document`, `ParentContext`, `AuditLog`, `QueryLog` |
| `db/repositories/*.py` | Tenant-safe persistence queries. Repositories only read/write storage; services own workflow decisions. | `DocumentRepository`, `TenantRepository`, `AuditRepository` |
| `db/migrations/` | Alembic schema migrations. | One migration per reviewed schema change |

Required relational data concepts:

- `User`: identity and password/SSO fields.
- `Tenant`: organization/workspace.
- `Membership`: user-to-tenant role (`admin`, `editor`, `viewer`).
- `Document`: source metadata, MinIO object key, lifecycle/status, version.
- `ParentContext`: larger context text or object key, document linkage, section metadata.
- `AuditLog`: security-sensitive activity.
- `QueryLog`: query metadata, retrieval/cost/feedback fields according to privacy policy.

### 5.5 Object storage layer

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `storage/minio_client.py` | Configure the S3-compatible client using endpoint, credentials, TLS setting, and bucket. Ensure the bucket exists during controlled startup/provisioning. | `get_s3_client()`, `ensure_bucket()` |
| `storage/s3_adapter.py` | Provide application-owned storage methods. No business service should call the MinIO SDK directly. | `upload_original()`, `download_object()`, `delete_object()`, `create_presigned_download_url()` |

MinIO bucket: `rag-documents`.

Original file key format:

```text
tenants/{tenant_id}/documents/{document_id}/versions/{version}/original/{filename}
```

Optional parent-context object key format:

```text
tenants/{tenant_id}/documents/{document_id}/versions/{version}/parents/{parent_id}.txt
```

Buckets are private. Browser access must be through backend authorization and, only where needed, short-lived presigned URLs.

## 6. Ingestion Architecture

### 6.1 Ingestion files

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `ingestion/types.py` | Typed transfer objects between ingestion stages. | `ParsedDocument`, `Section`, `ParentContextData`, `RetrievalChunkData` |
| `ingestion/parser.py` | Extract text from supported formats: TXT, MD, DOCX, PPTX, PDF, HTML. No OCR. | `parse_file()`, `parse_docx()`, `parse_pptx()`, `parse_pdf()`, `parse_html()` |
| `ingestion/normalize.py` | Normalize Unicode, whitespace, headings, and line endings; reject insufficient text. | `normalize_text()`, `validate_minimum_text()` |
| `ingestion/sections.py` | Identify structural sections using headings and source locations. | `split_into_sections()` |
| `ingestion/chunking.py` | Build parent contexts and child retrieval chunks using the approved recursive/semantic policy. | `build_chunk_plan()`, `recursive_chunk_section()`, `semantic_chunk_section()` |
| `ingestion/embed.py` | Batch embeddings with Redis content-hash cache. | `embed_texts()`, `get_or_create_embedding()` |
| `ingestion/index.py` | Upsert vectors/payloads into Qdrant and delete document points during lifecycle actions. | `upsert_retrieval_chunks()`, `delete_document_points()` |
| `ingestion/worker.py` | Execute ingestion jobs outside request handlers. Update state, logs, trace, and failures. | `process_ingest_job()`, `process_reindex_job()` |

### 6.2 Approved Chunking Policy

The system has **one parent-context model** and **one child retrieval-chunk model**. Recursive and semantic describe how retrieval boundaries are created, not different storage schemas.

| Concept | Recursive strategy | Semantic strategy |
|---|---|---|
| When used | Default for all extractable sections | Only considered when a section exceeds 2,500 tokens |
| Boundary selection | Rule-based structure: headings, paragraphs, lines, sentences, words | Topic shifts detected from embedding similarity between adjacent sentence windows |
| Parent context | Target 1,200 tokens; soft range 1,000–1,400 | Uses the same parent-context target/budget model |
| Retrieval child | Target 350 tokens; soft range 300–400; overlap 50 | Shaped toward target 350 tokens and overlap 50 after semantic boundary detection |
| Retrieval behavior | Primary for normal sections and fallback for long sections | Primary retrieval corpus for qualifying long sections |

Rules:

1. Every section gets recursive parent contexts and child retrieval chunks.
2. If the section has **2,500 tokens or fewer**, index recursive children only.
3. If the section has **more than 2,500 tokens**, create semantic retrieval chunks. Those semantic chunks are primary for that section.
4. Retain recursive chunks from qualifying long sections as controlled fallback, evaluation, and rollback data.
5. Store a `strategy` payload value (`recursive` or `semantic`) on every Qdrant child retrieval chunk.
6. At answer time, retrieve and rerank children, resolve their `parent_id` values, deduplicate contexts, and pass at most three parent contexts to the LLM within its context budget.

### 6.3 Ingestion worker flow

```text
Upload accepted
  -> original file saved to MinIO
  -> Document(status=processing) saved in PostgreSQL
  -> ingestion job queued
  -> worker downloads object from MinIO
  -> parse -> normalize -> create sections
  -> recursive chunking (+ semantic refinement only for >2,500-token sections)
  -> parent contexts persisted in PostgreSQL / optional MinIO text objects
  -> child retrieval chunks embedded in batches
  -> dense + sparse vectors upserted to Qdrant
  -> Document(status=ready) or Document(status=failed)
  -> audit event, metrics, LangSmith trace
```

The main worker function should look conceptually like this:

```python
async def process_ingest_job(document_id: UUID) -> None:
    document = await document_repository.get_for_processing(document_id)
    raw_file = await storage.download_object(document.storage_key)
    parsed = parse_file(raw_file, document.mime_type)
    normalized = normalize_text(parsed.text)
    sections = split_into_sections(normalized)
    plan = build_chunk_plan(sections)
    parent_ids = await document_repository.create_parent_contexts(document, plan.parents)
    vectors = await embed_texts([chunk.text for chunk in plan.children])
    await upsert_retrieval_chunks(document, plan.children, parent_ids, vectors)
    await document_repository.mark_ready(document_id)
```

## 7. Retrieval Architecture

### 7.1 Retrieval files

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `retrieval/types.py` | Retrieval request, candidate, reranked result, and assembled-context types. | `RetrievalRequest`, `CandidateChunk`, `RerankedChunk`, `AssembledContext` |
| `retrieval/qdrant_search.py` | Execute dense+sparse filtered Qdrant searches and RRF fusion. | `hybrid_search()` |
| `retrieval/retriever.py` | Custom LangChain `BaseRetriever`; applies tenant filters and strategy/fallback policy. | `TenantHybridRetriever`, `_aget_relevant_documents()` |
| `retrieval/rerank.py` | Batch call BGE reranker, score candidates, retain top 8. | `rerank_candidates()` |
| `retrieval/context_assembly.py` | Fetch parent contexts, deduplicate, tokenize, and fit the LLM context budget. | `assemble_parent_contexts()` |

### 7.2 Qdrant collection contract

Collection name: `chunks`.

Each point represents a **child retrieval chunk**, not a full document or parent context.

Required payload:

```json
{
  "tenant_id": "uuid",
  "doc_id": "uuid",
  "parent_id": "uuid",
  "section_number": "2.3",
  "strategy": "recursive | semantic",
  "chunk_index": 0,
  "is_active": true,
  "document_version": 1,
  "created_at": "ISO-8601 timestamp"
}
```

Use HNSW for dense vectors. The initial implementation uses BGE-M3 dense embeddings (1024 dimensions) and sparse BM25-style vectors. Tenant/status/version filtering happens inside the Qdrant query, not after results are returned.

### 7.3 Query flow

```text
Question from authenticated user
  -> resolve tenant membership and role
  -> apply per-user/per-tenant Redis rate limit
  -> query-cache lookup
  -> embed query
  -> tenant-filtered hybrid Qdrant retrieval (top 60)
  -> semantic-first policy where semantic chunks exist
  -> BGE reranking (top 60 -> top 8)
  -> resolve and deduplicate parent contexts (up to 3)
  -> build evidence-only prompt
  -> stream LLM answer and citation metadata
  -> optional grounding check
  -> cache safe result; write query/audit/usage data; trace in LangSmith
```

## 8. Generation Architecture

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `generation/prompts.py` | Own all prompt templates and prompt rendering. | `ANSWER_PROMPT`, `build_answer_messages()` |
| `generation/llm.py` | Create configurable LangChain chat models; stream tokens; retry transient failures; apply fallback provider/model policy. | `get_chat_model()`, `stream_answer()` |
| `generation/grounding.py` | Optional evidence/citation validation behind a feature flag. | `check_grounding()` |

Generation requirements:

- The prompt must instruct the model to answer only from supplied context.
- The prompt must require citations formatted as `[doc: <title>, section: <section_number>]` after factual claims.
- If context is insufficient, the model must say so rather than invent an answer.
- Initial temperature is 0.1–0.2.
- Answer tokens are streamed as Server-Sent Events (SSE).

## 9. Service Layer

Services orchestrate workflows. They may call repositories, storage adapters, retrieval/generation modules, and observability helpers. They must not depend on HTTP objects such as FastAPI `Request` or `Response`.

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `services/document_service.py` | Upload metadata, enqueue ingest/reindex, read/list documents, secure deletion across Postgres/Qdrant/MinIO/cache. | `create_upload()`, `delete_document()`, `request_reindex()` |
| `services/chat_service.py` | Full query orchestration: rate limits, cache, retrieve, rerank, assemble context, stream LLM, persist log/usage. | `run_chat_query()` |
| `services/tenant_service.py` | Tenant/membership CRUD and role changes. | `create_tenant()`, `add_member()`, `change_member_role()` |
| `services/audit_service.py` | Append-only audit-event creation. | `write_audit_event()` |
| `services/usage_service.py` | Token, model, storage, and quota accounting. | `record_model_usage()`, `check_quota()` |

## 10. Schemas and API Contracts

| File | Responsibilities |
|---|---|
| `schemas/auth.py` | Login, token refresh, token response, logout schemas |
| `schemas/documents.py` | Document create/list/detail/status/reindex schemas |
| `schemas/chat.py` | Chat request, citation, SSE event payload schemas |
| `schemas/tenants.py` | Tenant and membership request/response schemas |

All API errors must use one shape:

```json
{
  "code": "machine_readable_error_code",
  "message": "Safe human-readable message",
  "details": {},
  "request_id": "uuid"
}
```

## 11. Observability and Background Tasks

| File | Responsibilities | Important functions / objects |
|---|---|---|
| `observability/logging.py` | Configure structured JSON logs with request/tenant/user identifiers where policy permits. | `configure_logging()` |
| `observability/metrics.py` | Prometheus/OpenTelemetry counters, histograms, and latency metrics. | `record_request_latency()`, `record_retrieval_latency()` |
| `observability/tracing.py` | Initialize LangSmith and wrap ingestion/retrieval/generation operations in traces. | `trace_ingestion()`, `trace_chat()` |
| `tasks/queue.py` | Define the Redis-backed job enqueueing interface. | `enqueue_ingest()`, `enqueue_reindex()` |

Do not send confidential raw content to third-party observability systems unless the deployment policy explicitly permits it.

## 12. Frontend Responsibilities

### 12.1 Route files

| File | Responsibilities |
|---|---|
| `app/layout.tsx` | Root providers, page shell, global navigation, theme/auth provider wiring. |
| `app/page.tsx` | Dashboard/home: recent activity, document shortcuts, chat entry point. |
| `app/login/page.tsx` | Login form and redirect after successful authentication. |
| `app/chat/page.tsx` | Main chat page; owns local streaming state and composes chat components. |
| `app/documents/page.tsx` | Tenant-scoped document list and upload entry point. |
| `app/documents/[id]/page.tsx` | Document details, processing state, reindex/delete actions. |
| `app/admin/tenants/page.tsx` | Admin tenant and membership management. |
| `app/globals.css` | Tailwind imports, design tokens, global styles only. |

### 12.2 UI components

| File | Responsibilities |
|---|---|
| `components/chat/chat-input.tsx` | Query textarea, validation, submit, disable state while streaming. |
| `components/chat/chat-messages.tsx` | Render user/assistant messages and incremental token output. |
| `components/chat/citation-card.tsx` | Render title, section, and authorized document link. |
| `components/documents/document-list.tsx` | Render document table/cards and lifecycle status. |
| `components/documents/document-upload.tsx` | File selection, client-side validation, multipart upload, processing feedback. |
| `components/documents/document-status.tsx` | Status badge for uploaded/processing/ready/failed/deprecated. |
| `components/layout/navbar.tsx` | User menu, logout, global actions. |
| `components/layout/sidebar.tsx` | Navigation and, if enabled, tenant switcher. |
| `components/ui/*` | shadcn/ui primitives; do not place application business logic here. |

### 12.3 Frontend library files

| File | Responsibilities | Important functions |
|---|---|---|
| `lib/api.ts` | Typed backend API wrapper. | `login()`, `list_documents()`, `upload_document()`, `get_document()`, `delete_document()`, `stream_chat_query()` |
| `lib/auth.ts` | Authentication/session state and refresh handling. Prefer secure httpOnly cookies when possible. | `use_auth()`, `refresh_session()` |
| `lib/sse.ts` | Parse streaming chat events and safely dispatch token/citation/error handlers. | `consume_sse_stream()` |
| `lib/utils.ts` | General UI utility functions only. | `cn()` |
| `types/*.ts` | Shared TypeScript API/domain interfaces. | `Document`, `Citation`, `ChatEvent` |

## 13. End-to-End Flows

### 13.1 Upload a document

```text
DocumentUpload.tsx
  -> lib/api.ts: upload_document()
  -> POST /api/v1/documents
  -> api/documents.py
  -> services/document_service.py: create_upload()
  -> storage/s3_adapter.py: upload_original() to MinIO
  -> db/repositories/documents.py: create Document(status=processing)
  -> tasks/queue.py: enqueue_ingest(document_id)
  -> ingestion/worker.py: process_ingest_job()
  -> parser -> normalize -> sections -> chunking -> embed -> index
  -> PostgreSQL ParentContext rows + Qdrant child retrieval chunks
  -> Document(status=ready)
  -> DocumentStatus.tsx refreshes status
```

### 13.2 Ask a question

```text
chat-input.tsx
  -> lib/api.ts: stream_chat_query()
  -> POST /api/v1/chat/stream
  -> api/chat.py
  -> services/chat_service.py: run_chat_query()
  -> core/rate_limit.py
  -> Redis query cache
  -> retrieval/retriever.py + qdrant_search.py
  -> retrieval/rerank.py
  -> retrieval/context_assembly.py
  -> generation/prompts.py + generation/llm.py
  -> SSE tokens and citation events
  -> chat-messages.tsx + citation-card.tsx
  -> query/audit/usage records + LangSmith trace
```

### 13.3 Delete a document

```text
Document detail page
  -> DELETE /api/v1/documents/{id}
  -> document_service.delete_document()
  -> authorize tenant + role
  -> mark document deleting / inactive
  -> delete Qdrant points filtered by tenant_id + doc_id
  -> delete MinIO source/parent objects according to retention policy
  -> delete or retain PostgreSQL records according to retention/audit policy
  -> invalidate Redis entries associated with document/tenant
  -> write audit event
```

## 14. Docker Compose Services

| Service | Purpose |
|---|---|
| `frontend` | Next.js application |
| `backend` | FastAPI and LangChain API |
| `worker` | Asynchronous ingestion and reindex execution |
| `postgres` | Relational database |
| `qdrant` | Dense/sparse vector database |
| `redis` | Cache, rate limiter, queue support |
| `minio` | Free, self-hosted S3-compatible object storage |
| `embedding-service` | Optional self-hosted BGE-M3 inference endpoint |
| `reranker-service` | Optional self-hosted BGE reranker endpoint |

## 15. Implementation Guardrails

1. **No cross-tenant data access.** Every repository and Qdrant query requires a server-derived `tenant_id` filter.
2. **No long ingestion in request handlers.** Upload returns `202 Accepted`; workers do parse/chunk/embed/index work.
3. **No direct MinIO SDK use outside `storage/`.** Use `s3_adapter.py`.
4. **No direct Qdrant calls from API routes.** Retrieval/indexing modules own Qdrant interaction.
5. **No business logic in frontend components.** Place network and session logic in `frontend/lib/`; domain workflows belong to backend services.
6. **No answer without evidence.** Generation receives only assembled retrieved context and must abstain when evidence is insufficient.
7. **No unversioned schema changes.** Use Alembic for Postgres, documented collection migration/reindex procedures for Qdrant.
8. **No secrets in Git.** Use `.env.example` only for placeholder variable names.
9. **No changes to chunking defaults without evaluation.** Compare retrieval quality, latency, and cost with a LangSmith dataset before adopting a new default.
10. **Use async I/O and batched inference.** Database, Redis, Qdrant, object storage, embedding, and reranker clients must avoid blocking the FastAPI event loop.

## 16. Required Testing

| Test level | Coverage |
|---|---|
| Unit | Parsers, text normalization, section splitting, recursive/semantic chunk rules, RBAC checks, cache-key creation, prompt/citation formatting. |
| Integration | Upload -> queued ingest -> MinIO -> Postgres -> Qdrant; tenant filtering; delete/reindex behavior; Redis rate limits/cache. |
| End-to-end | Login, upload, processing state, chat streaming, citations, document deletion, and attempted cross-tenant access. |
| Evaluation | Retrieval Recall@5/10, MRR, citation accuracy, faithfulness/groundedness, answer relevance/correctness, latency/cost. |

## 17. Definition of Done for a Feature

A feature is complete only when it has:

- Tenant and role authorization where required.
- Request/response schema validation and structured errors.
- Audit logging where the action is security-sensitive or changes data.
- Relevant unit/integration tests.
- Safe structured logging and telemetry.
- Updated `ARCHITECTURE.md` if the file ownership, flow, dependency, data contract, or architectural decision changes.
