# Next steps

## 1. Email setup (password reset and email verification)

The password reset and email verification flows are fully built into the app but require an SMTP server to deliver emails. Without it the flows still work, the backend logs a warning and skips sending but users will not receive any emails.

### Configuration

Add the following to `.env` and restart the backend (`docker compose restart backend`):

```env
SMTP_HOST=smtp.yourmailprovider.com
SMTP_PORT=587
SMTP_USERNAME=you@yourdomain.com
SMTP_PASSWORD=yourpassword
SMTP_FROM_EMAIL=noreply@yourdomain.com
SMTP_FROM_NAME=A6 Stern
SMTP_USE_TLS=true
```

Also set `FRONTEND_URL` to the public address the browser uses to reach the app (e.g. `http://192.168.1.10`). This is prepended to the `/verify-email?token=...` and `/reset-password?token=...` links inside outgoing emails, if it is wrong, email links will not work.

Common SMTP providers: Resend, Mailgun, Postmark, Gmail with an App Password.

### Testing without SMTP

You can exercise the full token flow via the API docs at `http://<host>:3002/api/docs` without an SMTP server:

1. `POST /api/auth/request-password-reset` saves a reset token to the database
2. Read the token directly from Postgres via pgAdmin (port 5050)
3. `GET /api/auth/validate-reset-token?token=<token>` confirms it is valid
4. `POST /api/auth/reset-password`sets the new password

---

## 2. Unitree GO2 robot automated data collection

The lab's Unitree GO2 wheeled robot is planned as the primary data collection platform. Instead of manually uploading files through the browser, the robot will capture data on-site and push it directly to the platform.

### Intended workflow

1. **Capture**: The GO2 scans a room or area, collecting images, video, and/or LiDAR point clouds.
2. **Pre-flight validation**: Before uploading, an on-robot or edge process checks data usability: minimum point cloud density, image sharpness/exposure, spatial coverage. Files that fail the checks are flagged or discarded rather than sent.
3. **Upload**: Valid captures are pushed to the backend via the existing REST API, tagged with the correct project, room, and capture date. The backend handles storage, Potree conversion, and AI analysis exactly as it would for a manual upload.
4. **Review**: Results appear in the platform immediately, ready for field reports and annotations.

### Integration points in the current API


| Step                         | Endpoint                                                               |
| ---------------------------- | ---------------------------------------------------------------------- |
| Authenticate the robot       | `POST /api/auth/login` → store JWT                                     |
| Upload image / video / PDF   | `POST /api/upload/single`                                              |
| Upload point cloud (LAZ/LAS) | `POST /api/upload/init` → `/api/upload/chunk` → `/api/upload/complete` |
| Poll conversion status       | `GET /api/files/{file_id}/conversion-status`                           |


The API already accepts JWT bearer tokens so the robot client can authenticate as a dedicated service account.

### Open design questions

- On-robot validation vs. edge-node validation (a companion laptop or local server on the same network)
- Whether validation failures should be stored as low-confidence drafts or dropped entirely
- Network handoff strategy: robot uploads over Wi-Fi when back at base, or live upload during the scan
- Capture metadata format (room ID, capture date) manual input vs. derived from the robot's map/pose

