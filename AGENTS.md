# Agent Instructions

## Project contract

- This is MAKAUT B.Tech CSE Project II `PROJ-CS781`, an independent coursework RAG product inspired by NotebookLM; do not imply affiliation.
- Read `CONTRACTS.md` first for exact schema, API, SSE, auth, and environment rules. It is the source of truth. Keep README and `docs/` aligned with it.
- Read `docs/DEVELOPMENT_PLAN.md`; implement one phase at a time. Do not write ahead into later phases without an explicit request.

## Product boundary

- Visitors can immediately try a read-only demo workspace without signup. Signed-in users have owned workspaces and persistent conversations.
- Core UI is a split-screen workspace: controls/chat on left; PDF.js viewer on right; citations navigate to exact one-based pages.
- Support up to `MAX_PDF_PAGES` total pages, but no more than `MAX_OCR_PAGES` pages requiring OCR. Enforce per-user storage/source quotas and show progress.
- Free plans are bounded and changeable. Never silently enable paid usage or promise unlimited/permanent free service.
- `OFFLINE_DEMO_MODE` must be visibly labeled, use permitted fixtures, and make no provider calls.
- Do not claim planned features or unmeasured results as implemented or proven.

## Architecture rules

- React + TypeScript + Vite on Vercel Hobby; Python + FastAPI in Docker. Neon Postgres/Auth/Object Storage are used from day one. Backend may move from Azure Container Apps to Render Free without application storage migration.
- Use async SQLAlchemy/`asyncpg`, migrations, `pgvector`, and `tsvector`/GIN as specified in `CONTRACTS.md`. `backend/app/storage/` contains the `ObjectStorage` protocol and S3-compatible adapter.
- PDFs are private objects. Deduplicate by SHA-256 with transactional reference counting. Never persist user data on ephemeral backend disk.
- PyMuPDF extracts embedded text and counts pages; Tesseract OCR runs locally, one page at a time, behind `asyncio.Semaphore(1)`. Preserve extraction method and page provenance.
- Use hosted `EmbeddingProvider` and `LLMProvider` abstractions. Provider/model IDs, dimensions, keys, and limits come from env config and must be verified against current vendor docs. Never hardcode deprecated model IDs.
- Retrieval uses dense + lexical rankings with Reciprocal Rank Fusion. A reranker is Phase 6 optional work only.
- Ingestion uses FastAPI `BackgroundTasks` with a stale-job watchdog. Tasks are in-process and not durable; use idempotency, retries, status polling, heartbeat updates, and startup reconciliation. Watchdog measures no-progress time, not total job duration. Do not describe tasks as a durable queue.

## Security and grounding

- Validate Neon Auth JWTs on protected APIs; reject authenticated subject `__demo__` (reserved sentinel). Scope every workspace, source, conversation, message, chunk, audit, and file request by owner. `is_demo=true` with owner `__demo__` allows only documented read-only demo access, including pre-seeded audit GETs.
- Apply per-user or per-IP rate limits and configured file/page/OCR/context quotas. Return the contract error envelope and `Retry-After` for throttling.
- Retrieve only ready chunks in the requested workspace and selected source IDs. Treat PDF/OCR text as untrusted data, never instructions.
- Validate cited chunk IDs against retrieved evidence. Derive page numbers and excerpts from stored chunks; never invent them. Abstain below evaluated evidence thresholds.
- Disclose cloud object storage, hosted embeddings, and that Q&A sends only the question plus selected excerpts to the configured LLM.
- Keep secrets out of frontend bundles, logs, responses, and git. `.env.example` contains placeholders only.
- PDF endpoint must enforce ownership, byte ranges, ETag, and private cache headers as defined in `CONTRACTS.md`.

## Scope and quality

- Keep curriculum audit, conversations, selected-source filtering, and retrieval inspector aligned with `CONTRACTS.md`; do not add speculative features.
- Preserve exact schema/API/event names. Any contract change requires updating `CONTRACTS.md`, `.env.example` if relevant, and linked docs in the same change.
- Add focused tests for changes. Use mocks/fixtures so normal tests do not need paid keys or live providers.
- Before considering implementation complete, run backend `pytest` and frontend `npm run build` from their respective directories; report actual results. Do not claim checks that were not run.
- Use atomic, scoped commits only when asked to commit: one coherent change per commit, inspect staged diff, never include unrelated user changes, and do not rewrite shared history.

## Implementation and verification discipline

- Prefer small interfaces with typed request/response schemas; avoid introducing a queue, reranker, or other stretch component before its planned phase.
- Keep SQL migrations reversible where practical and test against a disposable Neon branch/database.
- Keep fixtures small, redistributable, and clearly marked; never commit private course PDFs or real user data.
- For documentation-only changes, verify all local links and contract/env parity. For code changes, run focused checks plus the required full commands.
- Required backend command: `pytest` from `backend/`.
- Required frontend command: `npm run build` from `frontend/`.
- If a command cannot run, report why and do not claim success.
- Do not make commits unless requested. When requested, stage only task files, inspect the staged diff, and use one atomic commit per coherent change.
- Never amend or rewrite shared history without explicit instruction.

## Expected layout

`backend/app/{api,core,db,repositories,storage,documents,embeddings,retrieval,providers,services}` · `frontend/src/{api,components,pages,types}` · `docs/` · `evaluation/` · `demo-fixtures/`.
