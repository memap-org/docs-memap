# Learning Service

## Overview

The Learning Service manages quizzes and assessments for the MeMap platform. It provides quiz creation, management, student attempts, and grading functionality. This service is currently a **Work in Progress**.

## Key Features

- **Quiz Management**: Create, update, delete quizzes (Teacher-only)
- **Question Management**: Multiple-choice questions with options
- **Visibility Control**: Draft/Published status for quizzes
- **Roadmap Integration**: Link quizzes to specific roadmaps
- **Pagination & Search**: Filter and search quizzes
- **Role-Based Access**: Teacher-only endpoints with Spring Security

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **Database**: MySQL 8.0+
- **Security**: Spring Security, OAuth2 Resource Server (Keycloak)
- **Migration**: Flyway
- **Build Tool**: Maven

## Port Configuration

- **Service Port**: `8089`
- **Base URL**: `http://localhost:8089`
- **Context Path**: `/learning`
- **Full URL**: `http://localhost:8089/learning`

## Database

The service uses MySQL for storing:

- Quizzes (title, description, roadmap association, visibility)
- Questions (text, correct answer)
- Options (answer choices)
- Quiz attempts (student submissions)
- Attempt answers (student responses)

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MySQL 8.0+
- Keycloak (for authentication)

### Running the Service

```bash
# Navigate to service directory
cd learning-serice

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yaml):

```yaml
server:
  port: 8089
  servlet:
    context-path: /learning

spring:
  application:
    name: learning-service
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI}
  datasource:
    url: jdbc:mysql://localhost:3306/learning_service
    username: learning_service
    password: learning_service
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration

cors:
  allowed-origins:
    - http://localhost:3000
    - http://localhost:5173
  allowed-methods: GET, POST, PUT, DELETE, OPTIONS
```

### Environment Variables

- `KEYCLOAK_ISSUER_URI`: Keycloak issuer URI for JWT validation

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Identity Service**: JWT token validation
- **Roadmap Service**: Link quizzes to roadmaps
- **Roadmap AI Service**: Generate quiz questions with AI

## Security

### Authentication

- OAuth2 Resource Server with Spring Security
- JWT tokens issued by Keycloak
- Token validation on every request

### Authorization

- **Teacher Role Required**: All quiz management endpoints require `ROLE_TEACHER`
- **Student Role**: Future endpoints for taking quizzes (coming soon)
- Role-based access control with `@PreAuthorize`

### CORS

- Configured for frontend development
- Allows credentials and specific origins
- Supports all standard HTTP methods

## Quiz Lifecycle

1. **Draft**: Quiz created but not visible to students
2. **Published**: Quiz is visible (set `visible: true`)
3. **Edit**: Only draft quizzes can be edited
4. **Delete**: Only draft quizzes can be deleted

## Current Status: Work in Progress

### Implemented Features

✅ Quiz CRUD operations (Teacher-only)  
✅ Question and option management  
✅ Visibility toggle  
✅ Pagination and filtering  
✅ Search functionality  
✅ Database migrations with Flyway

### Planned Features (Not Yet Implemented)

🚧 Student quiz-taking functionality  
🚧 Quiz attempt tracking  
🚧 Automatic grading  
🚧 Quiz results and analytics  
🚧 Time limits for quizzes  
🚧 Quiz categories and tags  
🚧 Question randomization  
🚧 Leaderboards

## Database Schema

### Tables

- `quizzes` - Quiz metadata
- `questions` - Quiz questions
- `options` - Answer options for questions
- `quiz_attempts` - Student quiz attempts
- `attempt_answers` - Student answers for each attempt

## Health Check

- Health endpoint: `http://localhost:8089/learning/actuator/health`
- Info endpoint: `http://localhost:8089/learning/actuator/info`

## Logging

- Debug level enabled for development
- Logs include quiz operations, authentication, and errors
- SLF4J with Logback

## Migration Management

Database schema is managed with Flyway:

- Migration files in `src/main/resources/db/migration`
- Automatic migration on startup
- Baseline on migrate enabled

## Future Enhancements

1. **Student Quiz Taking**: Complete quiz-taking flow for students
2. **Grading System**: Automatic and manual grading
3. **Analytics**: Quiz performance analytics and reports
4. **Question Bank**: Reusable question library
5. **Question Types**: Support for multiple question types (multiple choice, true/false, essay)
6. **Rich Text**: Support for rich text in questions (images, code, math)
7. **Timed Quizzes**: Set time limits for quiz completion
8. **Quiz Sharing**: Share quizzes with other teachers
9. **Import/Export**: Import/export quizzes in standard formats
10. **Mobile Support**: Optimize APIs for mobile apps
