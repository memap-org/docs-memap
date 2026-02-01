# Profile Service

## Overview

The Profile Service manages user profiles, avatars, email operations, and user-related information in the MeMap platform. It handles user registration, profile updates, password management, and integrates with the Identity Service for authentication.

## Key Features

- User registration and profile management
- Avatar upload and management
- Password change and forgot password functionality
- Email-based user search
- Bulk user information retrieval
- Integration with RabbitMQ for event-driven communication
- Redis caching for improved performance
- File service integration for avatar storage

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **Database**: MySQL 8.0+
- **Cache**: Redis 6.0+
- **Message Broker**: RabbitMQ 3.9+
- **Security**: Spring Security, OAuth2
- **Build Tool**: Maven

## Port Configuration

- **Service Port**: `8082`
- **Base URL**: `http://localhost:8082`
- **Context Path**: `/profile`

## Database

The service uses MySQL for storing:

- User profiles (personal information)
- User preferences
- Avatar references
- User metadata

## Message Queue Integration

The service integrates with RabbitMQ for:

- User registration events
- Profile update notifications
- Email notifications
- Cross-service communication

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MySQL 8.0+
- Redis 6.0+
- RabbitMQ 3.9+

### Running the Service

```bash
# Navigate to service directory
cd profile-service

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yml):

```yaml
server:
  port: 8082
  servlet:
    context-path: /profile

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/profile_service
    username: root
    password: your_password

  redis:
    host: localhost
    port: 6379

  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Identity Service**: Authentication and authorization
- **File Service**: Avatar and file storage
- **Roadmap Service**: User profile references
- **Learning Service**: User progress tracking

## Security

- All endpoints require JWT authentication (except registration)
- Role-based access control (STUDENT, TEACHER, ADMIN)
- Password hashing with BCrypt
- Secure avatar upload with file type validation
