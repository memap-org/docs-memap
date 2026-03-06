# Storage Service Architecture

## Overview

The storage-service is scaffolded with standard Spring Boot layers and shared conventions. Implementation details will be added in a follow-up change.

## Package Layout

```
storage-service/
├── controller/   # REST controllers (placeholder)
├── dto/          # Request/response models
├── entity/       # JPA entities (placeholder)
├── repository/   # Data access (placeholder)
├── service/      # Business logic (placeholder)
├── storage/      # Storage backend adapters (placeholder)
└── config/       # Security/OpenAPI configuration
```

## Configuration

- Local storage backend placeholders in `app.storage.*`
- MySQL connectivity via `spring.datasource.*`
- OAuth2 JWT via `spring.security.oauth2.resourceserver.jwt.*`
