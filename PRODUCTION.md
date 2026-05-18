# Production Hardening Checklist

The default `docker compose up -d` setup is suitable for an internal lab network. Before exposing the stack to the public internet (or to a larger team), work through this list.

## 0. Pre-flight

- Read `ARCHITECTURE.md` to understand the service topology, the Next.js API proxy, and the `BACKEND_URL` build-arg gotcha.
- Have a dedicated Linux host with at least the [recommended sizing](#sizing). Don't co-host with unrelated services.
- Have a domain or stable IP you can point users at. The browser must reach it; emails will contain links generated from `FRONTEND_URL`.

## 1. Reverse proxy + TLS

Terminate TLS at a reverse proxy in front of the Next.js container. Two common setups:

### Option A: Caddy (easiest, auto-TLS)

```caddy
# /etc/caddy/Caddyfile
sitescope.example.com {
    encode zstd gzip
    reverse_proxy localhost:3004 {
        # Increase if you raise NEXT_PROXY_MAX_BODY in frontend-next
        max_body_size 256MB
    }
}

# Optional: expose MinIO directly for browser-side presigned URLs and
# direct point-cloud uploads. Required if MINIO_PUBLIC_UPLOAD_BASE_URL is set.
minio.example.com {
    reverse_proxy localhost:9100
}
```

`systemctl reload caddy` and Caddy will provision a Let's Encrypt cert automatically.

### Option B: nginx + certbot

```nginx
server {
    listen 443 ssl http2;
    server_name sitescope.example.com;

    ssl_certificate     /etc/letsencrypt/live/sitescope.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sitescope.example.com/privkey.pem;

    client_max_body_size 256M;   # ≥ NEXT_PROXY_MAX_BODY
    proxy_request_buffering off; # streaming uploads
    proxy_read_timeout 600s;     # large point-cloud assemble + convert

    location / {
        proxy_pass http://localhost:3004;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name sitescope.example.com;
    return 301 https://$host$request_uri;
}
```

Issue cert with `certbot --nginx -d sitescope.example.com`.

### Mandatory after enabling TLS

- Set `FRONTEND_URL=https://sitescope.example.com` in `.env`.
- **Rebuild the frontend**: `BACKEND_URL` is baked at build time; if you're flipping the public URL, also rebuild: `docker compose up -d --build frontend-next`.
- Update `CORS_EXTRA_ORIGINS` if you have additional approved origins (e.g., IP-based admin access on the LAN).
- If MinIO is reachable via TLS too, set `MINIO_USE_SSL=true` and update `MINIO_PUBLIC_UPLOAD_BASE_URL`.

## 2. Secrets management

The default workflow puts secrets in `.env`. That's fine for a lab, not for production.

### Minimum: file permissions

```bash
chmod 600 deployment/.env
chown root:docker deployment/.env  # whatever group runs docker
```

Verify no group/world read. `.env` should never be committed to git, confirm it's in `.gitignore`.

### Recommended: Docker secrets

For sensitive values (`JWT_SECRET`, `SMTP_PASSWORD`, `DB_PASSWORD`, `MINIO_SECRET_KEY`), use Docker secrets so they never appear in `docker inspect` or process listings:

```yaml
# docker-compose.override.yml
services:
  backend:
    secrets:
      - jwt_secret
      - smtp_password
      - minio_secret_key
    environment:
      JWT_SECRET_FILE: /run/secrets/jwt_secret
      SMTP_PASSWORD_FILE: /run/secrets/smtp_password
      MINIO_SECRET_KEY_FILE: /run/secrets/minio_secret_key
secrets:
  jwt_secret:        { file: ./secrets/jwt_secret }
  smtp_password:     { file: ./secrets/smtp_password }
  minio_secret_key:  { file: ./secrets/minio_secret_key }
```

> The backend does not currently read `*_FILE` variants automatically, you'd need to add a small wrapper that reads the file before launching uvicorn, or shell-expand them in the container entrypoint. Until that's wired up, use plain env vars with strict `.env` permissions.

### Recommended: SOPS for git-tracked encrypted secrets

[Mozilla SOPS](https://github.com/getsops/sops) with `age` keys lets you commit an encrypted `.env.sops` and decrypt on the deploy host:

```bash
sops --decrypt deployment/.env.sops > deployment/.env
docker compose up -d
```

The age private key lives only on the deploy host; the public key is checked into the repo.

## 3. Generate strong secrets

```bash
# JWT signing key (32 bytes of base64)
openssl rand -base64 32

# MinIO root credentials (40+ chars)
openssl rand -base64 30
openssl rand -base64 40

# Postgres password
openssl rand -base64 24

# pgAdmin password
openssl rand -base64 20
```

Rotate `JWT_SECRET` if you suspect compromise, this invalidates every existing JWT and forces users to log back in. No data loss.

## 4. Firewall

Only these ports should be open to the public:


| Port                      | Service                                                               |
| ------------------------- | --------------------------------------------------------------------- |
| `80`                      | redirect to 443                                                       |
| `443`                     | Caddy / nginx → Next.js                                               |
| `443` (separate hostname) | Caddy / nginx → MinIO (only if `MINIO_PUBLIC_UPLOAD_BASE_URL` is set) |


Everything else stays bound to localhost or to a private interface:


| Port   | Service   | Bind to                                           |
| ------ | --------- | ------------------------------------------------- |
| `3004` | Next.js   | `127.0.0.1:3004`                                  |
| `3002` | Backend   | `127.0.0.1:3002`                                  |
| `5433` | Postgres  | `127.0.0.1:5433` (or remove the mapping entirely) |
| `5050` | pgAdmin   | LAN/VPN only, **never public**                    |
| `9999` | Dozzle    | LAN/VPN only                                      |
| `9100` | MinIO API | bound to the host's reverse-proxy interface only  |


In `docker-compose.override.yml` for production:

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5433:5432"     # localhost only
  pgadmin:
    ports: []                     # don't expose at all in prod (or via SSH tunnel)
  dozzle:
    ports: []                     # same
  backend:
    ports:
      - "127.0.0.1:3002:3001"
  frontend-next:
    ports:
      - "127.0.0.1:3004:3000"
```

Or, on the host: `ufw allow 80,443/tcp` and deny everything else.

### SSH tunnel for ops

For pgAdmin / Dozzle / direct DB access in production:

```bash
ssh -L 5050:localhost:5050 -L 9999:localhost:9999 -L 5432:localhost:5433 deploy@host
```

Then open `http://localhost:5050` (pgAdmin), `http://localhost:9999` (Dozzle), and use `psql -h localhost -p 5432` locally.

## 5. Logging

### Container logs

Docker's default `json-file` driver fills the disk over time. Cap it in `docker-compose.override.yml`:

```yaml
services:
  backend:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "10"
  frontend-next:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"
  db:
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "10"
```

Or ship to a central system (`syslog`, `gelf`, `journald`, `loki`).

### Application logs

The backend logs to stdout, Docker captures it. Useful filters:

```bash
docker compose logs -f backend | grep -i "error\|exception"
docker compose logs -f backend | grep -i "smtp\|email"           # email send results
docker compose logs -f backend | grep -i "conversion\|potree"    # pointcloud worker
```

Dozzle at `http://<host>:9999` gives the same view through a UI. Don't expose Dozzle publicly, it shows everything.

## 6. Database

### Backups (see also `DISASTER_RECOVERY.md`)

Set up an off-host backup. The README's `pg_dump` command in a cron entry is the minimum:

```cron
# /etc/cron.d/a6stern-backup
0 3 * * *  root  /usr/local/bin/a6stern-backup.sh
```

```bash
#!/bin/bash
# /usr/local/bin/a6stern-backup.sh
set -euo pipefail
DEST=/var/backups/a6stern
DATE=$(date +%Y-%m-%d)
mkdir -p "$DEST"
docker exec a6_stern_db pg_dump -U postgres -Fc a6_stern > "$DEST/db-$DATE.dump"
# Rotate, keep 30 days
find "$DEST" -name 'db-*.dump' -mtime +30 -delete
# Off-site (rsync to backup host, or aws s3 cp, or restic, pick one)
rsync -a "$DEST/db-$DATE.dump" backup-host:/srv/a6stern/
```

### Tuning

The Postgres defaults are conservative. For the typical 4–16 GB host:

```sql
ALTER SYSTEM SET shared_buffers       = '1GB';
ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM SET work_mem             = '32MB';
ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM SET random_page_cost     = '1.1';   -- if backed by SSD
SELECT pg_reload_conf();
```

Anything more ambitious, read [pgtune.leopard.in.ua](https://pgtune.leopard.in.ua/) and apply per-host.

## 7. MinIO

MinIO is external to this Compose stack, that's intentional, so you can size storage and IO independently from the app. In production:

- Run MinIO on storage-class disks (not a USB drive).
- Enable versioning on the buckets that matter (`construction-images`, `construction-pointclouds`, `construction-reports`), gives you object-level undo.
- Use distinct access keys per environment (dev/staging/prod).
- Configure off-host replication via `mc mirror` or MinIO server-side replication.
- Lock down the MinIO console (port 9101 by default) to LAN/VPN.

## 8. AI / vision

`POST /api/ai/analyze` is **unauthenticated** by design (see `backend/AI.md`). On the public internet this means:

- Anyone with the URL can trigger vision-model calls.
- If you pay per-token, this is a money-drain attack vector.

Mitigations:

- Front the route with rate-limiting at the reverse proxy:
  ```nginx
  limit_req_zone $binary_remote_addr zone=ai:10m rate=5r/m;
  location /api/ai/ {
      limit_req zone=ai burst=10 nodelay;
      proxy_pass http://localhost:3004;
  }
  ```
- Or patch `app/api/ai.py` to add `Depends(get_current_user)`. Update `AI.md` if you do.

## 9. Frontend hardening

- Set `DEBUG=false` (default) so FastAPI hides traceback details.
- Inspect `docker inspect a6_stern_frontend_next | grep BACKEND_URL` to confirm the baked URL matches what you intended.
- Confirm `NEXT_PUBLIC_API_URL` is **empty** (the default). If it's set to a backend URL, the browser bundle will reference it directly and you've lost the proxy benefit.
- Remove the legacy `frontend` container if you're not actively migrating from the old SPA. Edit `docker-compose.yml` and drop the `frontend:` service.

## 10. Updates

Establish a rhythm:

```bash
# Weekly
cd ~/a6-stern/backend       && git pull
cd ~/a6-stern/frontend-next && git pull
cd ~/a6-stern/deployment    && git pull && docker compose pull
docker compose up -d --build
```

Tag images before rebuilding so you can roll back (see README "Upgrade and rollback").

## Sizing


| Resource           | Minimum                                                                              | Recommended                                                 |
| ------------------ | ------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| CPU                | 2 vCPU                                                                               | 4–8 vCPU (more if many pointcloud conversions)              |
| RAM                | 4 GB                                                                                 | 8–16 GB                                                     |
| Disk (system + DB) | 20 GB                                                                                | 100 GB SSD                                                  |
| Disk (MinIO)       | depends entirely on capture volume, point clouds dominate, plan for 1–5 GB per scan |                                                             |
| Network            | 100 Mbit/s                                                                           | 1 Gbit/s, especially if uploading from LAN-attached devices |


A typical lab deployment (one A6-Stern project, ~10 users, daily scans) runs comfortably on 4 vCPU / 8 GB / 200 GB MinIO.

## Final smoke test

After everything's in place:

```bash
curl https://sitescope.example.com/api/health
# → {"status":"ok","app":"A6 Stern","environment":"production","storage":true}

# Register a fresh admin (the first registered user wins admin rights)
curl -X POST https://sitescope.example.com/api/auth/register \
     -H 'Content-Type: application/json' \
     -d '{"username":"prod_admin","password":"<strong>","email":"you@example.com"}'

# Check email arrived (or read the verification token from the DB)
docker exec a6_stern_db psql -U postgres a6_stern \
  -c "SELECT email_verification_token FROM users WHERE username='prod_admin';"
```

If all three succeed, the deployment is healthy.