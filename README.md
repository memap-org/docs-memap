# MeMap Backend - Microservices Documentation

## Overview

MeMap is a learning roadmap platform built with a microservices architecture using Spring Boot. The system allows users to create, share, and follow learning roadmaps with AI-powered features for content generation and recommendations.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Client Applications                             │
│                        (Web App, Mobile App, etc.)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway (Future)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌──────────────┬──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Identity   │ │   Profile    │ │   Roadmap    │ │  Roadmap AI  │ │    File      │
│   Service    │ │   Service    │ │   Service    │ │   Service    │ │   Service    │
│   :8081      │ │   :8082      │ │   :8083      │ │   :8084      │ │   :8085      │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
        │              │              │              │              │
        └──────────────┴──────────────┼──────────────┴──────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Infrastructure                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │    MySQL    │  │    Redis    │  │  RabbitMQ   │  │   MongoDB   │        │
│  │   :3306     │  │   :6379     │  │ :5672/15672 │  │   :27017    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                      ┌─────────────┐        │
│                                                      │    MinIO    │        │
│                                                      │   :9000     │        │
│                                                      └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Services

| Service                | Port | Description                         | Database      | Documentation                    |
| ---------------------- | ---- | ----------------------------------- | ------------- | -------------------------------- |
| **Identity Service**   | 8081 | Authentication, JWT tokens, OAuth2  | MySQL         | [📖 Docs](./identity-service/)   |
| **Profile Service**    | 8082 | User profiles, avatars, email       | MySQL + Redis | [📖 Docs](./profile-service/)    |
| **Roadmap Service**    | 8083 | Roadmaps, nodes, permissions        | MySQL + Redis | [📖 Docs](./roadmap-service/)    |
| **Roadmap AI Service** | 8084 | AI-powered roadmap generation, chat | MongoDB       | [📖 Docs](./roadmap-ai-service/) |
| **File Service**       | 8085 | File upload/download, storage       | MySQL + MinIO | [📖 Docs](./file-service/)       |
| **Storage Service**    | 8088 | Storage service foundation          | MySQL + Local | [📖 Docs](./storage-service/)    |
| **Learning Service**   | 8086 | Quiz management (WIP)               | MySQL         | [📖 Docs](./learning-service/)   |
| **Frontend**           | 5173 | React + TypeScript web application  | N/A           | [📖 Docs](./memap-frontend/)     |

## Service-Specific Documentation

Each service has its own detailed documentation folder:

- **[Identity Service Documentation](./identity-service/)** - Authentication, JWT, user credentials
- **[Profile Service Documentation](./profile-service/)** - User profiles, avatars, password management
- **[Roadmap Service Documentation](./roadmap-service/)** - Roadmap management, nodes, edges, progress tracking
- **[Roadmap AI Service Documentation](./roadmap-ai-service/)** - AI generation, summaries, chat, Q&A
- **[File Service Documentation](./file-service/)** - File upload/download, MinIO integration
- **[Storage Service Documentation](./storage-service/)** - Storage service foundation (setup only)
- **[Learning Service Documentation](./learning-service/)** - Quiz management (WIP)
- **[Frontend Documentation](./memap-frontend/)** - React application, routing, state management

Each service documentation includes:

- **README.md** - Service overview, features, tech stack, getting started
- **API.md** - Complete API reference with request/response examples
- **ARCHITECTURE.md** - Architecture diagrams, components, design patterns

## Quick Start

### Prerequisites

- Java 21 (JDK)
- Maven 3.8+
- Docker & Docker Compose
- MySQL 8.0+
- Redis 6.0+
- RabbitMQ 3.9+
- Node.js 18+ (for frontend)

### Running with Docker Compose

```bash
# Clone the repository
cd spring-boot

# Start all infrastructure services
docker-compose up -d

# Or start individual services
docker-compose up -d redis rabbitmq
```

### Running Services Locally

```bash
# Navigate to a service
cd profile-service

# Run with Maven
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

## Project Structure

```
spring-boot/
├── docs/                    # Project documentation
│   ├── identity-service/    # Identity service documentation
│   ├── profile-service/     # Profile service documentation
│   ├── roadmap-service/     # Roadmap service documentation
│   ├── roadmap-ai-service/  # AI service documentation
│   ├── file-service/        # File service documentation
│   ├── storage-service/     # Storage service documentation
│   ├── learning-service/    # Learning service documentation
│   └── memap-frontend/      # Frontend documentation
├── identity-service/        # Authentication & authorization
├── profile-service/         # User profile management
├── roadmap-service/         # Roadmap CRUD operations
├── roadmap-ai-service/      # AI features & chat
├── file-service/            # File storage with MinIO
├── storage-service/         # Storage service foundation
├── learning-serice/         # Learning management (WIP)
├── memap-frontend/          # React + TypeScript frontend
├── upload/                  # Local upload directory
└── docker-compose.yml       # Infrastructure services
```

## Technology Stack

### Backend

| Technology          | Version | Purpose               |
| ------------------- | ------- | --------------------- |
| Spring Boot         | 3.5.x   | Application framework |
| Spring Security     | 6.x     | Security & OAuth2     |
| Spring Data JPA     | 3.x     | Data access (MySQL)   |
| Spring Data MongoDB | 3.x     | Data access (MongoDB) |
| Spring AMQP         | 3.x     | Message queuing       |
| Spring AI           | 1.x     | AI/LLM integration    |

### Infrastructure

| Technology | Purpose                         |
| ---------- | ------------------------------- |
| MySQL 8.0  | Primary relational database     |
| MongoDB    | Document storage for AI service |
| Redis 6.0  | Caching & session storage       |
| RabbitMQ   | Message broker                  |
| MinIO      | Object storage (S3-compatible)  |

### Development Tools

| Tool              | Purpose                          |
| ----------------- | -------------------------------- |
| MapStruct         | Object mapping                   |
| Lombok            | Boilerplate reduction            |
| SpringDoc OpenAPI | API documentation                |
| OpenFeign         | Service-to-service communication |

## API Documentation

Each service exposes Swagger UI for API documentation:

| Service            | Swagger URL                                    |
| ------------------ | ---------------------------------------------- |
| Identity Service   | http://localhost:8081/identity/swagger-ui.html |
| Profile Service    | http://localhost:8082/profile/swagger-ui.html  |
| Roadmap Service    | http://localhost:8083/roadmap/swagger-ui.html  |
| Roadmap AI Service | http://localhost:8084/ai/swagger-ui.html       |
| File Service       | http://localhost:8085/file/swagger-ui.html     |
| Storage Service    | http://localhost:8088/storage/swagger-ui.html  |

## Error Code Ranges

Each service uses a unique error code range:

| Service          | Code Range | Example                   |
| ---------------- | ---------- | ------------------------- |
| Roadmap Service  | 1xxx       | 1001 - Roadmap not found  |
| Profile Service  | 2xxx       | 2006 - User not found     |
| Learning Service | 3xxx       | 3101 - Course not found   |
| Storage Service  | 4xxx       | 4001 - Argument not valid |
| General          | 9999       | Uncategorized error       |

## Inter-Service Communication

```
┌─────────────────┐         ┌─────────────────┐
│ Profile Service │◄───────►│Identity Service │
│                 │  Feign  │                 │
└─────────────────┘         └─────────────────┘
         │
         │ RabbitMQ
         ▼
┌─────────────────┐
│ Roadmap Service │
└─────────────────┘
```

- **Synchronous**: OpenFeign for REST calls
- **Asynchronous**: RabbitMQ for event-driven communication

## Security

- **Authentication**: JWT tokens with OAuth2 Resource Server
- **Authorization**: Role-based (STUDENT, TEACHER, ADMIN)
- **Token Storage**: HTTP-only cookies (secure, SameSite)

## Environment Configuration

### Required Environment Variables

```env
# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=memap_db
DB_USERNAME=root
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# RabbitMQ
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=admin
RABBITMQ_PASSWORD=admin

# AI Service (OpenAI)
OPENAI_API_KEY=your_api_key
```

## Development Guidelines

### Code Style

- Follow Java naming conventions
- Use Lombok for boilerplate reduction
- Document public APIs with Javadoc

### Git Workflow

1. Create feature branch from `main`
2. Implement changes
3. Create pull request
4. Code review
5. Merge to `main`

### Adding a New Service

1. Create new Spring Boot project
2. Add common dependencies
3. Configure error handling
4. Add to docker-compose.yml
5. Document in this README

## Related Documentation

- **[Identity Service Documentation](./identity-service/)** - Authentication & authorization
- **[Profile Service Documentation](./profile-service/)** - User profile management
- **[Roadmap Service Documentation](./roadmap-service/)** - Roadmap CRUD operations
- **[Roadmap AI Service Documentation](./roadmap-ai-service/)** - AI-powered features
- **[File Service Documentation](./file-service/)** - File upload & storage
- **[Learning Service Documentation](./learning-service/)** - Quiz management
- **[Frontend Documentation](./memap-frontend/)** - React application

## License

This project is proprietary software for MeMap application.

## Contact

For questions or support, please contact the development team.

---

**Documentation Structure Updated**: January 29, 2026  
Each service now has comprehensive documentation including README, API reference, and architecture details.
