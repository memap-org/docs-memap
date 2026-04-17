# Roadmap AI Service — Core Features

**Service**: `roadmap-ai-service`  
**Port**: `8085` | **Context path**: `/`  
**LLM**: Groq `llama-3.1-8b-instant` | **Embeddings**: OpenAI `text-embedding-3-small`  
**Vector store**: Pinecone | **Database**: MongoDB  
**Response language**: Vietnamese (tiếng Việt)

---

## 1. AI Roadmap Generation

Automatically generates a complete learning roadmap (nodes + edges) from a natural-language topic.

### Request

```http
POST /ai/roadmap/generate
Authorization: Bearer <token>

{
  "topic":       "Machine Learning",
  "description": "Practical ML for software engineers",   // optional
  "difficulty":  "INTERMEDIATE",                          // BEGINNER | INTERMEDIATE | ADVANCED
  "maxNodes":    15                                       // default: 15
}
```

### Response

Returns a structured JSON object with:

- `nodes[]` — array of node objects (title, description, position)
- `edges[]` — array of directed edges (source, target)

**The response is NOT automatically saved to `roadmap-service`.** The client must call `POST /roadmap/road-map/api/roadmap` separately.

### Under the Hood

```
Client → roadmap-ai-service
  → Spring AI prompt template (Vietnamese system prompt)
  → Groq API (llama-3.1-8b-instant)
  → Parse JSON response
  → Return structured roadmap
```

---

## 2. Node Content Summarization

Summarizes a node's rich-text content into concise bullet points.

```http
POST /ai/node/summarize
Authorization: Bearer <token>

{
  "nodeId":    "abc123",
  "roadmapId": "xyz789"
}
```

**Flow:**

1. Fetches node content from `roadmap-service` via REST (OpenFeign)
2. Sends content + prompt to Groq
3. Returns bullet-point summary in Vietnamese

---

## 3. Quiz Generation

Generates multiple-choice quiz questions from a topic or node content.

```http
POST /ai/quiz/generate
Authorization: Bearer <token>

{
  "topic":           "Docker & Containerization",
  "questionCount":   5,
  "difficulty":      "BEGINNER"
}
```

**Returns:**

```json
{
  "questions": [
    {
      "question": "What is a Docker container?",
      "options": ["A", "B", "C", "D"],
      "correctAnswer": "A",
      "explanation": "..."
    }
  ]
}
```

---

## 4. Node Q&A Chat (RAG)

Context-aware question answering about node content, using Retrieval-Augmented Generation (RAG).

### Start / Continue Chat

```http
POST /ai/chat
Authorization: Bearer <token>

{
  "roadmapId": "xyz789",
  "nodeId":    "abc123",
  "message":   "Explain this concept in simpler terms",
  "sessionId": "optional-session-id"   // omit to start new session
}
```

### RAG Pipeline

```
User message
  ↓
1. Embed query (OpenAI text-embedding-3-small)
  ↓
2. Similarity search in Pinecone (index: roadmap-ai)
  ↓
3. Retrieve top-k relevant chunks
  ↓
4. Load chat history from MongoDB (by sessionId)
  ↓
5. Build prompt: [system] + [chat history] + [context chunks] + [user message]
  ↓
6. Call Groq (llama-3.1-8b-instant)
  ↓
7. Save turn to MongoDB chat history
  ↓
8. Return response (Vietnamese)
```

### Chat Session Management

```http
GET  /ai/chat/sessions                     List user's chat sessions
GET  /ai/chat/sessions/{sessionId}         Get session history
DELETE /ai/chat/sessions/{sessionId}       Clear session
```

---

## 5. Document Ingestion (Embedding)

Node content is embedded and stored in Pinecone for RAG retrieval.

**Trigger:** Automatically called when a node's content is created or updated (via REST or event).

**Process:**

1. Chunk node content (by paragraph / sentence)
2. Embed each chunk via OpenAI `text-embedding-3-small`
3. Upsert vectors to Pinecone with metadata: `nodeId`, `roadmapId`, `userId`
4. Also store metadata in MongoDB for audit

```http
POST /ai/embedding/ingest
Authorization: Bearer <token>

{
  "nodeId":    "abc123",
  "roadmapId": "xyz789",
  "content":   "Rich text content of the node..."
}
```

---

## 6. Rate Limits

| API                 | Limit                      |
| ------------------- | -------------------------- |
| Groq (LLM)          | 30 req/min, 14 400 req/day |
| OpenAI (embeddings) | 3 000 req/min              |
| Pinecone            | Plan-dependent             |

Implement client-side retry with exponential backoff when hitting Groq rate limits.

---

## 7. Configuration Reference

Key `application.yaml` settings:

```yaml
spring:
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
      uri: ${MONGODB_URL}
```

---

## 8. Swagger Access

`http://localhost:8085/swagger-ui.html`
