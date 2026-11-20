# PROJ-CS781 Technical Contracts

This is the authoritative contract for Postgres schema, REST, authentication, SSE, and environment variables. Change it before or alongside implementation changes; update `.env.example` and `docs/` in the same change. SQL types below use PostgreSQL. UUIDs are generated server-side. Timestamps are UTC `timestamptz`.

## 1. Database schema

Neon-managed auth tables live in `neon_auth` and are provisioned by Neon Auth; the app must not modify them. `owner_id` is `text NOT NULL` on every application row that has an owner. Seeded demo rows use the reserved sentinel `__demo__`; authenticated Neon Auth subjects equal to this sentinel are rejected.

### Enums

```sql
CREATE TYPE source_status AS ENUM ('pending','parsing','indexing','ready','failed','deleting');
CREATE TYPE doc_type AS ENUM ('syllabus','notes','pyq','other');
CREATE TYPE chunk_content_type AS ENUM ('text','table','formula','mixed');
CREATE TYPE extraction_method AS ENUM ('embedded','ocr');
CREATE TYPE message_role AS ENUM ('user','assistant');
```

### Tables and exact columns

```sql
workspaces (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id text NOT NULL,
  name text NOT NULL CHECK (length(trim(name)) BETWEEN 1 AND 120),
  is_demo boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CHECK ((is_demo AND owner_id = '__demo__') OR (NOT is_demo AND owner_id <> '__demo__')),
  UNIQUE (id, owner_id)
)

stored_objects (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sha256 char(64) NOT NULL UNIQUE,
  object_key text NOT NULL UNIQUE,
  size_bytes bigint NOT NULL CHECK (size_bytes > 0),
  ref_count integer NOT NULL DEFAULT 0 CHECK (ref_count >= 0),
  state text NOT NULL CHECK (state IN ('uploading','ready','deleting','failed')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
)

source_documents (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  owner_id text NOT NULL,
  stored_object_id uuid NOT NULL REFERENCES stored_objects(id) ON DELETE RESTRICT,
  filename text NOT NULL,
  doc_type doc_type NOT NULL DEFAULT 'other',
  status source_status NOT NULL DEFAULT 'pending',
  total_pages integer NULL CHECK (total_pages IS NULL OR total_pages BETWEEN 1 AND 500),
  processed_pages integer NOT NULL DEFAULT 0 CHECK (processed_pages >= 0),
  error_code text NULL,
  error_message text NULL,
  retryable boolean NOT NULL DEFAULT false,
  extraction_summary jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  heartbeat_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id, workspace_id, owner_id),
  FOREIGN KEY (workspace_id, owner_id) REFERENCES workspaces(id, owner_id) ON DELETE CASCADE
)

chunks (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL,
  source_id uuid NOT NULL,
  owner_id text NOT NULL,
  ordinal integer NOT NULL CHECK (ordinal >= 0),
  text text NOT NULL,
  section_heading text NULL,
  content_type chunk_content_type NOT NULL DEFAULT 'text',
  page_start integer NOT NULL CHECK (page_start >= 1),
  page_end integer NOT NULL CHECK (page_end >= page_start),
  page_segments_json jsonb NOT NULL,
  extraction_method extraction_method NOT NULL,
  embedding vector(768) NULL,
  search_tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', coalesce(text,''))) STORED,
  content_sha256 char(64) NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (source_id, ordinal),
  FOREIGN KEY (source_id, workspace_id, owner_id)
    REFERENCES source_documents(id, workspace_id, owner_id) ON DELETE CASCADE
)

conversations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL,
  owner_id text NOT NULL,
  title text NOT NULL DEFAULT 'New conversation',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id, workspace_id, owner_id),
  FOREIGN KEY (workspace_id, owner_id) REFERENCES workspaces(id, owner_id) ON DELETE CASCADE
)

messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id uuid NOT NULL,
  workspace_id uuid NOT NULL,
  owner_id text NOT NULL,
  role message_role NOT NULL,
  content text NOT NULL,
  citations_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  retrieval_trace_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  abstained boolean NOT NULL DEFAULT false,
  original_question text NULL,
  rewritten_question text NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (conversation_id, workspace_id, owner_id)
    REFERENCES conversations(id, workspace_id, owner_id) ON DELETE CASCADE
)

curriculum_audits (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid NOT NULL,
  owner_id text NOT NULL,
  syllabus_source_id uuid NOT NULL,
  comparison_source_ids uuid[] NOT NULL,
  algorithm_version text NOT NULL,
  status text NOT NULL DEFAULT 'processing' CHECK (status IN ('processing','ready','failed')),
  error_code text NULL,
  error_message text NULL,
  result_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (workspace_id, owner_id) REFERENCES workspaces(id, owner_id) ON DELETE CASCADE,
  FOREIGN KEY (syllabus_source_id, workspace_id, owner_id)
    REFERENCES source_documents(id, workspace_id, owner_id) ON DELETE CASCADE
)
```

For schema portability, implement dimensions as migration-defined `vector(768)` matching `EMBEDDING_DIMENSIONS=768`; changing dimensions requires a reviewed migration/re-embedding, not just an env edit. `page_segments_json` is an array of `{page_number, text, extraction_method}` records; no bounding-box field is part of MVP.

### Required indexes

```sql
CREATE INDEX ix_workspaces_owner_updated ON workspaces(owner_id, updated_at DESC) WHERE is_demo = false;
CREATE INDEX ix_sources_workspace_status ON source_documents(workspace_id, status, created_at DESC);
CREATE INDEX ix_sources_owner_workspace ON source_documents(owner_id, workspace_id, created_at DESC);
CREATE INDEX ix_sources_object ON source_documents(stored_object_id);
CREATE INDEX ix_chunks_workspace_source_page ON chunks(workspace_id, source_id, page_start, page_end);
CREATE INDEX ix_chunks_owner_workspace ON chunks(owner_id, workspace_id);
CREATE INDEX ix_chunks_search_tsv ON chunks USING gin(search_tsv);
CREATE INDEX ix_chunks_embedding_hnsw ON chunks USING hnsw (embedding vector_cosine_ops) WHERE embedding IS NOT NULL;
CREATE INDEX ix_conversations_owner_workspace_updated ON conversations(owner_id, workspace_id, updated_at DESC);
CREATE INDEX ix_messages_conversation_created ON messages(conversation_id, created_at);
CREATE INDEX ix_messages_owner_workspace ON messages(owner_id, workspace_id);
CREATE INDEX ix_audits_owner_workspace_created ON curriculum_audits(owner_id, workspace_id, created_at DESC);
```

All queries still include explicit workspace/owner predicates; indexes are not authorization. Demo data is seeded with `is_demo=true`, `owner_id='__demo__'`; no regular user can mutate it. Every application row uses a non-null owner key so composite foreign keys cannot bypass validation through SQL NULL semantics.

### Quota and deduplication invariants

- `MAX_USER_SOURCES` counts a user's active source rows across all owned workspaces. `MAX_USER_STORAGE_MB` counts logical PDF bytes per user even when bytes are deduplicated. Also enforce project-wide circuit breakers: `MAX_GLOBAL_OBJECT_STORAGE_MB` checks unique stored-object bytes, and `MAX_GLOBAL_DB_USAGE_MB` checks `pg_database_size`; reject new ingestion before the configured headroom is consumed. Per-user quotas alone do not cap aggregate usage.
- A source references one `stored_objects` row. `ref_count` must equal referencing source rows. On duplicate SHA-256, reuse the existing object only in `ready` state and increment ref count atomically. Serialize first-upload races on the digest; compensate/remove partial object on failed metadata commit.
- Source deletion atomically removes derived records and source reference, then deletes remote object only when the final reference count becomes zero. `deleting` and recovery paths handle partial failures idempotently.
- `processed_pages` ranges from 0 through `total_pages`; `total_pages` is populated after PyMuPDF validation. Enforce `processed_pages <= total_pages` once total_pages is non-null. Refresh `heartbeat_at` on each successful page/batch/stage progress; the stale watchdog measures time since heartbeat, not total job duration. Files over 500 pages return `413 PDF_PAGE_LIMIT_EXCEEDED`. Files requiring OCR on more than 40 pages return `413 OCR_PAGE_LIMIT_EXCEEDED` before embedding/indexing, with the user-facing message: “This PDF has more scanned pages than this free demo can process. Upload chapters separately or provide a searchable PDF.” Explain the same recovery path in the UI.
- `search_tsv` is generated from chunk text. `embedding` is null until successful indexing. A source is `ready` only when all its chunks are indexed.

## 2. Authentication and authorization

- Neon Auth owns sign-up/sign-in/session lifecycle. The frontend uses Neon Auth's supported client/API; it does not collect or store passwords itself.
- Protected API requests use `Authorization: Bearer <Neon Auth access token>`. Backend validates signature from configured JWKS, issuer, audience, and expiry; `owner_id` is the verified `sub`. Reject a validly signed token whose `sub` equals reserved `__demo__` with `401 INVALID_AUTH_SUBJECT`. Verify the exact token/JWKS integration against current Neon Auth docs during implementation. Missing/invalid token returns `401`.
- Every workspace/source/chunk/conversation/message/audit/file query includes owner scope. Child resource IDs alone never authorize access. Cross-owner access returns `404` to avoid existence disclosure.
- Demo endpoints allow read-only access to seeded `is_demo=true`, `owner_id='__demo__'` workspace. No upload, mutation, delete, persisted chat, or audit creation is allowed there. Demo Q&A is nonpersistent and IP-rate-limited. Read-only demo audit GET endpoints may return pre-seeded audit results. `GET /api/demo/session` returns the demo workspace metadata.
- Demo and user routes cannot be used to enumerate other workspaces. Enforce request rate limits per authenticated `owner_id`, and per IP for demo/anonymous access; return `429` and `Retry-After`.

## 3. REST API

Base path `/api`. JSON uses `application/json`; upload uses `multipart/form-data`; IDs are UUID strings; timestamps are ISO-8601 UTC. Every non-streaming error is:

```json
{"error":{"code":"SOURCE_NOT_FOUND","message":"Source was not found.","retryable":false,"request_id":"uuid"}}
```

`details` may be included for field validation only; never include secrets, stack traces, or cross-owner existence. Rate-limit errors include HTTP `Retry-After`.

### Health and demo/auth

- `GET /api/health` → `200 {"status":"ok","version":"string"}`. No credentials/provider internals.
- Neon Auth signup/signin/signout routes are provided by configured Neon Auth endpoint and client, not duplicated by FastAPI.
- `GET /api/auth/session` requires bearer token → `200 {"user":{"id":"string","email":"string|null"}}`; invalid/missing auth → `401`.
- `GET /api/demo/session` → `200 {"demo":true,"workspace":{"id":"uuid","name":"string"},"read_only":true,"persistent_chat":false}`.

### Workspaces

- `GET /api/workspaces` → owned workspaces only; response `{"items":[Workspace]}`. Demo workspace is obtained from demo endpoint.
- `POST /api/workspaces` body `{ "name":"string" }` → `201 Workspace`.
- `GET /api/workspaces/{workspace_id}` → `200 Workspace`.
- `PATCH /api/workspaces/{workspace_id}` body `{ "name":"string" }` → `200 Workspace`.
- `DELETE /api/workspaces/{workspace_id}` → `204`; cascades user rows and decrements object refs; demo workspace deletion is forbidden (`403`).

`Workspace = {"id":"uuid","name":"string","is_demo":false,"source_count":0,"storage_used_bytes":0,"created_at":"datetime","updated_at":"datetime"}`.

### Sources

- `POST /api/workspaces/{workspace_id}/sources` multipart fields: `file` (PDF), `doc_type` (`syllabus|notes|pyq|other`, default `other`). → `202 Source`. Enforces 40 MB, 500 total pages, 40 OCR-required pages, max 10 active sources, and 100 MB logical user storage.
- `GET /api/workspaces/{workspace_id}/sources` → `200 {"items":[Source]}`.
- `GET /api/workspaces/{workspace_id}/sources/{source_id}` → `200 Source`, including live progress.
- `POST /api/workspaces/{workspace_id}/sources/{source_id}/retry` → `202 Source`; allowed only for owner and retryable `failed` sources. Demo returns `403`.
- `DELETE /api/workspaces/{workspace_id}/sources/{source_id}` → `202 {"id":"uuid","status":"deleting"}`; final removal is idempotent/background-safe.
- `GET /api/workspaces/{workspace_id}/sources/{source_id}/file` → PDF bytes, supports `Range`, `If-None-Match`; statuses `200`, `206`, `304`, or standard error. Required response headers: `Accept-Ranges: bytes`, `ETag: "sha256-<hex>"`, `Cache-Control: private, max-age=86400`, `Content-Type: application/pdf`, `Content-Disposition: inline; filename="safe-name.pdf"`. For `206`: `Content-Range: bytes start-end/total`, accurate `Content-Length`. CORS must include `Access-Control-Expose-Headers: Accept-Ranges, Content-Range, Content-Length, ETag, Retry-After` so PDF.js and frontend rate-limit handling can read them cross-origin. Never expose object key or public URL.

`Source = {"id":"uuid","workspace_id":"uuid","filename":"string","doc_type":"syllabus|notes|pyq|other","status":"pending|parsing|indexing|ready|failed|deleting","total_pages":500,"processed_pages":125,"size_bytes":1234,"extraction_summary":{"embedded_pages":120,"ocr_pages":5},"error_code":null,"error_message":null,"retryable":false,"created_at":"datetime","updated_at":"datetime"}`. `total_pages` may be null before validation; numbers above are examples.

### Conversations and messages

- `GET /api/workspaces/{workspace_id}/conversations` → `200 {"items":[Conversation]}` (owner only).
- `POST /api/workspaces/{workspace_id}/conversations` body `{ "title":"string|null" }` → `201 Conversation`.
- `GET /api/workspaces/{workspace_id}/conversations/{conversation_id}` → `200 Conversation`.
- `DELETE /api/workspaces/{workspace_id}/conversations/{conversation_id}` → `204`, deleting messages.
- `GET /api/workspaces/{workspace_id}/conversations/{conversation_id}/messages?limit=50&before=<datetime>` → `200 {"items":[Message],"next_before":"datetime|null"}`.
- `DELETE /api/workspaces/{workspace_id}/conversations/{conversation_id}/messages/{message_id}` → `204`, owner only. Only user messages may be individually removed; assistant messages are retained for citation integrity.

`Conversation = {"id":"uuid","workspace_id":"uuid","title":"string","created_at":"datetime","updated_at":"datetime"}`. `Message = {"id":"uuid","conversation_id":"uuid","role":"user|assistant","content":"string","citations_json":[],"retrieval_trace_json":[],"abstained":false,"created_at":"datetime"}`. Demo chat is not persisted.

### Q&A SSE

- `POST /api/workspaces/{workspace_id}/questions` requires bearer token except read-only demo access. Body:

```json
{"question":"string","conversation_id":"uuid|null","selected_source_ids":["uuid"]}
```

`selected_source_ids` may be omitted or empty to include all ready sources; if supplied, every ID must belong to workspace and owner, otherwise `422 INVALID_SOURCE_SELECTION`. A null conversation creates one for authenticated users. Demo questions are ephemeral and `conversation_id` must be null. Response `200 text/event-stream`; events are defined below. Non-stream validation/auth failures use normal error envelope.

### Curriculum Audit

- `POST /api/workspaces/{workspace_id}/curriculum-audit` body:

```json
{"syllabus_source_id":"uuid","comparison_source_ids":["uuid"]}
```

Syllabus source must be ready and `doc_type=syllabus`; comparison IDs must be ready owned `notes` or `pyq` sources. Returns `202 {"audit_id":"uuid","status":"processing"}` for async work. Demo audit creation is forbidden (`403`).
- `GET /api/workspaces/{workspace_id}/curriculum-audit/{audit_id}` → `200 Audit` when ready or failed; `202 {"id":"uuid","status":"processing"}` while processing. On the seeded demo workspace, return only pre-seeded audit rows with `owner_id='__demo__'`.
- `GET /api/workspaces/{workspace_id}/curriculum-audits` → `200 {"items":[AuditSummary]}` for owned workspaces and the seeded demo workspace; demo results are pre-seeded and read-only.

`Audit = {"id":"uuid","workspace_id":"uuid","syllabus_source_id":"uuid","comparison_source_ids":["uuid"],"status":"processing|ready|failed","topics":[{"module":"string|null","topic":"string","status":"covered|weak|missing","coverage_score":0.0,"citations":[{"source_id":"uuid","source_name":"string","doc_type":"notes|pyq","page_start":1,"page_end":1,"chunk_id":"uuid","excerpt":"string"}]}],"error_code":"string|null","error_message":"string|null","created_at":"datetime","updated_at":"datetime"}`. `AuditSummary = {"id":"uuid","workspace_id":"uuid","syllabus_source_id":"uuid","status":"processing|ready|failed","topic_count":0,"error_code":"string|null","created_at":"datetime","updated_at":"datetime"}`. `topics` is empty while processing or failed. `coverage_score` is an experimental score, not a calibrated probability. `missing` topics have an empty citation array.

## 4. SSE event contract

Each message is UTF-8 `event: <name>\ndata: <JSON>\n\n`. All payloads include `request_id`. IDs are strings. Events are ordered: `status` one or more; then either `abstained` or `token` events; then `citations`; optional `retrieval_trace` if requested; then `done`. A terminal `error` may occur at any point and is followed by stream close; no provider fallback after first token.

```json
{"event":"status","data":{"request_id":"uuid","stage":"rewriting|retrieving|generating|saving","message":"string"}}
{"event":"token","data":{"request_id":"uuid","text":"string"}}
{"event":"citations","data":{"request_id":"uuid","items":[{"chunk_id":"uuid","source_id":"uuid","source_name":"string","page_start":1,"page_end":2,"excerpt":"string","extraction_method":"embedded|ocr"}]}}
{"event":"retrieval_trace","data":{"request_id":"uuid","items":[{"chunk_id":"uuid","source_name":"string","page_start":1,"dense_rank":1,"lexical_rank":null,"rrf_score":0.03,"extraction_method":"embedded|ocr"}]}}
{"event":"abstained","data":{"request_id":"uuid","abstained":true,"reason":"insufficient_evidence","message":"The selected sources do not contain enough evidence to answer this."}}
{"event":"error","data":{"request_id":"uuid","code":"PROVIDER_RATE_LIMITED","message":"The answer service is temporarily unavailable. Please retry later.","retryable":true}}
{"event":"done","data":{"request_id":"uuid","conversation_id":"uuid|null","user_message_id":"uuid|null","assistant_message_id":"uuid|null","abstained":false}}
```

`token.text` is a delta, not the complete answer. `citations.items` only contains citations validated against chunks retrieved for this request. Retrieval trace is owner-only and optional; public demo trace must not reveal user data. If abstaining, emit no `token`; citations may be empty. `done.abstained` reflects final state. In demo mode message IDs and conversation ID are null.

## 5. Environment variables

`.env.example` is the canonical placeholder list; no real secrets belong in it. Every key appears in the table and vice versa. Values shown are defaults/illustrative except secrets and provider/model IDs, which must be supplied and verified at implementation time.

| Variable | Default/example | Meaning |
|---|---|---|
| `APP_ENV` | `development` | Runtime environment |
| `API_HOST` | `0.0.0.0` | Bind address |
| `PORT` | `8000` | Container port |
| `FRONTEND_ORIGINS` | local + placeholder Vercel origin | Allowed CORS origins |
| `CORS_EXPOSE_HEADERS` | `Accept-Ranges, Content-Range, Content-Length, ETag, Retry-After` | Response headers readable by cross-origin frontend |
| `VITE_API_BASE_URL` | `http://localhost:8000/api` | Public frontend API base URL |
| `VITE_NEON_AUTH_URL` | placeholder | Public Neon Auth endpoint for frontend SDK |
| `OFFLINE_DEMO_MODE` | `false` | Fixture-only mode; no external calls |
| `LOG_LEVEL` | `INFO` | Logging verbosity |
| `DATABASE_URL` | placeholder asyncpg URL | Neon Postgres connection string |
| `NEON_AUTH_URL` | placeholder | Neon Auth base URL |
| `NEON_AUTH_ISSUER` | placeholder | Expected token issuer |
| `NEON_AUTH_AUDIENCE` | placeholder | Expected API audience |
| `NEON_AUTH_JWKS_URL` | placeholder | Signing key set for token validation |
| `OBJECT_STORAGE_ENDPOINT` | placeholder | S3-compatible endpoint |
| `OBJECT_STORAGE_REGION` | placeholder | Storage region/required SDK value |
| `OBJECT_STORAGE_BUCKET` | `coursework-pdfs` | Private bucket |
| `OBJECT_STORAGE_ACCESS_KEY_ID` | placeholder | Server-only credential |
| `OBJECT_STORAGE_SECRET_ACCESS_KEY` | placeholder | Server-only credential |
| `OBJECT_STORAGE_ADDRESSING_STYLE` | `path` | S3 client addressing mode |
| `PDF_CACHE_MAX_AGE_SECONDS` | `86400` | Private viewer cache lifetime |
| `MAX_UPLOAD_MB` | `40` | Max PDF upload size |
| `MAX_PDF_PAGES` | `500` | Max total PDF pages |
| `MAX_OCR_PAGES` | `40` | Max pages requiring OCR |
| `MAX_USER_STORAGE_MB` | `100` | Logical source storage quota per user |
| `MAX_USER_SOURCES` | `10` | Active source count per user |
| `MAX_GLOBAL_OBJECT_STORAGE_MB` | `4500` | Circuit breaker under current Neon object-storage allowance |
| `MAX_GLOBAL_DB_USAGE_MB` | `850` | Circuit breaker under current Neon database allowance |
| `OCR_MIN_TEXT_CHARS` | `40` | Below this page text length, OCR is required |
| `OCR_RENDER_DPI` | `150` | Page rasterization target |
| `OCR_MAX_PAGE_PIXELS` | `12000000` | Raster resource ceiling |
| `OCR_PAGE_TIMEOUT_SECONDS` | `30` | Per-page OCR timeout |
| `INGESTION_STALE_TIMEOUT_SECONDS` | `300` | Parsing/indexing stale threshold |
| `EMBEDDING_BATCH_SIZE` | `64` | Texts per provider request batch |
| `EMBEDDING_MAX_RETRIES` | `5` | Max transient retry count |
| `EMBEDDING_BACKOFF_BASE_SECONDS` | `1` | Exponential backoff base |
| `EMBEDDING_PROVIDER` | `google` | Configured embedding adapter |
| `EMBEDDING_MODEL` | `configure-current-supported-model` | Active model ID, verified at implementation |
| `EMBEDDING_DIMENSIONS` | `768` | Must match DB vector migration/model output |
| `MAX_HISTORY_TURNS` | `3` | Conversation turns available to query rewrite |
| `MAX_CONTEXT_TOKENS` | `3500` | Strict history + evidence + prompt budget |
| `RRF_K` | `60` | Reciprocal rank fusion constant |
| `DENSE_CANDIDATE_LIMIT` | `30` | Dense candidate count |
| `LEXICAL_CANDIDATE_LIMIT` | `30` | Lexical candidate count |
| `RETRIEVAL_TOP_K` | `8` | Final evidence count |
| `ABSTENTION_THRESHOLD` | placeholder | Evaluated evidence threshold, not a probability |
| `LLM_PRIMARY` | `groq` | Primary streaming provider |
| `LLM_FALLBACK` | `gemini` | Fallback before first token only |
| `GROQ_API_KEY` | placeholder | Server-side provider secret |
| `GEMINI_API_KEY` | placeholder | Server-side provider secret |
| `LLM_PRIMARY_MODEL` | `configure-current-supported-model` | Verified active primary model ID |
| `LLM_FALLBACK_MODEL` | `configure-current-supported-model` | Verified active fallback model ID |
| `LLM_REQUEST_TIMEOUT_SECONDS` | `60` | Provider timeout |
| `RATE_LIMIT_USER_PER_MINUTE` | `20` | Authenticated request rate |
| `RATE_LIMIT_IP_PER_MINUTE` | `10` | Anonymous/demo IP rate |
| `RATE_LIMIT_UPLOADS_PER_HOUR` | `10` | Per-user upload rate; independent of source quota |


