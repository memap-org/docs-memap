# Roadmap Service

## Overview

The Roadmap Service is the core service for managing learning roadmaps in the MeMap platform. It handles roadmap creation, editing, node/edge management, progress tracking, permissions, and collaboration features.

## Key Features

- **Roadmap Management**: Create, read, update, delete roadmaps
- **Node & Edge Operations**: Manage roadmap nodes and connections
- **Progress Tracking**: Track user progress through roadmap nodes
- **Access Control**: Private, group, and public roadmap scopes
- **Permissions Management**: Grant/revoke user access and edit permissions
- **Categories**: Organize roadmaps by categories
- **File Resources**: Upload and manage roadmap-related files
- **Image Upload**: Roadmap thumbnail images
- **Collaboration**: Share roadmaps with other users
- **Real-time Updates**: WebSocket support for collaborative editing

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **Database**: MySQL 8.0+ (primary data), MongoDB (file metadata)
- **Cache**: Redis
- **Security**: Spring Security, OAuth2 Resource Server (Keycloak)
- **Build Tool**: Maven
- **Message Queue**: RabbitMQ (for async communication)
- **File Storage**: Local filesystem

## Port Configuration

- **Service Port**: `8083`
- **Base URL**: `http://localhost:8083`
- **Context Path**: `/road-map`
- **Full URL**: `http://localhost:8083/road-map`

## Database

The service uses MySQL for storing:

- Roadmap metadata (title, description, scope, owner)
- Nodes and edges
- Access permissions
- Progress tracking data
- Categories
- User relationships

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MySQL 8.0+
- MongoDB (for file metadata)
- Redis (for caching)
- Keycloak (for authentication)

### Running the Service

```bash
# Navigate to service directory
cd roadmap-service

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yaml):

```yaml
server:
  port: 8083
  servlet:
    context-path: /road-map

spring:
  application:
    name: roadmap-service
  data:
    mongodb:
      uri: ${MONGODB_URL}
    redis:
      host: localhost
      port: 6382
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI}

app:
  file:
    storage-dir: /data/roadmap/files
```

### Environment Variables

- `MONGODB_URL`: MongoDB connection string
- `KEYCLOAK_ISSUER_URI`: Keycloak issuer URI for JWT validation

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Identity Service**: JWT token validation
- **Profile Service**: User profile information
- **File Service**: File upload/download integration
- **Roadmap AI Service**: AI-powered roadmap generation
- **Learning Service**: Quiz integration

## Access Scopes

Roadmaps support three access scopes:

1. **PRIVATE**: Only owner can view and edit
2. **GROUP**: Owner + specifically granted users can view/edit
3. **PUBLIC**: Anyone can view, only owner can edit

## Progress Tracking

Users can track their progress through roadmap nodes with status:

- `PENDING`: Not started
- `IN_PROGRESS`: Currently working on
- `DONE`: Completed
- `SKIPPED`: Skipped this node

## Swagger Documentation

Access Swagger UI at: `http://localhost:8083/road-map/swagger-ui.html`
