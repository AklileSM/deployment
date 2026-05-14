# A6-Stern Deployment

This repo wires together the three A6-Stern services using Docker Compose. It does not contain application code, it only contains orchestration config.

## Repository layout

All four repos must be cloned as siblings in the same parent directory:

```
/opt/a6-stern/          ← or any directory you prefer
├── backend/            ← FastAPI + PostgreSQL
├── frontend-next/      ← Next.js app
├── deployment/         ← this repo (run docker compose from here)
└── frontend/           ← legacy SPA (optional, can be omitted)
```

## Prerequisites

- Docker Engine 24+ and Docker Compose v2 (`docker compose`, not `docker-compose`)
- MinIO running and accessible from this host. **MinIO is not managed by this Compose file**
- The `backend/` and `frontend-next/` repos cloned as siblings (see layout above)
- Internet access during the first `docker compose build`. the backend image downloads PotreeConverter from GitHub at build time

## Quick start

```bash
cd deployment
cp .env.docker .env
# Edit .env — at minimum set DB_PASSWORD, MINIO_*, and JWT_SECRET
docker compose up -d --build
```

On first boot the backend automatically:

- Creates all database tables
- Runs any pending schema migrations
- Seeds a default admin user (see backend repo for credentials)

## Environment variables

Copy `.env.docker` to `.env` and fill in the required values.

### Database


| Variable      | Default    | Description                                   |
| ------------- | ---------- | --------------------------------------------- |
| `DB_NAME`     | `a6_stern` | PostgreSQL database name                      |
| `DB_USER`     | `postgres` | PostgreSQL user                               |
| `DB_PASSWORD` | —          | PostgreSQL password. Change from the default. |


### MinIO (object storage)

MinIO is external to this Compose stack. Point these variables at your running MinIO instance.


| Variable                       | Required | Default   | Description                                                                                                                                                                                                                                  |
| ------------------------------ | -------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MINIO_ENDPOINT`               | **Yes**  | —         | MinIO host IP or hostname (no port, no scheme)                                                                                                                                                                                               |
| `MINIO_API_PORT`               | **Yes**  | —         | MinIO S3 API port (commonly `9000` or `9100`)                                                                                                                                                                                                |
| `MINIO_CONSOLE_PORT`           | No       | —         | MinIO web console port (only used for reference)                                                                                                                                                                                             |
| `MINIO_ACCESS_KEY`             | **Yes**  | —         | MinIO access key                                                                                                                                                                                                                             |
| `MINIO_SECRET_KEY`             | **Yes**  | —         | MinIO secret key                                                                                                                                                                                                                             |
| `MINIO_USE_SSL`                | No       | `false`   | Set to `true` if MinIO is behind HTTPS                                                                                                                                                                                                       |
| `MINIO_PUBLIC_UPLOAD_BASE_URL` | No       | *(empty)* | Public-facing base URL for MinIO if it differs from the internal endpoint (e.g. `https://minio.example.com`). Used to construct presigned URLs the browser can reach. Leave empty if the internal endpoint is already reachable by browsers. |


MinIO buckets are created automatically by the backend on first startup.

### Application


| Variable                                      | Required | Default                   | Description                                                                                                                                                                     |
| --------------------------------------------- | -------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `JWT_SECRET`                                  | **Yes**  | `change-me-in-production` | Secret used to sign JWT tokens. Use a long random string in production.                                                                                                         |
| `DEBUG`                                       | No       | `false`                   | Enables FastAPI debug mode and verbose logging                                                                                                                                  |
| `CORS_EXTRA_ORIGINS`                          | No       | *(empty)*                 | Comma-separated list of additional browser origins to allow (e.g. `http://192.168.1.10:3004`). Useful when the UI is accessed via IP rather than the configured `FRONTEND_URL`. |
| `DELETE_ORIGINAL_POINTCLOUD_AFTER_CONVERSION` | No       | `true`                    | After a LAZ/LAS point cloud is converted to Potree format, delete the original from MinIO. Set to `false` to keep originals for re-conversion or archiving (uses more storage). |


### AI vision (optional)

The AI image analysis feature can use either a local Ollama model or the Hyperbolic cloud API.


| Variable             | Required | Default   | Description                                                                                                                                                       |
| -------------------- | -------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `HYPERBOLIC_API_KEY` | No       | *(empty)* | API key for Hyperbolic cloud vision API. If empty and no local model is reachable, AI analysis endpoints will fail gracefully, all other features are unaffected. |


To use a local Ollama model instead of Hyperbolic, set `VISION_API_URL` and `VISION_MODEL` directly in the backend environment inside `docker-compose.yml`. The default local URL is `http://192.168.50.103:11434/v1/chat/completions` with model `qwen3-vl:8b`.

### pgAdmin


| Variable           | Required | Default             | Description                              |
| ------------------ | -------- | ------------------- | ---------------------------------------- |
| `PGADMIN_EMAIL`    | No       | `admin@example.com` | Login email for the pgAdmin web UI       |
| `PGADMIN_PASSWORD` | No       | `admin`             | Login password for pgAdmin. Change this. |


## Services and ports


| Service               | Container                | Host port | Notes                                               |
| --------------------- | ------------------------ | --------- | --------------------------------------------------- |
| PostgreSQL            | `a6_stern_db`            | 5433      | Data volume: `a6_stern_postgres_data`               |
| pgAdmin               | `a6_stern_pgadmin`       | 5050      | Web UI for database inspection                      |
| Backend (FastAPI)     | `a6_stern_api`           | 3002      | All API routes under `/api/`                        |
| Frontend (Next.js)    | `a6_stern_frontend_next` | 3004      | Main application UI                                 |
| Dozzle (log viewer)   | `a6_stern_logs`          | 9999      | Real-time container log viewer                      |
| Frontend (legacy SPA) | `a6_stern_frontend`      | 3003      | Optional; only needed if migrating from the old SPA |


## Architecture notes

### API proxy, no CORS needed

The Next.js container never exposes the backend URL to the browser. All `/api/`* requests from the browser go to `http://<host>:3004/api/`*, and Next.js rewrites them server-side to `http://backend:3001`. This means:

- No CORS configuration is needed between the browser and the backend
- `BACKEND_URL` is a Docker **build arg**, not just a runtime env var, Next.js bakes the rewrite destination into its route manifest at build time. If you change `BACKEND_URL`, you must rebuild the frontend image.

### Point cloud pipeline

When a LAZ/LAS file is uploaded:

1. It is uploaded in 32 MB chunks to the backend
2. The backend assembles the chunks and passes the file to PotreeConverter
3. PotreeConverter is a pre-built Linux binary downloaded automatically during `docker build`
4. The converted Potree files are stored in MinIO; the original LAZ can be kept or deleted (see `DELETE_ORIGINAL_POINTCLOUD_AFTER_CONVERSION`)

### Database migrations

The backend runs lightweight schema migrations at startup (adding columns, creating tables). These are safe to run on every restart and do not destroy data.

## Data persistence


| Data                                                    | Where it lives                           |
| ------------------------------------------------------- | ---------------------------------------- |
| PostgreSQL data                                         | Docker volume `a6_stern_postgres_data`   |
| pgAdmin config                                          | Docker volume `a6_stern_pgadmin_data`    |
| All uploaded files (images, videos, PDFs, point clouds) | External MinIO, back up MinIO separately |


### PostgreSQL backup and restore

```bash
# Backup — produces a plain-SQL dump you can store anywhere
docker exec a6_stern_db pg_dump -U postgres a6_stern > backup_$(date +%Y%m%d).sql

# Restore — into a running (empty) database
docker exec -i a6_stern_db psql -U postgres a6_stern < backup_20260401.sql
```

For automated nightly backups, run the `pg_dump` command from a cron job on the host and copy the output to off-site storage.

### MinIO backup and restore

MinIO is external to this Compose stack. Use the [MinIO Client (`mc`)](https://min.io/docs/minio/linux/reference/minio-mc.html) to back up and restore bucket contents.

**Install mc on the host:**

```bash
# Linux
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc && chmod +x mc && mv mc /usr/local/bin/
```

**Register your MinIO instance:**

```bash
mc alias set a6minio http://<MINIO_ENDPOINT>:<MINIO_API_PORT> <MINIO_ACCESS_KEY> <MINIO_SECRET_KEY>
```

**Back up all buckets to a local directory:**

```bash
mc mirror a6minio/ ./minio-backup-$(date +%Y%m%d)/
```

`mc mirror` is incremental — re-running it only copies new or changed objects.

**Restore from backup:**

```bash
# Restore a specific bucket
mc mirror ./minio-backup-20260401/construction-images/ a6minio/construction-images/

# Restore all buckets
mc mirror ./minio-backup-20260401/ a6minio/
```

**Sync to off-site storage (e.g. S3-compatible remote):**

```bash
mc alias set offsite s3.amazonaws.com ACCESS SECRET
mc mirror a6minio/ offsite/a6-stern-backup/
```

> Point cloud files are typically the largest; the `construction-pointclouds` bucket holds both raw LAZ originals (if `DELETE_ORIGINAL_POINTCLOUD_AFTER_CONVERSION=false`) and the converted Potree octree files. Plan storage accordingly.

## Common operations

**Rebuild after a code change:**

```bash
docker compose up -d --build backend          # rebuild only the backend
docker compose up -d --build frontend-next    # rebuild only the frontend
docker compose up -d --build                  # rebuild everything
```

**View logs:**

```bash
docker compose logs -f backend
docker compose logs -f frontend-next
# or open Dozzle at http://<host>:9999
```

**Stop the stack:**

```bash
docker compose down          # stop containers, keep volumes
```

## Legacy volume migration

If you previously ran this stack and your data is in volumes named `deployment_postgres_data` / `deployment_pgadmin_data` (from an older Compose project name), use the override file to reattach them:

```bash
docker compose -f docker-compose.yml -f docker-compose.legacy-volumes.yml up -d
```

Omit the override file on fresh installs the default volume names are `a6_stern_postgres_data` and `a6_stern_pgadmin_data`.

## Upgrade and rollback

### Upgrading a service

Pull the latest code for the relevant repo, then rebuild only that service:

```bash
cd ../backend && git pull
cd ../deployment
docker compose up -d --build backend
```

The backend runs schema migrations automatically on startup, new columns are added without data loss. You do not need to run anything manually.

For the frontend:

```bash
cd ../frontend-next && git pull
cd ../deployment
docker compose up -d --build frontend-next
```

> **Remember:** `BACKEND_URL` is baked into the Next.js image at build time. If you change that variable, you must rebuild.

### Rolling back a service

Tag images before upgrading so you have a known-good reference:

```bash
# Before upgrading — save the current image under a dated tag
docker tag a6_stern_api a6_stern_api:backup-$(date +%Y%m%d)
docker tag a6_stern_frontend_next a6_stern_frontend_next:backup-$(date +%Y%m%d)
```

To roll back a service to the saved tag:

```bash
# Stop and remove the current container
docker compose stop backend
docker compose rm -f backend

# Re-tag the backup as latest and restart
docker tag a6_stern_api:backup-20260401 a6_stern_api:latest
docker compose up -d backend
```

### Schema migration rollback

There is no automated rollback for database migrations, the migrations are additive-only (new nullable columns, new tables) so downgrading application code is safe without reversing the schema. Old code simply ignores new columns it does not know about.

If a migration caused a problem you need to reverse manually, connect to the database and `ALTER TABLE … DROP COLUMN …` for the column(s) added by that migration. See `backend/app/services/db_migrations.py` for the exact column names added by each function.

### Changing BACKEND_URL mid-deployment

`BACKEND_URL` is baked into the Next.js image at build time (it controls the server-side rewrite target). Changing it in `.env` alone has no effect. Steps:

1. Update `BACKEND_URL` in `.env`
2. Rebuild: `docker compose up -d --build frontend-next`
3. Verify: `docker inspect a6_stern_frontend_next | grep BACKEND_URL`

## Troubleshooting

### Frontend shows "Failed to fetch" or API calls return 502

The Next.js container cannot reach the backend. Check:

1. Is the backend container running? `docker compose ps` look for `a6_stern_api` with status `Up`.
2. Was the frontend image built with the right `BACKEND_URL`? The URL is baked at build time. Run `docker inspect a6_stern_frontend_next | grep BACKEND_URL` to verify. If wrong, rebuild: `docker compose up -d --build frontend-next`.
3. Both containers must be on the same Docker network. If you started them separately rather than with a single `docker compose up`, they may be on different networks.

### Backend fails to start, database connection error

Check that the `db` container is healthy before the backend starts:

```bash
docker compose logs db
docker compose logs backend
```

If the DB is still initialising when the backend starts, restart the backend: `docker compose restart backend`. The backend does not retry the initial connection.

### Point cloud stuck in "converting" forever

The conversion runs in a background process inside the backend container. Check the backend logs:

```bash
docker compose logs backend | grep -i potree
docker compose logs backend | grep -i conversion
```

Or watch in real time via Dozzle at `http://<host>:9999`.

Common causes:

- **PotreeConverter not found:** should not happen in Docker (installed at build time), but check the log for `PotreeConverter binary not found`.
- **Conversion timed out**: the 10-minute timeout was exceeded. Large files (>1 GB) may need the conversion pool adjusted.
- **Server restarted mid-conversion**: on the next startup the backend resets interrupted jobs to `failed`. Re-upload the file to retry.

### MinIO buckets not created / files not uploading

The backend creates the six MinIO buckets on startup. If it cannot reach MinIO, uploads will fail silently at the bucket-creation step.

1. Check `MINIO_ENDPOINT`, `MINIO_API_PORT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` in your `.env`.
2. Verify MinIO is reachable from the backend container: `docker exec a6_stern_api curl -s http://${MINIO_ENDPOINT}:${MINIO_API_PORT}/minio/health/live`
3. If `MINIO_USE_SSL=true`, ensure the MinIO certificate is trusted inside the container.

### Direct point cloud upload fails, falls back to chunked

Direct upload (browser → MinIO presigned URL) requires `MINIO_PUBLIC_UPLOAD_BASE_URL` to be set to a URL the **browser** can reach. If the variable is empty or points to an internal Docker hostname, the backend returns 400 on the direct-init endpoint and the frontend automatically falls back to chunked upload through the proxy. The chunked fallback is fully functional direct upload is an optimisation only.

### pgAdmin cannot connect to the database

In pgAdmin's server configuration, use `db` as the hostname (the Docker service name), not `localhost`. Port is `5432` (the internal port, not `5433`).

### Logs show "Reset N interrupted pointcloud conversion(s) to failed"

This is normal after a server restart and means conversions that were in-progress when the server stopped have been marked as failed. Re-upload any affected files to re-trigger conversion.

## Local development (without Docker)

To run services locally without Docker, see the README in each repo:

- `backend/` FastAPI with uvicorn, PostgreSQL and MinIO pointed at local/remote instances
- `frontend-next/`, `npm run dev`, with `BACKEND_URL` pointing at the running backend

