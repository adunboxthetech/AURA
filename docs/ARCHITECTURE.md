# Architecture

## Deployment

```text
Browser ── Vercel Hobby: React + Vite
  ├─ left: workspaces, PDF dropzone/status, Chat, Curriculum Audit, inspector
  └─ right: react-pdf/PDF.js viewer
       │ HTTPS / SSE / CORS
       ▼
FastAPI Docker API (Azure Container Apps → later Render Free)
  ├─ Neon Auth token validation
  ├─ async SQLAlchemy repositories → Neon Postgres
  │    ├─ metadata/conversations/messages/audits
  │    ├─ pgvector dense search + tsvector lexical search → RRF
  │    └─ chunks and citation provenance
  ├─ ObjectStorage → Neon S3-compatible Object Storage
  ├─ PyMuPDF native text + bounded Tesseract OCR
  └─ hosted EmbeddingProvider + Groq/Gemini LLMProvider
```

All persistent application data stays in Neon from day one. Moving the stateless backend image from Azure Container Apps to Render requires deployment/configuration only; there is no database or object migration. Keep current plan rules and quotas verified; never promise a free tier indefinitely.

## Repository layout

```text
.
├── README.md, AGENTS.md, CONTRACTS.md, .env.example
├── docs/PROJECT_BRIEF.md
├── docs/TECH_STACK.md
├── docs/ARCHITECTURE.md
├── docs/DEVELOPMENT_PLAN.md
├── backend/Dockerfile
├── backend/app/{api,core,db,repositories,storage,documents,embeddings,retrieval,providers,services}/
├── backend/tests/
├── frontend/src/{api,components,pages,types}/
├── evaluation/
└── demo-fixtures/
```

## Ingestion flow

1. Authenticated upload request declares `doc_type` and includes PDF. Validate ownership, MIME/signature, 40 MB byte limit, 500 page maximum, user source count, and logical storage quota. Calculate SHA-256 while streaming to a temporary bounded buffer/stream; do not persist user files on local disk.
2. Reuse or create `stored_objects` row for the digest, upload once to private Neon object storage, and increment reference count transactionally. Source record starts `pending`, `processed_pages=0`, `total_pages` initially null; return 202.
3. FastAPI `BackgroundTasks` moves source to `parsing`. Since this execution is in-process, use idempotent phase markers and reconcile stale jobs. A watchdog marks `parsing` or `indexing` with no heartbeat/progress for the configured 300 seconds as `failed` with retry action; startup scan catches jobs interrupted by restart.
4. PyMuPDF validates the file and determines page count. For each page, extract embedded text. Pages below `OCR_MIN_TEXT_CHARS` require OCR. Preflight or count OCR-required pages; if more than 40, fail cleanly before starting expensive OCR, remove partial derived data, and advise chapter-by-chapter upload or a searchable PDF.
5. For eligible files, increment `processed_pages` as each page completes. OCR is one page at a time at configured 150 DPI, protected by `asyncio.Semaphore(1)`, with image pixel and timeout limits. Record extraction method (`embedded` or `ocr`), page, and text. Chunk with page provenance and `section_heading`/`content_type` where available.
6. Status `indexing`: batch hosted embedding calls at 64 chunks; bounded exponential backoff and retry honors provider retry-after; persist chunks, `vector(768)`, and generated/updated `search_tsv` transactionally/idempotently. Source becomes `ready` only when all expected chunks and indexes are available.
7. UI polls source status and displays `processed_pages/total_pages`, current status, and a safe error/retry action.

## File deduplication and deletion

`stored_objects.sha256` is unique. Multiple source rows may reference the same stored object, including across owners. `ref_count` equals the count of referencing source rows and is changed under a transaction/lock. Uploading an existing digest does not transfer or store duplicate bytes. Per-user quota counts logical source bytes for each source, even if physically deduplicated. On source delete, remove its chunks, decrement user's logical quota and object ref count; delete the remote object only when ref_count becomes zero. Retry/reconcile partial object/database failures; never orphan data silently.

## Query/retrieval flow

1. Authenticate request or authorize public demo access. Verify workspace owner unless it is read-only `is_demo=true`. Resolve optional `selected_source_ids`; reject IDs outside workspace/owner and include only ready sources.
2. If conversation exists, verify ownership and workspace. Load at most `MAX_HISTORY_TURNS=3` turns, rewrite follow-up question into a standalone query with a lightweight provider call or deterministic local rule; retain original question. Keep all evidence and history under `MAX_CONTEXT_TOKENS=3500`.
3. Search dense vector and `tsvector` lexical results independently, filtering by `workspace_id` and selected ready sources. Fuse ranks with configurable RRF. Persist diagnostic fields: chunk/source/page, dense/lexical ranks, RRF score, extraction method. Optional reranking is not MVP; Phase 6 only.
4. Apply evidence threshold calibrated using development split. If insufficient, emit abstention. Otherwise send only question plus selected excerpts to LLM. Treat PDF/OCR text as untrusted data.
5. Validate every citation chunk ID against retrieved evidence. Derive source name, excerpt and one-based page from database, not model output. Persist messages and retrieval trace for signed-in users. Demo Q&A is rate limited and nonpersistent; demo corpus remains read-only.

## Curriculum audit

A user selects a syllabus source and optionally notes/PYQ sources. Service extracts syllabus modules/topics from ready syllabus chunks, computes semantic and lexical coverage against ready notes and PYQ chunks, and returns `covered`, `weak`, or `missing`. Every non-missing status includes only validated source/page evidence; missing status has no fabricated citation. Persist input source IDs, algorithm/config versions, structured result JSON, and timestamp in `curriculum_audits`; allow later retrieval. Audit is owner scoped and respects selected source types and readiness. Demo users may GET pre-seeded audit rows read-only; POST audit creation remains forbidden.

## PDF viewer and caching

`GET /api/workspaces/{workspace_id}/sources/{source_id}/file` checks auth, owner, source-workspace link, and object reference. Demo source is readable without login. Proxy object bytes with HTTP byte ranges (`206`, `Content-Range`, `Accept-Ranges`), ETag, `If-None-Match`/`304`, and `Cache-Control: private, max-age=86400`. Configure CORS with `Access-Control-Expose-Headers: Accept-Ranges, Content-Range, Content-Length, ETag, Retry-After` so the Vercel client can inspect range and throttling headers. Never emit public permanent object URLs or storage keys. PDF citation click selects source, navigates to exact page, highlights matching text-layer snippet for embedded text; for OCR-only content show a “Cited Excerpt” callout instead of inaccurate page-coordinate highlight.

## Ownership and demo security

Protected routes require a valid Neon Auth bearer token; backend validates issuer, audience, expiry, signature/JWKS and derives `owner_id` from `sub`. All repository access includes owner/workspace predicates; child IDs are never trusted alone. Demo workspace rows use `is_demo=true`, `owner_id='__demo__'`, are seeded read-only, and cannot be edited/deleted/uploaded into. Demo Q&A can use ephemeral/nonpersistent messages under stricter IP limits. Rate limit per user for authenticated requests and per IP for anonymous demo. API errors follow `CONTRACTS.md`.

## Reliability and quotas

Set per-user limits to 100 MB logical stored PDFs and 10 sources, plus project-wide object/database circuit breakers; many individually compliant users can still exhaust a shared free project. The upload cap is 40 MB; digital PDFs may be up to 500 pages; only up to 40 pages may require OCR. One process-wide OCR semaphore prevents parallel page rasterization. Batch embeddings (64) with bounded retries. Stale-job watchdog uses configured timeout (300 seconds), marks stalled work failed with retry affordance; run periodic checks and startup reconciliation. BackgroundTasks are not durable; this design is appropriate for bounded capstone usage, not high-reliability production jobs.

`OFFLINE_DEMO_MODE` serves fixtures and deterministic responses without calling Neon, OCR, embedding, or LLM providers. Clearly identify demo data/results. If a provider quota is exhausted, show safe retry/limit errors; do not enable billing.
