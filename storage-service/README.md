# Storage Service

## Overview

The storage-service provides file upload, download, and management capabilities for MeMap. It stores files on the local filesystem and persists metadata (filename, size, checksum, owner) in MySQL.

## Service Info

- **Port**: 8088
- **Context Path**: `/storage`
- **Swagger UI**: http://localhost:8088/storage/swagger-ui.html
- **API Docs**: http://localhost:8088/storage/api-docs

## Features

- File upload with automatic MD5 checksum computation
- Download files with proper Content-Type and Content-Disposition headers
- File metadata retrieval
- List files owned by the authenticated user
- Delete files (owner-only access control)
- Date-based storage paths: `{year}/{month}/{uuid}.{ext}`

## Configuration

### Environment Variables

| Variable              | Description               | Example                                  |
| --------------------- | ------------------------- | ---------------------------------------- |
| `STORAGE_DB_URL`      | MySQL JDBC URL            | `jdbc:mysql://localhost:3306/storage_db` |
| `STORAGE_DB_USERNAME` | Database username         | `storage_user`                           |
| `STORAGE_DB_PASSWORD` | Database password         | `secret`                                 |
| `KEYCLOAK_ISSUER_URI` | Keycloak realm issuer URI | `http://localhost:8180/realms/memap`     |

### Application Properties

| Property                                    | Default                         | Description                     |
| ------------------------------------------- | ------------------------------- | ------------------------------- |
| `app.storage.backend`                       | `local`                         | Storage backend type            |
| `app.storage.local.base-dir`                | `/data/storage/files`           | Base directory for file storage |
| `app.storage.download-url-prefix`           | `http://localhost:8088/storage` | URL prefix for download links   |
| `spring.servlet.multipart.max-file-size`    | `1024MB`                        | Maximum file upload size        |
| `spring.servlet.multipart.max-request-size` | `1024MB`                        | Maximum request size            |

## Getting Started

### Prerequisites

1. MySQL database created and accessible
2. Keycloak realm configured with JWT issuer
3. Storage directory writable by the application

### Setup

1. Create the storage directory:

   ```bash
   mkdir -p /data/storage/files
   ```

2. Set environment variables:

   ```bash
   export STORAGE_DB_URL=jdbc:mysql://localhost:3306/storage_db
   export STORAGE_DB_USERNAME=storage_user
   export STORAGE_DB_PASSWORD=secret
   export KEYCLOAK_ISSUER_URI=http://localhost:8180/realms/memap
   ```

3. Run the service:

   ```bash
   cd storage-service
   ./mvnw spring-boot:run
   ```

4. Access Swagger UI at http://localhost:8088/storage/swagger-ui.html

### Docker Setup

When running with Docker, mount a volume for file storage:

```bash
docker run -v /host/storage:/data/storage/files \
  -e STORAGE_DB_URL=jdbc:mysql://db:3306/storage_db \
  -e STORAGE_DB_USERNAME=storage_user \
  -e STORAGE_DB_PASSWORD=secret \
  -e KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/memap \
  storage-service
```

## Database Schema

The service uses Flyway migrations. The `file_metadata` table stores:

| Column          | Type          | Description                       |
| --------------- | ------------- | --------------------------------- |
| `id`            | UUID          | Primary key                       |
| `name`          | VARCHAR(500)  | Display name (custom or original) |
| `original_name` | VARCHAR(500)  | Original uploaded filename        |
| `content_type`  | VARCHAR(255)  | MIME type                         |
| `size`          | BIGINT        | File size in bytes                |
| `md5_checksum`  | CHAR(32)      | MD5 hash for integrity            |
| `storage_path`  | VARCHAR(1000) | Relative path in storage backend  |
| `owner_id`      | VARCHAR(255)  | User ID from JWT subject          |
| `created_at`    | TIMESTAMP     | Upload timestamp                  |
| `updated_at`    | TIMESTAMP     | Last modification timestamp       |

## Security

All file endpoints require authentication via JWT Bearer token. The owner_id is extracted from the JWT `sub` claim. Only the file owner can delete their files.
