# A6-Stern — Architecture

This document covers how the three repos fit together at runtime. For setup instructions see [README.md](README.md).

## System overview

```mermaid
graph TD
    Browser["Browser"]
    Next["frontend-next\nNext.js  :3004"]
    API["backend\nFastAPI  :3001"]
    DB["PostgreSQL\n:5432"]
    MinIO["MinIO\nexternal"]

    Browser -- "HTTP :3004" --> Next
    Next -- "/api/* rewrite\nserver-side" --> API
    API -- "SQLAlchemy" --> DB
    API -- "MinIO SDK" --> MinIO
    Browser -- "presigned URLs\n(direct)" --> MinIO
```



The browser never sees the backend's internal address. All `/api/*` requests go to the Next.js server, which rewrites them server-side to the backend. MinIO is the only service the browser contacts directly for file downloads via presigned URLs.

## Repos and responsibilities


| Repo            | Runtime role                                               | Key config                     |
| --------------- | ---------------------------------------------------------- | ------------------------------ |
| `backend`       | FastAPI app, business logic, DB access, file orchestration | `.env` DB, MinIO, JWT, AI      |
| `frontend-next` | Next.js server + browser bundle, API proxy                 | `BACKEND_URL` build arg        |
| `deployment`    | Docker Compose orchestration                               | `.env` passed to both services |


## API proxy, why BACKEND_URL is a build arg

```mermaid
sequenceDiagram
    participant B as Browser
    participant N as Next.js :3004
    participant A as FastAPI :3001

    B->>N: GET /api/projects
    Note over N: rewrites to http://backend:3001/api/projects
    N->>A: GET /api/projects
    A-->>N: JSON response
    N-->>B: JSON response
```



Next.js bakes rewrite destinations into `routes-manifest.json` at `npm run build` time. This means:

- `BACKEND_URL` must be set **before** `docker compose build`
- The browser bundle contains no reference to internal Docker hostnames
- No CORS configuration is needed between the browser and the backend
- Changing `BACKEND_URL` requires a frontend rebuild

For local dev outside Docker, `BACKEND_URL` defaults to `http://localhost:3002` (the backend container's mapped host port).

## Authentication flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant N as Next.js
    participant A as FastAPI
    participant DB as PostgreSQL

    B->>N: POST /api/auth/login
    N->>A: POST /api/auth/login
    A->>DB: lookup user, verify bcrypt hash
    DB-->>A: user record
    A-->>B: { access_token, user }
    Note over B: stores token in localStorage (key: a6_auth_v2)

    B->>N: GET /api/projects\nAuthorization: Bearer <token>
    N->>A: GET /api/projects\nAuthorization: Bearer <token>
    A->>A: decode JWT, load user from DB
    A-->>B: projects JSON
```



**Token details:**

- Algorithm: HS256
- Lifetime: 7 days
- Storage: `localStorage` cleared on logout or 401 response
- No refresh token expired sessions redirect to `/login`
- First account registered is automatically granted admin rights

## Role system

Two independent layers of access control:

```
Global admin (User.is_admin)
├── Manage all users and projects
├── Upload files to any project
└── Access /api/admin/* routes

Project membership (project_members.role)
├── owner   — manage project settings and members
├── editor  — create annotations and reports
└── viewer  — read-only access
```

Admins bypass project membership checks and have implicit access to all projects.

## Data model

```mermaid
erDiagram
    User {
        string id PK
        string username
        string email
        bool is_admin
        bool is_active
    }
    Project {
        string id PK
        string slug
        string name
        string status
        string floorplan_url
    }
    ProjectMember {
        string project_id FK
        string user_id FK
        string role
    }
    Room {
        string id PK
        string project_id FK
        string slug
        string name
        json floor_plan_coordinates
    }
    FileAsset {
        string id PK
        string room_id FK
        string media_type
        date capture_date
        string bucket_name
        string object_name
        string thumbnail_object_name
        string ai_description
        string sha256_hash
    }
    Annotation {
        string id PK
        string file_id FK
        string annotation_type
        json data
    }
    Report {
        string id PK
        string file_id FK
        string pdf_bucket_name
        string pdf_object_name
        string ai_description
        json flags
    }
    ComparisonDraft {
        string id PK
        string file_id FK
        json state_json
        json flags
    }
    ViewerReportDraft {
        string id PK
        string file_id FK
        string viewer_kind
        json state_json
        json flags
    }

    User ||--o{ ProjectMember : "has"
    Project ||--o{ ProjectMember : "has"
    Project ||--o{ Room : "contains"
    Room ||--o{ FileAsset : "holds"
    FileAsset ||--o{ Annotation : "has"
    FileAsset ||--o{ Report : "has"
    FileAsset ||--o{ ComparisonDraft : "has"
    FileAsset ||--o{ ViewerReportDraft : "has"
```



`media_type` on `FileAsset` is one of: `image`, `video`, `pointcloud`, `pdf`.

## Object storage (MinIO)

MinIO is external to the Docker Compose stack. The backend manages all six buckets and creates them on first startup.

```
MinIO
├── construction-images        ← uploaded images (original)
├── construction-thumbnails    ← auto-generated thumbnails (400×300, quality 82)
├── construction-pointclouds   ← LAZ originals + Potree-converted output (_potree/ prefix)
├── construction-pdfs          ← uploaded PDF documents
├── construction-reports       ← generated report PDFs
└── construction-floorplans    ← project floor plan images
```

The browser accesses files via **presigned URLs** (7-day expiry) returned by the API. The browser PUT-uploads point clouds directly to MinIO using a presigned URL, bypassing the Next.js proxy for large files.

## Point cloud pipeline

```mermaid
sequenceDiagram
    participant B as Browser
    participant A as FastAPI
    participant PC as PotreeConverter
    participant M as MinIO

    B->>A: POST /api/upload/pointcloud/direct-init
    A-->>B: { upload_url (presigned PUT) }
    B->>M: PUT <presigned URL> (LAZ file, direct)
    B->>A: POST /api/upload/pointcloud/direct-complete

    Note over A: fallback if direct init fails:
    Note over B,A: chunked upload (64 MB chunks, 5 concurrent,\n3 retries each) → /api/upload/pointcloud/chunk

    A->>PC: PotreeConverter LAZ → Potree format
    PC-->>A: Potree directory tree
    A->>M: store Potree files under _potree/ prefix
    Note over A,M: optionally delete original LAZ
    A-->>B: FileAsset with status=done
```



PotreeConverter is a pre-built Linux binary installed in the backend Docker image at build time. Two concurrent conversion workers run as a background pool; uploaded files queue if both workers are busy.

## AI image analysis pipeline

```mermaid
sequenceDiagram
    participant B as Browser
    participant A as FastAPI
    participant DB as PostgreSQL
    participant V as Vision API

    B->>A: POST /api/ai/analyze\n{ image_url, file_id }
    A->>DB: check cached ai_description for file_id
    alt cache hit
        DB-->>A: existing description
        A-->>B: 200 { description }
    else cache miss
        A-->>B: 202 (processing)
        A->>V: vision model request (image + prompt)
        V-->>A: text description
        A->>DB: store description on FileAsset
        Note over B: browser polls every 2s\nup to 30 attempts
        B->>A: POST /api/ai/analyze (poll)
        A-->>B: 200 { description }
    end
```



The vision API is OpenAI-compatible. Supports local Ollama models (no key needed) or the Hyperbolic cloud API (`VISION_API_KEY`). The feature degrades gracefully all other functionality is unaffected if no model is reachable.

## Report generation

Reports are generated client-side in the browser using **jsPDF**, then uploaded as finished PDF blobs to the backend. The backend stores them in MinIO and records metadata in PostgreSQL.

Two report types:

- **Viewer report** field observation from any viewer (panorama, point cloud, static image, PDF). Saved as a draft (`ViewerReportDraft`), then published as a `Report`.
- **Comparison report** side-by-side image comparison with annotations. Saved as `ComparisonDraft` entries, consolidated and published together.

## Container networking

All services share the default Docker Compose network (`a6-stern_default` or similar). Internal service names resolve as hostnames:


| Internal hostname | Service    |
| ----------------- | ---------- |
| `db`              | PostgreSQL |
| `backend`         | FastAPI    |
| `frontend-next`   | Next.js    |


The backend hardcodes `DB_HOST=db` in the Compose file. The frontend uses `BACKEND_URL=http://backend:3001` baked at build time. No service communicates with another using host-port mappings — those exist only for external access.