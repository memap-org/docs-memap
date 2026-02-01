# MeMap Backend Architecture

## Service-Specific Architecture Documentation

For detailed architecture documentation of each service, please refer to their respective folders:

- **[Identity Service Architecture](./identity-service/ARCHITECTURE.md)** - Authentication, JWT, security
- **[Profile Service Architecture](./profile-service/ARCHITECTURE.md)** - Profile management, caching, messaging
- **[Roadmap Service Architecture](./roadmap-service/ARCHITECTURE.md)** - Roadmap management, permissions, progress tracking
- **[Roadmap AI Service Architecture](./roadmap-ai-service/ARCHITECTURE.md)** - AI integration, RAG, vector store
- **[File Service Architecture](./file-service/ARCHITECTURE.md)** - File storage, MinIO integration
- **[Learning Service Architecture](./learning-service/ARCHITECTURE.md)** - Quiz management
- **[Payment Service Architecture](./payment-service/ARCHITECTURE.md)** - Credit management, Stripe integration
- **[Frontend Architecture](./memap-frontend/ARCHITECTURE.md)** - React application architecture

## System Overview

MeMap follows a microservices architecture pattern where each service is independently deployable and responsible for a specific business domain.

## Architecture Diagram

```
                                    ┌─────────────────┐
                                    │     Clients     │
                                    │  (Web/Mobile)   │
                                    └────────┬────────┘
                                             │
                                             │ HTTPS
                                             ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                                  API Layer                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                         Load Balancer / Nginx                             │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────┘
                                             │
         ┌───────────────┬───────────────────┼───────────────────┬───────────────┬───────────────┐
         ▼               ▼                   ▼                   ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Identity     │ │     Profile     │ │     Roadmap     │ │   Roadmap AI    │ │      File       │ │     Payment     │
│    Service      │ │     Service     │ │     Service     │ │    Service      │ │    Service      │ │    Service      │
│                 │ │                 │ │                 │ │                 │ │                 │ │   (NestJS)      │
│ • Auth/Login    │ │ • User CRUD     │ │ • Roadmap CRUD  │ │ • AI Generation │ │ • Upload        │ │ • Credits       │
│ • JWT Tokens    │ │ • Avatars       │ │ • Nodes         │ │ • Chat/Q&A      │ │ • Download      │ │ • Subscriptions │
│ • OAuth2        │ │ • Email         │ │ • Permissions   │ │ • Embeddings    │ │ • MinIO Storage │ │ • Stripe        │
│                 │ │ • Password      │ │ • Categories    │ │ • Summaries     │ │                 │ │ • Invoices      │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │                   │                   │                   │
         │                   │                   │                   │                   │                   │
         └───────────────────┴─────────┬─────────┴───────────────────┴───────────────────┴───────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────────────────────────────┐
│                             Message Broker                                               │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              RabbitMQ                                               │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                     │ │
│  │  │  user.events    │  │ roadmap.events  │  │  email.queue    │                     │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                     │ │
│  └────────────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────────────────────────────┐
│                               Data Layer                                                 │
│                                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │     MySQL       │  │   PostgreSQL    │  │     Redis       │  │    MongoDB      │    │
│  │                 │  │                 │  │                 │  │                 │    │
│  │ • Identity DB   │  │ • Payment DB    │  │ • Session Cache │  │ • AI Vectors    │    │
│  │ • Profile DB    │  │ • Credits       │  │ • Token Cache   │  │ • Chat History  │    │
│  │ • Roadmap DB    │  │ • Subscriptions │  │ • Rate Limiting │  │ • Embeddings    │    │
│  │ • File DB       │  │ • Transactions  │  │                 │  │                 │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                                          │
│  ┌─────────────────┐                                                                    │
│  │     MinIO       │                                                                    │
│  │                 │                                                                    │
│  │ • File Storage  │                                                                    │
│  │ • Avatars       │                                                                    │
│  │ • Documents     │                                                                    │
│  └─────────────────┘                                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────────────────────────────┐
│                            External Services                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                          │
│  │   Stripe API    │  │   OpenAI API    │  │   Groq API      │                          │
│  │ • Payments      │  │ • Embeddings    │  │ • LLM Inference │                          │
│  │ • Subscriptions │  │                 │  │ • llama-3.1     │                          │
│  │ • Webhooks      │  │                 │  │                 │                          │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

## Service Details

### Identity Service (Port 8081)

**Responsibility**: Authentication and authorization

```
identity-service/
├── controller/     # Auth endpoints
├── dto/           # Request/Response DTOs
├── entity/        # User credentials, tokens
├── service/       # Auth logic, JWT generation
└── repository/    # Data access
```

**Key Features**:

- User login/logout
- JWT token generation & validation
- OAuth2 provider integration
- Token refresh mechanism

### Profile Service (Port 8082)

**Responsibility**: User profile management

```
profile-service/
├── controller/     # User, Email endpoints
├── dto/           # Profile DTOs
├── entity/        # User profile entity
├── service/       # Profile business logic
├── messaging/     # RabbitMQ producers
└── config/        # Security, cache config
```

**Key Features**:

- User registration
- Profile CRUD operations
- Avatar upload
- Password management
- Email notifications

### Roadmap Service (Port 8083)

**Responsibility**: Learning roadmap management

```
roadmap-service/
├── controller/     # Roadmap, Node endpoints
├── dto/           # Roadmap DTOs
├── entity/        # Roadmap, Node, Permission
├── service/       # Roadmap business logic
├── event/         # Domain events
└── mapper/        # Entity-DTO mapping
```

**Key Features**:

- Roadmap CRUD
- Node management
- Permission system
- Categories
- Public/private roadmaps

### Roadmap AI Service (Port 8084)

**Responsibility**: AI-powered features

```
roadmap-ai-service/
├── controller/     # AI endpoints
├── dto/           # AI request/response
├── service/       # AI business logic
│   ├── ChatService
│   ├── RoadmapGeneratorService
│   ├── NodeSummaryService
│   └── DocumentIngestService
└── config/        # AI, embedding config
```

**Key Features**:

- AI roadmap generation
- Chat with roadmap context
- Node content summarization
- Question generation
- Vector embeddings (RAG)

### File Service (Port 8085)

**Responsibility**: File storage and retrieval

```
file-service/
├── controller/     # Upload/download endpoints
├── domain/        # File entity
├── service/       # File operations
└── config/        # MinIO configuration
```

**Key Features**:

- File upload to MinIO
- File download
- Metadata management
- Pre-signed URLs

### Payment Service (Port 8087)

**Responsibility**: Payment processing and credit management

```
payment-service/           # NestJS Application
├── modules/
│   ├── credits/       # Credit balance, transactions
│   ├── subscriptions/ # Subscription plans, billing
│   ├── payments/      # Payment history, invoices
│   ├── packages/      # Credit packages for purchase
│   ├── webhooks/      # Stripe webhook handlers
│   └── stripe/        # Stripe API integration
├── common/            # Guards, filters, decorators
└── config/            # Database, Stripe config
```

**Key Features**:

- Credit account management
- Subscription billing (Free/Pro/Enterprise)
- Stripe Checkout integration
- Webhook event processing
- Payment history and invoices
- Feature usage metering (AI, storage)

## Communication Patterns

### Synchronous Communication

```
┌─────────────────┐    HTTP/REST    ┌─────────────────┐
│ Profile Service │◄───────────────►│Identity Service │
│                 │    (OpenFeign)  │                 │
└─────────────────┘                 └─────────────────┘
```

**Use Cases**:

- Token validation
- User authentication checks
- Real-time data fetching

### Asynchronous Communication

```
┌─────────────────┐                 ┌─────────────────┐
│ Profile Service │──── publish ───►│    RabbitMQ     │
└─────────────────┘                 └────────┬────────┘
                                             │
                                    consume  │
                                             ▼
                                    ┌─────────────────┐
                                    │ Roadmap Service │
                                    └─────────────────┘
```

**Use Cases**:

- User created event
- Profile updated event
- Email notifications

## Data Flow Examples

### User Registration Flow

```
1. Client ──POST /register──► Profile Service
2. Profile Service ──validate──► Identity Service
3. Profile Service ──save user──► MySQL
4. Profile Service ──publish event──► RabbitMQ
5. RabbitMQ ──notify──► Other Services
6. Profile Service ──response──► Client
```

### AI Roadmap Generation Flow

```
1. Client ──POST /generate──► Roadmap AI Service
2. AI Service ──call──► OpenAI API
3. AI Service ──parse response──► Roadmap structure
4. AI Service ──store vectors──► MongoDB
5. AI Service ──response──► Client
6. Client ──POST /roadmap──► Roadmap Service
7. Roadmap Service ──save──► MySQL
```

## Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Filter Chain                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              JWT Authentication Filter                 │  │
│  │  • Extract token from cookie/header                   │  │
│  │  • Validate signature with public key                 │  │
│  │  • Set SecurityContext                                │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                 │
│                            ▼                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Authorization Check                       │  │
│  │  • @PreAuthorize annotations                          │  │
│  │  • Role-based: STUDENT, TEACHER, ADMIN                │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Token Flow

```
1. Login ──credentials──► Identity Service
2. Identity Service ──JWT tokens──► Client (HTTP-only cookies)
3. Client ──request + cookies──► Any Service
4. Service ──validate token──► Process request
5. Token expired? ──refresh token──► Identity Service
```

## Scalability Patterns

### Horizontal Scaling

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Profile Service │ │ Profile Service │ │ Profile Service │
│   Instance 1    │ │   Instance 2    │ │   Instance 3    │
└─────────────────┘ └─────────────────┘ └─────────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
                    ┌─────────────────┐
                    │  Shared Redis   │
                    │  (Session/Cache)│
                    └─────────────────┘
```

### Database Scaling

- **Read replicas** for heavy read workloads
- **Sharding** for large datasets
- **Caching** with Redis for frequently accessed data

## Monitoring & Observability

### Health Checks

Each service exposes actuator endpoints:

```
GET /actuator/health       # Overall health
GET /actuator/health/liveness   # Kubernetes liveness
GET /actuator/health/readiness  # Kubernetes readiness
GET /actuator/metrics      # Prometheus metrics
```

### Logging

- Structured JSON logging
- Correlation IDs for request tracing
- Centralized log aggregation (ELK/Loki)

## Deployment Architecture

### Docker Compose (Development)

```yaml
services:
  profile-service:
    build: ./profile-service
    ports: ['8082:8082']
  roadmap-service:
    build: ./roadmap-service
    ports: ['8083:8083']
  # ... other services
  redis:
    image: redis:6
  rabbitmq:
    image: rabbitmq:management
```

### Kubernetes (Production)

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Ingress Controller                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Deployment  │  │ Deployment  │  │ Deployment  │         │
│  │  profile    │  │  roadmap    │  │    ai       │         │
│  │  replicas:3 │  │  replicas:2 │  │  replicas:2 │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                            │                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  StatefulSets                          │  │
│  │    MySQL │ Redis │ RabbitMQ │ MongoDB │ MinIO         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```
