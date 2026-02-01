# Identity Service

## Overview

The Identity Service is responsible for authentication, authorization, and user credential management in the MeMap platform. It handles JWT token generation, validation, and user identity operations.

## Key Features

- User authentication (login/logout)
- JWT token generation and validation
- OAuth2 integration support
- Token refresh mechanism
- User credentials management (CRUD operations)
- Role-based access control

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **Database**: MySQL 8.0+
- **Security**: Spring Security, JWT
- **Build Tool**: Maven

## Port Configuration

- **Service Port**: `8081`
- **Base URL**: `http://localhost:8081`
- **Context Path**: `/identity`

## Database

The service uses MySQL for storing:

- User credentials
- JWT tokens
- OAuth2 client configurations
- User roles and permissions

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MySQL 8.0+

### Running the Service

```bash
# Navigate to service directory
cd identity-service

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yml):

```yaml
server:
  port: 8081
  servlet:
    context-path: /identity

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/identity_service
    username: root
    password: your_password
```

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Profile Service**: Uses Identity Service for authentication validation
- **All Services**: Depend on Identity Service for JWT token validation

## Security

- All passwords are hashed using BCrypt
- JWT tokens are signed using HS512 algorithm
- Tokens are stored in HTTP-only cookies for security
- Access tokens expire after 10 minutes
- Refresh tokens expire after 7 days
