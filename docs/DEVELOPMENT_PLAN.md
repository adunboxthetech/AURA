# Development Plan

Build one demonstrable phase at a time. `CONTRACTS.md` is the API/schema/config authority. Keep the release zero-card: Neon Free for data/auth/storage, Vercel Hobby for frontend, Azure Container Apps under student credits through May 2027 for backend, then a stateless backend redeploy to Render Free/comparable no-card host. No automatic paid upgrades. Quotas can change.

## Phase 0 — Lean monorepo and demo shell

- Create frontend/backend skeleton, Dockerfile with Tesseract, `.env.example`, Pydantic settings validation, CORS, health endpoint, migrations, Neon connections, Neon Auth client, and private S3-compatible storage adapter.
- Create Vercel/Vite UI shell with PDF.js viewer, Markdown/KaTeX chat shell, read-only demo entry point, and `OFFLINE_DEMO_MODE` banner.
- Implement rate-limit/config foundations and docs for no-card secrets/deploy setup.

**Exit:** CI/local `pytest` and frontend build commands are documented; health works, auth configuration connects, seeded read-only demo can render fixture sources, offline demo runs without network/provider calls.

## Phase 1 — Accounts, data contracts, uploads, OCR, dedup

- Implement exact Postgres tables/indexes and repository methods from `CONTRACTS.md`; owner-scoped access and demo read-only policy.
- Implement source upload/list/status/retry/delete and PDF file range endpoint.
- Enforce 40 MB/500-page PDF limits, 40 OCR-required-page limit, 100 MB/user logical storage, 10 sources/user, and project-wide object/database circuit breakers.
- Validate/count/extract with PyMuPDF; use Tesseract only where needed, one page at a time with `asyncio.Semaphore(1)`; persist extraction method and processed/total pages.
- Add SHA-256 shared-object dedup and transactional reference counting; ensure retries and deletion are safe.
- Add ingestion statuses, polling, stale-job watchdog at 300 seconds without heartbeat/progress, and startup reconciliation. Explain BackgroundTasks process durability limit.

**Exit:** tests cover digital textbook up to configured page limit, mixed/scanned PDF within OCR cap, clean over-OCR-cap failure/recovery advice, upload/user quota, duplicate object refs, concurrent upload race, progress, stale jobs, and cross-user authorization.

## Phase 2 — Embeddings and hybrid RRF retrieval

- Add hosted embedding provider abstraction and config-validated dimensions (default 768); batch 64, bounded backoff/retry for rate limits.
- Implement dense `pgvector` and lexical `tsvector` candidate retrieval, source selection filters, and configurable RRF.
- Persist compact retrieval trace and expose only to owning user.

**Exit:** deterministic fixtures validate dense/lexical/RRF ranks, ownership filters, selected source behavior, retry cap and error handling, and storage footprint.

## Phase 3 — Persistent SSE Q&A and follow-up context

- Add conversations/messages and SSE endpoint contract. Stream status/token/citations/retrieval_trace/abstained/error/done events.
- Add follow-up rewriting using max last 3 turns; enforce max 3,500 context tokens including history/evidence.
- Implement Groq primary with Gemini fallback before first token only; citation validation by retrieved chunk ID; abstention policy calibrated on development examples.
- Implement nonpersistent demo questions and offline deterministic event flow.

**Exit:** tests cover SSE schemas, early fallback vs mid-stream error, token/context budgets, conversation ownership, citation validation, abstention, demo rate limits, disconnect, and offline mode.

## Phase 4 — Complete split-screen MVP and Curriculum Audit

- Finish workspace manager, drag/drop uploader, async statuses/progress, selected source checkboxes, privacy disclosures, Chat and Curriculum Audit views, citation pills, PDF.js exact-page navigation, text-layer highlight for embedded content, OCR Cited Excerpt callout, and collapsible Retrieval Inspector.
- Implement persisted audit endpoint with syllabus modules/topic extraction, semantic+lexical coverage against notes/PYQs, `covered`/`weak`/`missing`, and validated page citations.

**Exit:** end-to-end public demo flow includes no-signup read-only demo; authenticated user upload and Q&A; follow-up query; source filtering; audit; citation navigation; quotas/errors; and deletion. UI is accessible and responsive.

## Phase 5 — Small, honest evaluation baseline

- Curate 5–8 permitted coursework PDFs and 30–50 labeled answerable/unanswerable questions. Freeze development and held-out split; document rights and annotation guide.
- Measure Recall@K, MRR, nDCG@K where graded relevance applies, citation accuracy, abstention precision/recall, TTFT, and p50/p95 latency where sample size supports it.

**Exit:** reproducible scripts/report with actual sample sizes, method, config/date/provider, failures, quota use, and limitations. Do not tune on held-out examples.

## Phase 6 — Optional ablations

- Compare dense vs lexical vs RRF; embedded-text vs OCR pages; chunk size/overlap and OCR thresholds.
- Optional reranker only as stretch goal; compare quality, latency, quota impact, and storage/compute tradeoffs.
- Report negative outcomes and uncertainty.

**Exit:** recorded, fair ablation report; no claim of improvement without measured evidence. Do not make reranker a core MVP dependency.

## Phase 7 — Hardening and deploy

- Harden Neon Auth token validation, ownership, demo restrictions, per-user/IP limits, object privacy, cache/ETag/ranges, dedup races, deletion, quota boundaries, watchdogs, logs, provider failures, and offline presentation.
- Deploy Vercel Hobby frontend, Neon Free DB/Auth/Object Storage, and stateless FastAPI container on Azure Container Apps while user-stated student credits remain through May 2027. Thereafter redeploy same Docker image to Render Free/comparable host. No DB/object migration is needed.
- Document actual free-plan limits, sleep/cold-start impact, account-specific Azure expiration, and safe shutdown if limits change.

**Exit:** deployed URL verified within current quotas, security/privacy disclosures accurate, offline demo functional, and no-card operating boundary documented. Do not claim production-grade durability: in-process BackgroundTasks can be lost.

## Evaluation discipline

Use Recall@K and MRR for ranked retrieval; nDCG@K only with graded labels. Use a written rubric for citation accuracy, answer faithfulness, and abstention precision/recall. Measure TTFT plus p50/p95 latency only when sample size supports stable percentiles. Compare on frozen, permitted data; record provider/model versions and dates without inventing results. Automated judges are supplementary to human review.
