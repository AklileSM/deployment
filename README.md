# Deployment

This repo contains the shared deployment files for the full stack.

It assumes you have these repos cloned side-by-side:

```text
/opt/a6-stern/
  frontend/
  frontend-next/
  backend/
  deployment/
```

## Contents

- `docker-compose.yml` - runs frontend (legacy SPA), frontend-next (Next.js rebuild), backend, PostgreSQL, and pgAdmin
- `.env.docker` - example environment values for the Docker stack

## Run the stack

```bash
cd deployment
cp .env.docker .env
docker-compose up -d --build
```

## Services & ports

| Service        | Container               | Host port | Internal port | Notes                                       |
| -------------- | ----------------------- | --------- | ------------- | ------------------------------------------- |
| Postgres       | a6_stern_db             | 5433      | 5432          | DB volume `a6_stern_postgres_data`          |
| pgAdmin        | a6_stern_pgadmin        | 5050      | 80            |                                             |
| Backend (FastAPI) | a6_stern_api         | 3002      | 3001          | All API under `/api/`                       |
| Frontend (legacy SPA) | a6_stern_frontend | 3003      | 80            | Nginx; proxies `/api/` to backend           |
| Frontend-Next  | a6_stern_frontend_next  | 3004      | 3000          | Next.js standalone; proxies `/api/` to backend via `BACKEND_URL` baked at build time |

The Compose file builds:
- frontend from `../frontend`
- frontend-next from `../frontend-next`
- backend from `../backend`

## Frontend-Next notes

- Browser only ever talks to `http://<host>:3004` — `/api/*` is rewritten to `backend:3001` by Next.js's server-side rewrite. No CORS needed.
- `BACKEND_URL` is a Docker **build arg** (not just runtime env) because Next.js bakes rewrite destinations into `routes-manifest.json` at build time. Compose passes the right value via `build.args`.
- For local dev outside Docker: `cd frontend-next && PORT=3010 npm run dev` (or any free port). The default `BACKEND_URL=http://localhost:3002` in `.env.local` points at the backend container's host port mapping.

## Legacy volumes

If your real data lives in older volumes named `deployment_postgres_data` / `deployment_pgadmin_data`, layer in the override:

```bash
docker compose -f docker-compose.yml -f docker-compose.legacy-volumes.yml up -d
```
