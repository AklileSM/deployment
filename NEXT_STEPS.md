# Next steps

> **Email setup moved.** SMTP and `FRONTEND_URL` are now part of the standard config, see the "Email (SMTP)" section in [README.md](README.md), and `backend/AUTH_AND_EMAIL.md` for the full token flow and tested providers.

---

## 1. Unitree GO2 robot automated data collection

The lab's Unitree GO2 wheeled robot is planned as the primary data collection platform. Instead of manually uploading files through the browser, the robot will capture data on-site and push it directly to the platform.

### Intended workflow

1. **Capture**: The GO2 scans a room or area, collecting images, video, and/or LiDAR point clouds.
2. **Pre-flight validation**: Before uploading, an on-robot or edge process checks data usability: minimum point cloud density, image sharpness/exposure, spatial coverage. Files that fail the checks are flagged or discarded rather than sent.
3. **Upload**: Valid captures are pushed to the backend via the existing REST API, tagged with the correct project, room, and capture date. The backend handles storage, Potree conversion, and AI analysis exactly as it would for a manual upload.
4. **Review**: Results appear in the platform immediately, ready for field reports and annotations.

### Integration points in the current API


| Step                         | Endpoint                                                                                                |
| ---------------------------- | ------------------------------------------------------------------------------------------------------- |
| Authenticate the robot       | `POST /api/auth/login` → store JWT                                                                      |
| Upload image / video / PDF   | `POST /api/upload/single`                                                                               |
| Upload point cloud (LAZ/LAS) | `POST /api/upload/pointcloud/init` → `/api/upload/pointcloud/chunk` → `/api/upload/pointcloud/complete` |
| Poll conversion status       | `GET /api/files/{file_id}/conversion-status`                                                            |


The API already accepts JWT bearer tokens so the robot client can authenticate as a dedicated service account.

### Open design questions

- On-robot validation vs. edge-node validation (a companion laptop or local server on the same network)
- Whether validation failures should be stored as low-confidence drafts or dropped entirely
- Network handoff strategy: robot uploads over Wi-Fi when back at base, or live upload during the scan
- Capture metadata format (room ID, capture date) manual input vs. derived from the robot's map/pose

