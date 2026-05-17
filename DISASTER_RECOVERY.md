# Disaster Recovery

This is the runbook for two scenarios:

1. **Routine restore** — bring back a specific database snapshot or restore some MinIO objects.
2. **Total host loss** — set up a brand-new machine from your backups.

The Compose stack itself is reproducible from git. The data you actually need to back up is:

- **PostgreSQL** — all metadata (users, projects, rooms, file rows, reports, drafts, annotations).
- **MinIO** — the file bytes themselves (images, pointclouds, PDFs, report PDFs, floorplans, annotation attachments).

Neither can recreate the other. **You need both backups.**

## What does NOT need to be backed up

- The Docker images — rebuilt from `git pull` + `docker compose build`.
- The `.env` file — keep a separate, encrypted copy (see `PRODUCTION.md`). It's not "data".
- pgAdmin's saved server configs — convenience only; recreate by hand.
- Dozzle — stateless.

## Backup strategy

### Recommended frequency

| Asset | Frequency | Retention |
|---|---|---|
| Postgres `pg_dump` | nightly | 30 days local + 90 days off-site |
| MinIO `mc mirror` | nightly (incremental) | 30 days local + 90 days off-site |
| `.env` | on every change | versioned in encrypted form (SOPS) |
| Compose files | git | indefinite |

Anything less than nightly leaves you with > 24 h of potential data loss. Going more frequent (every 6 h) requires WAL archiving on Postgres and is out of scope here.

### Postgres backup

Already covered in `README.md`. The cron pattern from `PRODUCTION.md`:

```bash
#!/bin/bash
set -euo pipefail
DEST=/var/backups/a6stern
DATE=$(date +%Y-%m-%d)
mkdir -p "$DEST"
docker exec a6_stern_db pg_dump -U postgres -Fc a6_stern > "$DEST/db-$DATE.dump"
find "$DEST" -name 'db-*.dump' -mtime +30 -delete
rsync -a "$DEST/db-$DATE.dump" backup-host:/srv/a6stern/
```

The `-Fc` flag produces a custom-format dump (compressed, allows partial restore). If you prefer plain SQL for easy inspection, use `-F p` instead.

### MinIO backup

`mc mirror` is incremental — re-running it only copies new or changed objects.

```bash
#!/bin/bash
# /usr/local/bin/a6stern-minio-backup.sh
set -euo pipefail
ALIAS=a6minio
DEST=/var/backups/a6stern-minio
mkdir -p "$DEST"
mc mirror --remove "$ALIAS/" "$DEST/"
rsync -a --delete "$DEST/" backup-host:/srv/a6stern-minio/
```

The `--remove` flag on `mc mirror` deletes objects from the backup that no longer exist in MinIO. **Skip it if you want a soft-delete safety net** (orphaned objects accumulate but never disappear).

### Off-site

The local backup is one disk failure away from useless. Always replicate off-host. Cheap options:

- A second internal server: `rsync -a /var/backups/a6stern/ backup-host:/srv/a6stern/`
- An S3-compatible bucket: `aws s3 sync /var/backups/a6stern/ s3://your-backup-bucket/a6stern/`
- Restic + B2 / Wasabi: `restic -r b2:bucket:path backup /var/backups`

## Restore drills

The whole point of backups is to test the restore. Do this in a **non-prod environment** at least once per quarter. Steps below.

## Scenario A — Restore a Postgres snapshot

The current DB has corrupt data (or someone hard-deleted a project that should still exist).

```bash
# 1. Stop dependent services so they don't write while you're restoring.
docker compose stop backend frontend-next

# 2. Drop the existing DB and recreate empty.
docker exec a6_stern_db psql -U postgres -c "DROP DATABASE a6_stern;"
docker exec a6_stern_db psql -U postgres -c "CREATE DATABASE a6_stern OWNER postgres;"

# 3. Restore.
docker exec -i a6_stern_db pg_restore -U postgres -d a6_stern --no-owner < db-2026-05-15.dump

# 4. Restart.
docker compose up -d backend frontend-next
```

> If your dump is plain SQL (`pg_dump > ...sql`), use `psql ... < ...sql` instead of `pg_restore`.

**Verify** by hitting `/api/health` and logging in as a known user. If the file rows reference MinIO objects that don't exist anymore, you'll see 404s on the file grid — that's the cue to also restore MinIO for the affected date range.

## Scenario B — Restore some MinIO objects

A user deleted files they shouldn't have.

```bash
# Restore one bucket from the most recent backup.
mc mirror /var/backups/a6stern-minio/construction-images/ a6minio/construction-images/

# Or one specific object.
mc cp /var/backups/a6stern-minio/construction-images/<roomId>/2026-04-01/<file> \
      a6minio/construction-images/<roomId>/2026-04-01/<file>
```

The DB row will already exist if you restored from a snapshot **after** the file was uploaded. If the DB row was also deleted, restore Postgres (Scenario A) to a point in time where the row existed.

## Scenario C — Total host loss (fresh-host recovery)

Disk failure, fire, ransomware. You have only the off-site backups and the git repos.

### Pre-requisites

- A fresh Linux host with Docker Engine 24+ and Docker Compose v2 installed.
- SSH access from the host you'll restore *from* (or `aws cli`/`restic`/whatever you used for off-site).
- The encrypted `.env` (or know the SOPS / age key needed to decrypt it).
- A reachable MinIO instance — see step 3.

### Step 1. Clone the repos

```bash
mkdir -p /opt/a6-stern && cd /opt/a6-stern
git clone <backend-repo>       backend
git clone <frontend-next-repo> frontend-next
git clone <deployment-repo>    deployment
```

Match the structure documented in `README.md` ("Repository layout"). The legacy `frontend/` repo is optional.

### Step 2. Restore `.env`

```bash
cd deployment
# From an encrypted-in-git copy:
sops --decrypt .env.sops > .env
chmod 600 .env

# Or from off-site:
scp backup-host:/srv/a6stern-env/.env-prod ./.env
chmod 600 .env
```

Verify the password / secret fields are non-empty.

### Step 3. Restore MinIO

**Option A** — MinIO is back up at the same address. Restore objects:

```bash
mc alias set a6minio http://<MINIO_ENDPOINT>:<MINIO_API_PORT> <ACCESS> <SECRET>

# Pull the off-site backup down first (or skip if it's already local).
rsync -a backup-host:/srv/a6stern-minio/ /var/backups/a6stern-minio/

mc mirror /var/backups/a6stern-minio/ a6minio/
```

**Option B** — MinIO is gone too. Stand up a fresh MinIO instance (Docker, `minio server`, or whatever), point the `MINIO_*` env vars at it in `.env`, then restore as above. The backend will create the buckets on first start if they don't exist, but you can pre-create them with `mc mb` to avoid race conditions:

```bash
for b in construction-images construction-thumbnails construction-pointclouds \
         construction-pdfs construction-reports construction-floorplans \
         construction-annotation-attachments; do
  mc mb --ignore-existing a6minio/$b
done
```

### Step 4. Bring up Postgres + restore

```bash
cd /opt/a6-stern/deployment
docker compose up -d db pgadmin
# Wait ~10s for Postgres to initialise.
docker compose ps   # confirm db is "Up"

# Restore the latest dump.
docker exec a6_stern_db psql -U postgres -c "DROP DATABASE IF EXISTS a6_stern;"
docker exec a6_stern_db psql -U postgres -c "CREATE DATABASE a6_stern OWNER postgres;"
docker exec -i a6_stern_db pg_restore -U postgres -d a6_stern --no-owner < /path/to/latest-db.dump
```

### Step 5. Build and start the backend and frontend

```bash
docker compose up -d --build
```

The backend runs additive schema migrations at startup. If your code is newer than the backup, the migrations bring the DB schema up to current — that's safe (additive only).

### Step 6. Reverse proxy + TLS

Re-run your reverse-proxy config (see `PRODUCTION.md`). Confirm:

```bash
curl https://<your-domain>/api/health
# → {"status":"ok","app":"A6 Stern","environment":"production","storage":true}
```

If `"storage": false`, MinIO is unreachable from the backend container — check `MINIO_*` env vars and network connectivity.

### Step 7. Smoke test

1. Log in as a known user. If you don't remember any user's password, reset via:
   ```sql
   SELECT id, username, email FROM users WHERE is_admin = true;
   -- Then either:
   --   1. Trigger a password reset via POST /api/auth/request-password-reset
   --      (requires SMTP) and follow the link.
   --   2. Set a known bcrypt hash directly:
   --      UPDATE users SET password_hash='<bcrypt hash>' WHERE username='admin';
   ```
2. Open a project. The room list and file grid should load.
3. Click a thumbnail. The image should render (this confirms MinIO + DB are in sync).
4. Trigger a small upload. Confirm the file appears immediately.
5. If pointclouds matter — upload a small LAZ, watch `conversion_status` move through `uploading → pending → processing → ready`.

If steps 3 and 4 work, the restore is functionally complete.

## Recovery time / recovery point

With nightly backups:

- **RPO** (max data loss): up to 24 hours.
- **RTO** (time to restore): ~30 min on a pre-provisioned host (most of it is the MinIO mirror and the backend container build).

If you need < 1-hour RPO, set up Postgres WAL archiving (`archive_mode=on`, `archive_command=...`) and point-in-time-recovery — that's outside this doc's scope.

## What to do if a restore fails

1. **Don't keep trying.** Stop and re-read the error.
2. Inspect the actual database state:
   ```bash
   docker exec -it a6_stern_db psql -U postgres a6_stern -c "\dt"
   ```
   If the tables exist but are empty, the restore command was a no-op (wrong file? wrong format?). If the tables don't exist, the backend hasn't started yet and ran no migrations.
3. Inspect MinIO:
   ```bash
   mc ls a6minio/
   mc ls a6minio/construction-images/ | head
   ```
4. Look at backend logs: `docker compose logs --tail=200 backend`. The startup sequence (`Base.metadata.create_all`, migrations, bucket creation) prints each step. The line that fails tells you which dependency is broken.

If the dump file itself is corrupt:

```bash
pg_restore -l /path/to/dump  # lists the TOC; errors here = corrupt file
```

If the dump is corrupt and you've only kept one backup — you've learned why the retention table in this doc says **30 days local + 90 days off-site**.

## Test schedule

Put this in the team calendar:

| When | What |
|---|---|
| Quarterly | Restore the latest backup into a sandbox host. Run smoke tests. Time it. |
| After a major schema change | Restore an older backup against current code. Confirm migrations apply cleanly. |
| When ownership of the deploy changes | Walk the new owner through a full Scenario C run. |

A backup you've never restored is not a backup.
