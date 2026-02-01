# File Service

## Overview

The File Service is responsible for file storage and management in the MeMap platform. It handles file uploads, downloads, and presigned URL generation using MinIO object storage.

## Key Features

- **File Upload**: Upload files with automatic metadata storage
- **File Download**: Download files directly or via presigned URLs
- **Presigned URLs**: Generate temporary URLs for secure file access
- **Object Storage**: MinIO integration for scalable file storage
- **Metadata Management**: Store file metadata in MySQL
- **Security**: Authenticated access to files

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **Object Storage**: MinIO
- **Database**: MySQL 8.0+ (metadata)
- **Security**: Spring Security
- **Build Tool**: Maven
- **Migration**: Flyway

## Port Configuration

- **Service Port**: `8087`
- **Base URL**: `http://localhost:8087`
- **Context Path**: `/`

## Storage

### MinIO Object Storage

- **Endpoint**: `http://localhost:9000`
- **Bucket**: `memap-file-service`
- **Access**: Credential-based authentication

### Database

MySQL is used for storing file metadata:

- File ID
- Original filename
- Content type
- Size
- Upload timestamp
- User ID (uploader)
- Storage path

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MySQL 8.0+
- MinIO server

### Running MinIO

```bash
# Run MinIO with Docker
docker run -p 9000:9000 -p 9001:9001 \
  -e "MINIO_ROOT_USER=admin123" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  minio/minio server /data --console-address ":9001"
```

### Running the Service

```bash
# Navigate to service directory
cd file-service

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yml):

```yaml
server:
  port: 8087

spring:
  application:
    name: file-service
  datasource:
    url: jdbc:mysql://localhost:3306/file-service-db
    username: file-service
    password: file-service
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true

minio:
  endpoint: http://localhost:9000
  access-key: admin123
  secret-key: admin123
  bucket: memap-file-service
```

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Roadmap Service**: Uses File Service for roadmap images and resources
- **Profile Service**: Uses File Service for user avatars
- **Identity Service**: Authentication validation

## Security

- JWT-based authentication
- File access control based on user permissions
- Presigned URLs expire after configured duration
- MinIO bucket policies for access control

## File Size Limits

- Maximum file size: 10 MB (configurable)
- Supported file types: All types (validation in client)
- Bucket storage: Unlimited (depends on MinIO configuration)

## Bucket Structure

```
memap-file-service/
├── roadmaps/
│   ├── {roadmapId}/
│   │   ├── images/
│   │   │   └── thumbnail.jpg
│   │   └── resources/
│   │       └── document.pdf
├── profiles/
│   └── {userId}/
│       └── avatar.jpg
└── temp/
    └── {tempId}/
        └── file.ext
```
