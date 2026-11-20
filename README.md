# PROJ-CS781 — Coursework Knowledge Retrieval

**Intelligent Coursework Knowledge Retrieval and Curriculum Analysis System using Retrieval-Augmented Generation (RAG)** is a MAKAUT B.Tech CSE Project II. It is a live, student-facing web app for uploading coursework PDFs, asking cited questions, and auditing syllabus coverage.

## Product

A split-screen workspace places a source-grounded streaming chat and course controls beside a PDF viewer. Citation clicks jump to the cited page. Students can upload searchable PDFs up to the configured limits; scanned pages use local Tesseract OCR, subject to a separate scanned-page limit. A read-only demo workspace lets visitors try the product without signing up. `OFFLINE_DEMO_MODE` supports viva demos without network/provider calls.

## Architecture and cost

Vercel Hobby hosts the React/Vite frontend. Neon Free provides Postgres (`pgvector` + `tsvector`), S3-compatible object storage, and managed auth from day one. FastAPI runs as a Docker container on Azure Container Apps through the active Azure for Students period, then the same stateless image can move to Render Free or a comparable no-card host without migrating database or file storage. Free quotas and terms can change; the app must not auto-upgrade or silently incur charges.

## Read next

- [Agent rules](AGENTS.md)
- [Single source of truth: API, schema, SSE, auth, configuration](CONTRACTS.md)
- [Project brief](docs/PROJECT_BRIEF.md)
- [Technical stack](docs/TECH_STACK.md)
- [Architecture](docs/ARCHITECTURE.md)
- [Development plan](docs/DEVELOPMENT_PLAN.md)
- [Environment placeholders](.env.example)

This repository is a specification and handoff package; it is not an implemented application yet.
