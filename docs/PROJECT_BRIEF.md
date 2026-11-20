# Project Brief

## Identity

**Intelligent Coursework Knowledge Retrieval and Curriculum Analysis System using Retrieval-Augmented Generation (RAG)** is MAKAUT B.Tech CSE Project II, `PROJ-CS781`. It is an independent student product inspired by NotebookLM's source-grounded study workflow; no affiliation is claimed.

## User goal

A student opens the deployed URL, optionally signs in, creates a subject workspace, uploads a syllabus, notes, previous-year questions, or a digital textbook, and asks questions grounded in those sources. Answers stream into chat with source/page citations; clicking a citation navigates the PDF viewer to that one-based page. If sources do not support a response, the system abstains.

Visitors can open a one-click, read-only Demo Workspace without signup. The left pane contains workspace/source controls, upload/progress, privacy disclosure, Chat and Curriculum Audit tabs, source-selection checkboxes, streamed Markdown/math/table answers, and a collapsible Retrieval Inspector. The right pane is a PDF.js viewer. On narrow screens panes stack or become tabs.

## MVP features

- Neon Auth accounts and owner-scoped workspaces; read-only shared demo workspace whose seeded rows use the reserved `__demo__` owner sentinel.
- Private Neon S3-compatible object storage and SHA-256 PDF deduplication with reference counts.
- Async PDF ingestion statuses: `pending`, `parsing`, `indexing`, `ready`, `failed`, `deleting`.
- Accept PDFs up to 40 MB / 500 pages, subject to 100 MB / 10-source per-user quotas and project-wide object/database storage circuit breakers. At most 40 pages may require Tesseract OCR. If more are scanned/low-text, fail cleanly and advise chapter-by-chapter upload or a searchable PDF.
- PyMuPDF native extraction and page count; Tesseract OCR for scanned/low-text pages, processed one page at a time with `asyncio.Semaphore(1)`. Persist `embedded` or `ocr` extraction method and page provenance. Display `processed_pages` / `total_pages` progress.
- Hosted embeddings in batches of 64 with bounded exponential backoff/retry; PostgreSQL `pgvector` + `tsvector` hybrid retrieval and RRF.
- Persistent conversations/messages; lightweight follow-up query rewriting uses at most the last three turns and stays within 3,500 context tokens.
- Optional selected-source filtering for Q&A.
- Groq-primary SSE answer generation, Gemini fallback before the first token, deterministic citation validation, evidence-based abstention, and persisted compact retrieval traces.
- Curriculum Coverage & Gap Analyzer: syllabus-tagged sources define topics; notes/PYQ chunks provide semantic + lexical evidence; results are `covered`, `weak`, or `missing`, with validated exact-page citations.
- `OFFLINE_DEMO_MODE` uses pre-cached permitted fixtures with deterministic, visibly labeled responses and no provider calls.

## Free-tier and deployment boundary

Use Vercel Hobby for frontend, Neon Free for Postgres/Auth/object storage, and a Dockerized FastAPI backend on Azure Container Apps while Azure for Students credits are active (user-stated through May 2027). Thereafter redeploy the same stateless image to Render Free or a comparable no-card Docker host; Neon data remains in place. All free quotas, availability, and terms can change. Never auto-upgrade or silently incur charges. Apply explicit user quotas and expose safe quota errors.

## Academic contribution

Evaluate dense, lexical, and RRF retrieval; embedded-text vs OCR pages; and OCR/ingestion cost and latency. Use a versioned corpus of 5–8 permitted PDFs and 30–50 labeled answerable/unanswerable questions as a realistic starting target. Report actual results, limitations, annotation method, and negative findings. Do not fabricate metrics or claim improvement without evidence.

## MVP acceptance

1. Public URL loads; visitor can try read-only demo without account; user can sign in and create an owned workspace.
2. Upload accepts only configured limits and enforces per-user source/storage quotas. Duplicate file content reuses one object safely.
3. Digital textbook up to 500 pages is eligible when within byte/quota limits; Tesseract is invoked for no more than 40 pages. Excess scanned-page files fail with the documented recovery advice.
4. UI reflects ingestion status and page progress; failed/stale jobs can be safely retried.
5. Source viewer access and all workspace records enforce ownership; demo access is read-only except bounded, nonpersistent demo Q&A.
6. Q&A is workspace- and source-filtered, streams exact SSE event contract, persists history for signed-in users, rewrites follow-ups from at most three prior turns, validates citations, and abstains as needed.
7. Curriculum Audit returns persisted topic coverage statuses with source/page citations.
8. Retrieval Inspector exposes documented diagnostics only when requested.
9. Deletion correctly decrements dedup object reference counts and removes an object only after its final source reference is deleted.
10. Offline demo works without network/provider calls; UI/docs disclose cloud object storage, hosted embeddings, and LLM excerpt/question transmission.
11. Verification includes backend tests and frontend build when implementation exists; research results are reproducible and honest.
