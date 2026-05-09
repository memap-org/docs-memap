# Storage Service API Documentation

> **Branch:** `main` | **Port:** 8088 | **Context path:** `/storage`

---

## Base URLs

```
Via API Gateway:  http://localhost:8090/storage/...
Direct:           http://localhost:8088/storage/...
Swagger UI:       http://localhost:8088/storage/swagger-ui.html
Health:           http://localhost:8088/storage/actuator/health
```

---

## REST Endpoints

### File Management — `FileController`

Base: `/file`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/file/upload` | JWT | Upload a file (`multipart/form-data`) |
| GET | `/file/{fileId}/download` | JWT | Download a file (browser download) |
| GET | `/file/{fileId}/access` | **Public** | Access file inline (images/PDFs display in browser) |
| GET | `/file/{fileId}` | JWT | Get file metadata |
| GET | `/file/my-files` | JWT | List files uploaded by current user |
| DELETE | `/file/{fileId}` | JWT | Delete a file (owner only) |

The `/file/{fileId}/access` endpoint is intentionally **public** — used for inline viewing of images and PDFs embedded in roadmap nodes without authentication.

### Internal File Management — `InternalFileController`

Base: `/file/internal`

| Method | Path | Auth | Description |
|---|---|---|---|
| DELETE | `/file/internal/{fileId}` | Service-internal | Delete a file without ownership check (called by other services) |

### Roadmap Storage Quota — `RoadmapStorageController`

Base: `/roadmap-storage`

| Method | Path | Auth | Description |
|---|---|---|---|
| PUT | `/roadmap-storage/quota` | JWT / Internal | Update storage quota for a roadmap |

---

## Response Examples

**Upload response:**
```json
{
  "code": 1000,
  "result": {
    "fileId": "uuid",
    "fileName": "example.png",
    "contentType": "image/png",
    "size": 102400,
    "uploadedAt": "2025-01-01T00:00:00Z"
  }
}
```

**File info response:**
```json
{
  "code": 1000,
  "result": {
    "fileId": "uuid",
    "fileName": "string",
    "contentType": "string",
    "size": 0,
    "accessUrl": "http://localhost:8090/storage/file/{fileId}/access",
    "uploadedAt": "ISO-8601"
  }
}
```

---

## Storage Backend

Files are stored in **MinIO** (S3-compatible object storage). Configuration via environment variables in `.env` / `docker-compose.yml`.
