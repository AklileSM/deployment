# SiteScope: Project Overview

**Start here.** This document is the cross-repo front page: what SiteScope is, how the three repos fit together, and where to read about every feature.

## What it is

SiteScope is a browser-based **construction site documentation platform**. Users upload photos, 360° panoramas, videos, PDFs, and 3D LiDAR point clouds captured on site; the platform organises them by project / room / date, generates AI-assisted analyses, and lets engineers publish structured PDF field reports.

A6-Stern is a single construction project being monitored on the platform. The platform itself supports multiple projects, multiple users, role-based access control, and side-by-side temporal comparison of any two captures.

## The three repos

The codebase is split across **three sibling repos** that are cloned next to each other:

```
/opt/SiteScope/             ← parent directory (any name is fine)
├── backend/               ← FastAPI + PostgreSQL + MinIO orchestration
├── frontend-next/         ← Next.js 16 (App Router) + React 19
└── deployment/            ← Docker Compose orchestration (you are here)
```


| Repo              | Role                                                                                          | Language              |
| ----------------- | --------------------------------------------------------------------------------------------- | --------------------- |
| **backend**       | Business logic, DB access, file storage, AI calls, PDF metadata, auth. All `/api/`* routes.   | Python 3.11 / FastAPI |
| **frontend-next** | UI, viewers (Three.js, Potree, PDF.js), client-side PDF generation (jsPDF).                   | TypeScript / Next.js  |
| **deployment**    | Docker Compose, env vars, production hardening, backup/restore runbooks. No application code. | YAML                  |


PostgreSQL runs in a container managed by the deployment repo. **MinIO is external** to the Compose stack, point the backend at any MinIO-compatible S3 endpoint.

## How it fits together at runtime

```
Browser ──►HTTP──► Next.js (:3004)──►server-side rewrite──► FastAPI (:3001) ──► PostgreSQL
                                                              │
                                                              └──► MinIO (external)
```

The browser never sees the backend's internal address. All `/api/*` requests go to Next.js, which rewrites them server-side. This is the reason `BACKEND_URL` is a **build-time** arg in the frontend image, see `ARCHITECTURE.md` for the why.

Full topology, networking, and the API-proxy rationale are in [ARCHITECTURE.md](ARCHITECTURE.md).

## Documentation map (by topic)

Every feature is documented in depth in the repo that owns the relevant code. Use the topic table below as the index, pick a topic, then read the per-repo docs in any order.


| Topic                                                         | Backend                                                           | Frontend                                                                                                                           | Diagrams / Other                                                                                                                                                                |
| ------------------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Authentication & email**                                    | `[backend/AUTH_AND_EMAIL.md](../backend/AUTH_AND_EMAIL.md)`       | `[frontend-next/AUTH_FLOWS.md](../frontend-next/AUTH_FLOWS.md)`                                                                    | BPMN `[02-join-platform](../docs/bpmn/02-join-platform.bpmn)`                                                                                                                   |
| **Permissions matrix**                                        | `[backend/PERMISSIONS.md](../backend/PERMISSIONS.md)`             | —                                                                                                                                  | —                                                                                                                                                                               |
| **Projects, rooms, members**                                  | `[backend/PROJECTS.md](../backend/PROJECTS.md)`                   | `[frontend-next/PROJECT_SETTINGS.md](../frontend-next/PROJECT_SETTINGS.md)`, `[frontend-next/ADMIN.md](../frontend-next/ADMIN.md)` | BPMN `[03-project-setup](../docs/bpmn/03-project-setup.bpmn)`                                                                                                                   |
| **File explorer & uploads**                                   | `[backend/FILES.md](../backend/FILES.md)`                         | `[frontend-next/EXPLORER.md](../frontend-next/EXPLORER.md)`                                                                        | BPMN `[04-upload](../docs/bpmn/04-upload.bpmn)`, `[05-pointcloud-conversion](../docs/bpmn/05-pointcloud-conversion.bpmn)`, `[06-browse-view](../docs/bpmn/06-browse-view.bpmn)` |
| **AI vision analysis**                                        | `[backend/AI.md](../backend/AI.md)`                               | (UI in `components/viewers/StaticViewer.tsx`)                                                                                      | BPMN `[07-ai-analysis](../docs/bpmn/07-ai-analysis.bpmn)`                                                                                                                       |
| **Annotations**                                               | `[backend/ANNOTATIONS.md](../backend/ANNOTATIONS.md)`             | `[frontend-next/ANNOTATIONS.md](../frontend-next/ANNOTATIONS.md)`                                                                  | BPMN `[08-annotations](../docs/bpmn/08-annotations.bpmn)`                                                                                                                       |
| **Reports & drafts (PDF)**                                    | `[backend/REPORTS.md](../backend/REPORTS.md)`                     | `[frontend-next/REPORTS.md](../frontend-next/REPORTS.md)`                                                                          | BPMN `[09-field-report](../docs/bpmn/09-field-report.bpmn)`                                                                                                                     |
| **Viewers (panorama / point cloud / static / PDF / compare)** | —                                                                 | `[frontend-next/VIEWERS.md](../frontend-next/VIEWERS.md)`                                                                          | —                                                                                                                                                                               |
| **UI styling tokens & conventions**                           | —                                                                 | `[frontend-next/STYLING.md](../frontend-next/STYLING.md)`                                                                          | —                                                                                                                                                                               |
| **API client (how to add an endpoint)**                       | —                                                                 | `[frontend-next/API_CLIENT.md](../frontend-next/API_CLIENT.md)`                                                                    | —                                                                                                                                                                               |
| **Schema migrations**                                         | `[backend/migrations/README.md](../backend/migrations/README.md)` | —                                                                                                                                  | —                                                                                                                                                                               |
| **Testing**                                                   | (see backend `README.md` § Testing)                               | `[frontend-next/TESTING.md](../frontend-next/TESTING.md)`                                                                          | —                                                                                                                                                                               |


## Operations docs (this repo)


| Doc                                            | Covers                                                              |
| ---------------------------------------------- | ------------------------------------------------------------------- |
| `[README.md](README.md)`                       | Quick start, env vars, services, common operations, troubleshooting |
| `[ARCHITECTURE.md](ARCHITECTURE.md)`           | System topology, API proxy, networking, data model                  |
| `[PRODUCTION.md](PRODUCTION.md)`               | Hardening checklist, TLS, secrets, firewall, logging, sizing        |
| `[DISASTER_RECOVERY.md](DISASTER_RECOVERY.md)` | Backup/restore runbook for Postgres + MinIO; fresh-host recovery    |
| `[NEXT_STEPS.md](NEXT_STEPS.md)`               | Roadmap notes (Unitree GO2 robot integration)                       |


## Setting up an environment

For a first-time bring-up, follow `[README.md](README.md)` § "Quick start", clone the three repos as siblings, copy `.env.docker` to `.env`, fill in the required values, then `docker compose up -d --build`.

For production deployment, also work through `[PRODUCTION.md](PRODUCTION.md)` (TLS, secrets, firewall) and `[DISASTER_RECOVERY.md](DISASTER_RECOVERY.md)` (so you have a tested restore path before you need one).

## Key things to know before reading the code

A short list of non-obvious decisions that surface across the codebase:

- **First registered user becomes admin.** No hardcoded credentials. See `backend/AUTH_AND_EMAIL.md`.
- **Role is read from the DB, not the JWT.** Role changes take effect immediately, no re-login required. See `backend/PERMISSIONS.md`.
- **Reports are creator-scoped.** Even admins cannot read another user's reports or drafts. See `backend/REPORTS.md`.
- **PDFs are generated client-side** with jsPDF and uploaded as finished blobs; the backend never renders PDFs. See `frontend-next/REPORTS.md`.
- `**BACKEND_URL` is a build-time arg.** Changing it requires rebuilding the frontend image. See `ARCHITECTURE.md`.
- **MinIO is external** to the Compose stack. See `README.md` § MinIO.
- **Migrations are code-driven**, additive only, run at backend startup. See `backend/migrations/README.md`.

