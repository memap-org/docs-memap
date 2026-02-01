# Roadmap AI Service

## Overview

The Roadmap AI Service provides AI-powered features for the MeMap platform, including automatic roadmap generation, node content summarization, quiz generation, and Q&A chat functionality. It leverages OpenAI/LLM models and vector embeddings for intelligent content creation.

## Key Features

- **AI Roadmap Generation**: Automatically generate complete learning roadmaps from a topic
- **Node Summarization**: Summarize node content into concise bullet points using AI
- **Quiz Generation**: Generate multiple-choice quiz questions from topics
- **Node Q&A**: Ask questions about specific node content and get AI answers
- **Vector Embeddings**: Store and query embeddings for semantic search
- **Chat History**: Maintain conversation context for better responses

## Technology Stack

- **Framework**: Spring Boot 3.x
- **Language**: Java 21
- **AI**: Spring AI with OpenAI/Groq LLM
- **Embeddings**: OpenAI text-embedding-3-small
- **Vector Database**: Pinecone
- **Database**: MongoDB (chat history, metadata)
- **Build Tool**: Maven

## Port Configuration

- **Service Port**: `8085`
- **Base URL**: `http://localhost:8085`
- **Context Path**: `/`

## AI Integration

### LLM Provider: Groq (llama-3.1-8b-instant)

- **Endpoint**: `https://api.groq.com/openai/v1/chat/completions`
- **Model**: llama-3.1-8b-instant
- **Use**: Chat completions, content generation

### Embeddings: OpenAI

- **Model**: text-embedding-3-small
- **Dimensions**: 1536
- **Use**: Vector embeddings for semantic search

### Vector Store: Pinecone

- **Index**: roadmap-ai
- **Namespace**: default (configurable)
- **Use**: Store and query document embeddings

## Database

MongoDB is used for storing:

- Chat sessions and history
- User data
- Roadmap metadata (cache)
- Conversation context

## Getting Started

### Prerequisites

- JDK 21 or higher
- Maven 3.8+
- MongoDB
- Groq API Key
- OpenAI API Key (for embeddings)
- Pinecone API Key

### Running the Service

```bash
# Navigate to service directory
cd roadmap-ai-service

# Run with Maven
./mvnw spring-boot:run

# Or run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Configuration

Key configuration properties (application.yaml):

```yaml
server:
  port: 8085

spring:
  application:
    name: spring-ai-roadmap
  ai:
    openai:
      api-key: ${GROQ_KEY}
      chat:
        base-url: https://api.groq.com/openai
        options:
          model: llama-3.1-8b-instant
      embedding:
        api-key: ${OPENAI_API_KEY}
        options:
          model: text-embedding-3-small
    vectorstore:
      pinecone:
        api-key: ${PINECONE_API_KEY}
        index-name: roadmap-ai
  data:
    mongodb:
      uri: ${MONGODB_URL:mongodb://localhost:27017/roadmap_ai}

embedding:
  service:
    url: ${EMBEDDING_SERVICE_URL:http://localhost:8000}
```

### Environment Variables

- `GROQ_KEY`: Groq API key for LLM
- `OPENAI_API_KEY`: OpenAI API key for embeddings
- `PINECONE_API_KEY`: Pinecone API key for vector store
- `PINECONE_INDEX_NAME`: Pinecone index name (default: roadmap-ai)
- `MONGODB_URL`: MongoDB connection string

## API Documentation

For detailed API endpoints, request/response formats, and examples, see [API.md](./API.md).

## Architecture

For architectural details, design patterns, and component interactions, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Related Services

- **Roadmap Service**: Fetches roadmap and node data for AI processing
- **Learning Service**: Uses AI-generated quizzes
- **Identity Service**: JWT token validation

## AI Features

### 1. Roadmap Generation

Generate a complete learning roadmap with nodes and edges based on:

- Topic name
- Description (optional)
- Difficulty level (BEGINNER, INTERMEDIATE, ADVANCED)
- Maximum nodes (default: 15)

### 2. Node Summarization

- Fetch node content from Roadmap Service
- Generate concise bullet-point summary
- Return structured summary with key points

### 3. Quiz Generation

- Generate multiple-choice questions from a topic
- Configurable number of questions
- Include correct answers and explanations

### 4. Node Q&A

- Ask questions about specific node content
- Context-aware responses using RAG (Retrieval-Augmented Generation)
- Chat history maintained for follow-up questions

## Swagger Documentation

Access Swagger UI at: `http://localhost:8085/swagger-ui.html`

API docs available at: `http://localhost:8085/v3/api-docs`

## Response Language

All AI-generated content is in Vietnamese (tiếng Việt) by default, as configured in the system prompts.

## Rate Limits

- Groq API: 30 requests/minute, 14400 requests/day
- OpenAI Embeddings: 3000 requests/minute
- Pinecone: Based on plan tier

## Error Handling

The service provides detailed error messages for:

- Invalid requests
- AI service failures
- API rate limits
- Missing roadmap/node data
- Database connection issues

## Monitoring

- Spring Boot Actuator enabled
- Health check: `http://localhost:8085/actuator/health`
- Custom metrics for AI requests
- Logging with SLF4J
