# Technical Stack

`CONTRACTS.md` is the exact authority for schema, API, events, auth, and environment variable names. This document explains choices and boundaries.

| Layer | Choice | Purpose |
|---|---|---|
| Frontend | React + TypeScript + Vite, Vercel Hobby | Public student UI |
| PDF viewer | `react-pdf` / PDF.js | Page navigation and text layer |
| Chat rendering | `react-markdown`, `remark-math`, `rehype-katex` | Markdown, tables, formulas |
| Auth | Neon Auth / Managed Better Auth | Accounts and auth tokens; read-only demo path is separate |
| API | Python + FastAPI in Docker | REST, SSE, async ingestion |
| Database | Neon Postgres Free, async SQLAlchemy 2.x + `asyncpg` | Metadata, conversations, chunks, `pgvector`, `tsvector` |
| File storage | Neon S3-compatible Object Storage via one `S3CompatibleStorageAdapter` | Private PDF persistence and deduplicated objects |
| PDF/OCR | PyMuPDF + Tesseract (`pytesseract`) | Native text extraction and bounded local OCR |
| Embeddings | Configured hosted `EmbeddingProvider` | Dense retrieval vectors |
| Search | PostgreSQL `pgvector` + `tsvector`, Reciprocal Rank Fusion | Hybrid retrieval |
| Generation | Provider-neutral `LLMProvider`; Groq primary, Gemini fallback pre-first-token | Grounded SSE answers |
| Optional reranking | `Reranker` interface | Phase 6 ablation only; not core MVP |

## Storage and ingestion guardrails

- Defaults: `MAX_UPLOAD_MB=40`, `MAX_PDF_PAGES=500`, `MAX_OCR_PAGES=40`, `MAX_USER_STORAGE_MB=100`, `MAX_USER_SOURCES=10`, plus project circuit breakers `MAX_GLOBAL_OBJECT_STORAGE_MB=4500` and `MAX_GLOBAL_DB_USAGE_MB=850`. Neon currently advertises 1 GB Postgres and 5 GB object storage per free project; verify [current Neon limits](https://neon.com/blog/neon-free-plan-1-gb-per-project) at deployment. Per-user quotas alone cannot prevent aggregate exhaustion.
- Use native text when page text is sufficient. Count all pages requiring OCR before OCR work; if over 40, fail before indexing with advice to upload chapters or a searchable PDF.
- OCR one page at a time, target 150 DPI, enforce rendered-pixel/page timeouts, and guard all OCR with one `asyncio.Semaphore(1)` per process.
- Persist progress in source row (`processed_pages`, `total_pages`) so clients can poll reliably.
- FastAPI `BackgroundTasks` are in-process, not durable. Add stale parsing/indexing watchdog at 300 seconds, safe retry, idempotent writes, and startup reconciliation. Use a durable queue only if demonstrated needs justify it.
- Deduplicate binary objects by SHA-256; source records reference a `stored_objects` row with `ref_count`. User quotas count each user's logical source size even if bytes are shared, preventing quota bypass.

## Search and provider design

Postgres stores `vector(768)` by default, with a migration/configuration requirement if dimensions change. Use a currently supported hosted embedding model configured only through `.env`; use dimension reduction only if provider explicitly supports it and evaluation verifies quality. Batch 64 texts, retry transient 429/5xx failures with bounded exponential backoff and jitter, respect `Retry-After`, and stop after configured attempts.

Dense and lexical candidates are independently workspace/source scoped. Fuse with RRF; persist compact ranks/scores for inspection/evaluation. Keep reranking outside MVP phases 0–4. Query rewriting uses at most the last three conversation turns, avoids inventing entities, and stores original and rewritten query in retrieval trace.

Groq is primary LLM; Gemini fallback only before first streamed token. Once a token is emitted, errors terminate the stream rather than switching providers and duplicating output. Configure model IDs, vector dimensions, tokens, quotas, keys, and limits via `.env` / `.env.example`; verify current vendor docs at implementation time. Do not name deprecated or unverified model IDs in specifications.

## Neon object delivery and egress

Use S3-compatible storage, private bucket, server-side credentials, and proxy through the API after ownership checks. Implement byte ranges, ETag/If-None-Match, and `Cache-Control: private, max-age=86400` for PDF viewer requests. Keep browser cache private because documents are user-specific. Use `processed_pages`, selected sources, and excerpts—not whole books—for LLM context. Free-tier storage/egress and provider allowances change; enforce quotas, monitor usage, and fail without auto-billing.

## Deployment

Vercel Hobby serves frontend only. Azure Container Apps runs the stateless Docker backend through current Azure for Students credit period; after May 2027, target Render Free/comparable no-card Docker host with same image. Neon remains the database, auth, and file store, so no Postgres or object migration is planned for that transition. Render's idle sleep/cold start is acceptable for an academic demo; no user files or database live on its ephemeral filesystem.
