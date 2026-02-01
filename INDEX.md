# MeMap Platform Documentation Index

Welcome to the MeMap platform documentation! This index helps you navigate to the right documentation for each service.

## 📚 Documentation Structure

```
docs/
├── README.md                    # Main documentation (you are here)
├── API.md                       # API overview with links to service APIs
├── ARCHITECTURE.md              # System architecture overview
├── DEPLOYMENT.md                # Deployment guides
├── ERROR_CODES.md               # Error code reference
├── GETTING_STARTED.md           # Quick start guide
│
├── identity-service/            # Authentication & Authorization Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── profile-service/             # User Profile Management Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── roadmap-service/             # Roadmap Management Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── roadmap-ai-service/          # AI-Powered Features Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── file-service/                # File Storage Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── learning-service/            # Learning & Quiz Management Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
├── payment-service/             # Payment & Credit Management Service
│   ├── README.md                # Service overview
│   ├── API.md                   # API endpoints & examples
│   └── ARCHITECTURE.md          # Service architecture
│
└── memap-frontend/              # React Frontend Application
    ├── README.md                # Application overview
    ├── API.md                   # Frontend API integration
    └── ARCHITECTURE.md          # Frontend architecture
```

## 🚀 Quick Start Guides

| Document                | Description                  | Link                                       |
| ----------------------- | ---------------------------- | ------------------------------------------ |
| **Getting Started**     | Set up the entire platform   | [GETTING_STARTED.md](./GETTING_STARTED.md) |
| **Deployment Guide**    | Deploy to production         | [DEPLOYMENT.md](./DEPLOYMENT.md)           |
| **System Architecture** | Understand the system design | [ARCHITECTURE.md](./ARCHITECTURE.md)       |
| **API Overview**        | Browse all service APIs      | [API.md](./API.md)                         |

## 🔧 Backend Services

### Identity Service (Port 8081)

Authentication and authorization service managing JWT tokens, user credentials, and OAuth2 integration.

- **[README](./identity-service/README.md)** - Overview, features, getting started
- **[API Documentation](./identity-service/API.md)** - Login, logout, token management
- **[Architecture](./identity-service/ARCHITECTURE.md)** - Design, security, patterns

### Profile Service (Port 8082)

User profile management with avatar uploads, password management, and user search.

- **[README](./profile-service/README.md)** - Overview, features, getting started
- **[API Documentation](./profile-service/API.md)** - Registration, profile updates, avatar uploads
- **[Architecture](./profile-service/ARCHITECTURE.md)** - Caching, messaging, integrations

### Roadmap Service (Port 8083)

Roadmap creation and management with nodes, edges, permissions, and progress tracking.

- **[README](./roadmap-service/README.md)** - Overview, features, getting started
- **[API Documentation](./roadmap-service/API.md)** - CRUD operations, nodes, edges, sharing
- **[Architecture](./roadmap-service/ARCHITECTURE.md)** - Data models, permissions, caching

### Roadmap AI Service (Port 8084)

AI-powered roadmap generation, node summaries, quiz generation, and Q&A chat.

- **[README](./roadmap-ai-service/README.md)** - Overview, AI features, getting started
- **[API Documentation](./roadmap-ai-service/API.md)** - Generation, summaries, chat
- **[Architecture](./roadmap-ai-service/ARCHITECTURE.md)** - RAG, LLM integration, vector store

### File Service (Port 8085)

File upload, download, and storage using MinIO object storage.

- **[README](./file-service/README.md)** - Overview, features, getting started
- **[API Documentation](./file-service/API.md)** - Upload, download, presigned URLs
- **[Architecture](./file-service/ARCHITECTURE.md)** - MinIO integration, metadata storage

### Learning Service (Port 8086)

Quiz and learning management system (Work in Progress).

- **[README](./learning-service/README.md)** - Overview, features, getting started
- **[API Documentation](./learning-service/API.md)** - Quiz CRUD, questions, visibility
- **[Architecture](./learning-service/ARCHITECTURE.md)** - Data models, security

### Payment Service (Port 8087)

Payment processing and credit management with Stripe integration.

- **[README](./payment-service/README.md)** - Overview, credit system, pricing
- **[API Documentation](./payment-service/API.md)** - Credits, subscriptions, payments
- **[Architecture](./payment-service/ARCHITECTURE.md)** - Stripe integration, NestJS patterns

## 🎨 Frontend Application

### MeMap Frontend (Port 5173)

React + TypeScript web application with Redux state management and protected routes.

- **[README](./memap-frontend/README.md)** - Overview, features, getting started
- **[API Integration](./memap-frontend/API.md)** - RTK Query, authentication flow
- **[Architecture](./memap-frontend/ARCHITECTURE.md)** - Components, routing, state management

## 📖 Reference Documentation

### Error Codes

Complete reference of error codes across all services.

- **[ERROR_CODES.md](./ERROR_CODES.md)**

### Deployment

Production deployment guides for Docker, Kubernetes, and cloud platforms.

- **[DEPLOYMENT.md](./DEPLOYMENT.md)**

## 🔍 Quick Search

### By Technology

- **Spring Boot**: Identity, Profile, Roadmap, File, Learning, AI services
- **NestJS**: Payment service
- **MySQL**: Identity, Profile, Roadmap, File, Learning services
- **PostgreSQL**: Payment service
- **MongoDB**: Roadmap AI service (vectors, chat history)
- **Redis**: Profile, Roadmap services (caching)
- **RabbitMQ**: Profile, Roadmap services (messaging)
- **MinIO**: File service (object storage)
- **Stripe**: Payment service (payment processing)
- **React**: Frontend application
- **AI/LLM**: Roadmap AI service

### By Feature

- **Authentication**: [Identity Service](./identity-service/)
- **User Management**: [Profile Service](./profile-service/)
- **Roadmap Creation**: [Roadmap Service](./roadmap-service/)
- **AI Generation**: [Roadmap AI Service](./roadmap-ai-service/)
- **File Upload**: [File Service](./file-service/)
- **Quiz Management**: [Learning Service](./learning-service/)
- **Payments & Credits**: [Payment Service](./payment-service/)
- **Web Interface**: [Frontend](./memap-frontend/)

## 🛠️ Development Workflow

1. **Start with** [GETTING_STARTED.md](./GETTING_STARTED.md) to set up your environment
2. **Review** [ARCHITECTURE.md](./ARCHITECTURE.md) to understand the system
3. **Explore** service-specific documentation for the service you're working on
4. **Reference** [API.md](./API.md) for endpoint details
5. **Check** [ERROR_CODES.md](./ERROR_CODES.md) when troubleshooting

## 📝 Documentation Conventions

Each service documentation follows this structure:

### README.md

- Service overview and purpose
- Key features
- Technology stack
- Port and URL configuration
- Getting started guide
- Related services

### API.md

- Base URLs
- Authentication requirements
- All endpoints with examples
- Request/response formats
- Error responses
- Data models

### ARCHITECTURE.md

- Architecture diagrams
- Component details
- Data models
- Design patterns
- Security considerations
- Performance optimizations

## 🔄 Last Updated

**January 29, 2026**

This documentation structure was created to provide comprehensive, service-specific documentation based on the current codebase implementation.

## 📞 Support

For questions or issues with the documentation:

- Review the service-specific docs first
- Check the troubleshooting sections
- Contact the development team

---

**Navigation**: [Main README](./README.md) | [API Overview](./API.md) | [Architecture](./ARCHITECTURE.md)
