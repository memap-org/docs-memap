# Roadmap AI Service Architecture

## Service Overview

The Roadmap AI Service is an AI-powered microservice that provides intelligent content generation and processing capabilities for the MeMap platform. It leverages Large Language Models (LLMs), embeddings, and vector databases to generate roadmaps, summarize content, create quizzes, and answer questions.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Layer                              │
│              (Frontend, Roadmap Service, Learning Service)          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                      Controller Layer                               │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ RoadmapGenerator │  │  NodeAICtrl     │  │  ChatController  │  │
│  │   Controller     │  │ - Summarize     │  │ - Quiz generation│  │
│  │ - Generate       │  │ - Q&A chat      │  │                  │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                        Service Layer                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RoadmapGeneratorService                                      │  │
│  │ - Build prompts for roadmap generation                       │  │
│  │ - Parse AI responses                                         │  │
│  │ - Assign node positions                                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ NodeSummaryService                                           │  │
│  │ - Fetch node from Roadmap Service                            │  │
│  │ - Generate summarization prompts                             │  │
│  │ - Extract key points                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ChatService                                                  │  │
│  │ - Generate quiz questions                                    │  │
│  │ - Format questions and answers                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ChatNodeService                                              │  │
│  │ - Q&A on node content                                        │  │
│  │ - Context-aware responses                                    │  │
│  │ - RAG implementation                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ DocumentIngestService                                        │  │
│  │ - Ingest documents                                           │  │
│  │ - Create embeddings                                          │  │
│  │ - Store in vector DB                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                      AI Integration Layer                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Spring AI ChatClient                                         │  │
│  │ - ChatClient.Builder                                         │  │
│  │ - Prompt construction                                        │  │
│  │ - Response parsing                                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ EmbeddingModel                                               │  │
│  │ - OpenAI text-embedding-3-small                              │  │
│  │ - Generate vector embeddings                                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ VectorStore (Pinecone)                                       │  │
│  │ - Store document embeddings                                  │  │
│  │ - Similarity search                                          │  │
│  │ - RAG retrieval                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                    Repository Layer                                 │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ UserRepository   │  │ ChatSession     │  │ RoadMapRepo      │  │
│  │                  │  │  Repository     │  │ (MongoDB)        │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────┐
│                        Data Layer                                   │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐   │
│  │ MongoDB                 │    │ Pinecone Vector DB           │   │
│  │ ┌─────────────────────┐ │    │ ┌──────────────────────────┐ │   │
│  │ │ chat_sessions       │ │    │ │ Index: roadmap-ai        │ │   │
│  │ │ users               │ │    │ │ Dimensions: 1536         │ │   │
│  │ │ roadmaps (cache)    │ │    │ │ Metric: cosine           │ │   │
│  │ └─────────────────────┘ │    │ │                          │ │   │
│  └─────────────────────────┘    │ │ Vectors:                 │ │   │
│                                  │ │ - Node content           │ │   │
│                                  │ │ - Resources              │ │   │
│                                  │ └──────────────────────────┘ │   │
│                                  └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────┐
│                    External Services                                │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ Groq API         │  │ OpenAI API      │  │ Roadmap Service  │  │
│  │ - LLM inference  │  │ - Embeddings    │  │ - Node data      │  │
│  │ - llama-3.1      │  │ - text-embed... │  │                  │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### RoadmapGeneratorController

**Responsibility**: Handle roadmap generation requests

**Key Methods**:

```java
@PostMapping("/generate")
public ResponseEntity<GenerateRoadmapResponse> generateRoadmap(
    @RequestBody GenerateRoadmapRequest request);

@GetMapping("/generate")
public ResponseEntity<GenerateRoadmapResponse> generateRoadmapByTopic(
    @RequestParam String topic,
    @RequestParam(required = false, defaultValue = "15") Integer maxNodes,
    @RequestParam(required = false, defaultValue = "INTERMEDIATE") String difficulty);
```

#### NodeAIController

**Responsibility**: Handle node-level AI operations

**Key Methods**:

```java
@PostMapping("/summary")
public ResponseEntity<NodeSummaryResponse> summarizeNode(
    @RequestBody NodeSummaryRequest request);

@GetMapping("/summary/{roadmapId}/{nodeId}")
public ResponseEntity<NodeSummaryResponse> summarizeNodeByPath(
    @PathVariable String roadmapId,
    @PathVariable String nodeId);

@PostMapping("/ask")
public ResponseEntity<ChatNodeResponse> chatNodeContent(
    @RequestBody ChatNodeRequest request);
```

#### ChatController

**Responsibility**: Generate quiz questions

**Key Methods**:

```java
@PostMapping
public QuestionListResponseDto getQuestionListForRoadmap(
    @RequestBody ChatRoadmapRequest chatRoadmapRequest);
```

### 2. Service Layer

#### RoadmapGeneratorService

**Core Logic**:

```java
public GenerateRoadmapResponse generateRoadmap(GenerateRoadmapRequest request) {
    // 1. Build system prompt
    SystemMessage systemMessage = new SystemMessage(buildSystemPrompt());

    // 2. Build user prompt with topic, description, difficulty
    UserMessage userMessage = new UserMessage(
        buildUserPrompt(request.getTopic(), request.getDescription(),
                       request.getMaxNodes(), request.getDifficulty())
    );

    // 3. Create prompt
    Prompt prompt = new Prompt(systemMessage, userMessage);

    // 4. Call LLM
    GenerateRoadmapResponse response = chatClient
        .prompt(prompt)
        .call()
        .entity(GenerateRoadmapResponse.class);

    // 5. Assign node positions
    assignNodePositions(response);

    return response;
}
```

**System Prompt**:

```
Bạn là Memap.AI, một chuyên gia thiết kế lộ trình học tập (roadmap).
Nhiệm vụ của bạn là tạo ra cấu trúc roadmap dạng JSON để giúp người học có lộ trình rõ ràng.

Output của bạn PHẢI là một JSON object hợp lệ với cấu trúc sau:
{
  "name": "Tên roadmap",
  "description": "Mô tả ngắn gọn về roadmap",
  "nodes": [...],
  "edges": [...]
}

QUY TẮC:
1. nodeType chỉ có 2 giá trị: "TOPIC_NODE" hoặc "NO_TOPIC_NODE"
2. nodeId phải unique và theo format: node_1, node_2, ...
3. Roadmap phải có luồng logic từ cơ bản đến nâng cao
4. CHỈ trả về JSON, KHÔNG có text giải thích
5. Tất cả nội dung phải bằng tiếng Việt
```

**Node Positioning Algorithm**:

```java
private void assignNodePositions(GenerateRoadmapResponse response) {
    int x = 100;
    int y = 100;
    int xOffset = 300;
    int yOffset = 150;

    for (int i = 0; i < response.getNodes().size(); i++) {
        Node node = response.getNodes().get(i);

        // Arrange in rows of 3
        int col = i % 3;
        int row = i / 3;

        node.setPosition(new Position(
            x + (col * xOffset),
            y + (row * yOffset)
        ));
    }
}
```

#### NodeSummaryService

**Core Logic**:

```java
public NodeSummaryResponse summarizeNodeContent(NodeSummaryRequest request) {
    // 1. Fetch roadmap from Roadmap Service
    RoadMap roadmap = fetchRoadmapFromService(request.getRoadmapId());

    // 2. Find specific node
    Node node = roadmap.getNodes().stream()
        .filter(n -> n.getNodeId().equals(request.getNodeId()))
        .findFirst()
        .orElseThrow(() -> new NodeNotFoundException());

    // 3. Build summarization prompt
    String prompt = buildSummarizationPrompt(node);

    // 4. Call LLM for summary
    NodeSummaryResponse response = chatClient
        .prompt(prompt)
        .call()
        .entity(NodeSummaryResponse.class);

    // 5. Add metadata
    response.setRoadmapId(request.getRoadmapId());
    response.setNodeId(request.getNodeId());
    response.setNodeLabel(node.getLabel());

    return response;
}
```

**Summarization Prompt Template**:

```
Tóm tắt nội dung sau thành các bullet points ngắn gọn và dễ hiểu:

Tiêu đề: {nodeLabel}
Nội dung: {nodeContent}

Hãy trả về JSON với format:
{
  "mainPoints": ["điểm chính 1", "điểm chính 2", ...],
  "keyTakeaways": ["điểm quan trọng 1", "điểm quan trọng 2", ...]
}
```

#### ChatNodeService (RAG Implementation)

**Core Logic**:

```java
public ChatNodeResponse chatNodeContent(ChatNodeRequest request) {
    // 1. Fetch node content
    Node node = fetchNodeFromRoadmap(request.getRoadmapId(), request.getNodeId());

    // 2. Create embedding for question
    List<Double> questionEmbedding = embeddingModel.embed(request.getQuestion());

    // 3. Retrieve similar documents from vector store
    List<Document> similarDocs = vectorStore.similaritySearch(
        SearchRequest.builder()
            .query(request.getQuestion())
            .topK(5)
            .build()
    );

    // 4. Build context from node content + similar docs
    String context = buildContext(node, similarDocs);

    // 5. Build RAG prompt
    String prompt = buildRAGPrompt(request.getQuestion(), context);

    // 6. Get answer from LLM
    String answer = chatClient
        .prompt(prompt)
        .call()
        .content();

    return ChatNodeResponse.builder()
        .question(request.getQuestion())
        .answer(answer)
        .context("Based on node content and related materials")
        .build();
}
```

**RAG Prompt Template**:

```
Bạn là một trợ lý thông minh. Hãy trả lời câu hỏi dựa trên context sau:

Context:
{context}

Câu hỏi: {question}

Hãy trả lời một cách chi tiết, dễ hiểu, bằng tiếng Việt.
```

#### ChatService (Quiz Generation)

**Core Logic**:

```java
public QuestionListResponseDto getQuestionList(ChatRoadmapRequest request) {
    // 1. Build quiz generation prompt
    String prompt = buildQuizPrompt(
        request.getTitle(),
        request.getNumberOfQuestions() != null ? request.getNumberOfQuestions() : 5
    );

    // 2. Call LLM
    QuestionListResponseDto response = chatClient
        .prompt(prompt)
        .call()
        .entity(QuestionListResponseDto.class);

    // 3. Validate and format questions
    validateQuizQuestions(response);

    return response;
}
```

**Quiz Generation Prompt**:

```
Tạo {numberOfQuestions} câu hỏi trắc nghiệm về chủ đề: {title}

Format JSON:
{
  "title": "...",
  "totalQuestions": ...,
  "questions": [
    {
      "questionId": "q1",
      "questionText": "Câu hỏi...",
      "options": [
        {"optionId": "A", "text": "Đáp án A"},
        {"optionId": "B", "text": "Đáp án B"},
        {"optionId": "C", "text": "Đáp án C"},
        {"optionId": "D", "text": "Đáp án D"}
      ],
      "correctAnswer": "A",
      "explanation": "Giải thích..."
    }
  ]
}

Yêu cầu:
- Câu hỏi phải rõ ràng, chính xác
- 4 đáp án cho mỗi câu
- Có explanation cho đáp án đúng
- Độ khó từ dễ đến trung bình
```

#### DocumentIngestService

**Core Logic**:

```java
@Service
public class DocumentIngestService {
    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public void ingestDocument(String content, Map<String, Object> metadata) {
        // 1. Create embedding
        List<Double> embedding = embeddingModel.embed(content);

        // 2. Create document
        Document document = Document.builder()
            .content(content)
            .metadata(metadata)
            .embedding(embedding)
            .build();

        // 3. Store in Pinecone
        vectorStore.add(List.of(document));
    }

    public void ingestRoadmap(RoadMap roadmap) {
        for (Node node : roadmap.getNodes()) {
            String content = String.format(
                "Title: %s\nContent: %s",
                node.getLabel(),
                node.getContent()
            );

            Map<String, Object> metadata = Map.of(
                "roadmapId", roadmap.getId(),
                "nodeId", node.getNodeId(),
                "type", "node"
            );

            ingestDocument(content, metadata);
        }
    }
}
```

### 3. Data Models

#### ChatSession (MongoDB)

```java
@Document(collection = "chat_sessions")
@Data
public class ChatSession extends BaseEntity {
    @Id
    private String id;

    private String userId;
    private String roadmapId;
    private String nodeId;

    private List<ChatMessage> messages;

    private LocalDateTime startedAt;
    private LocalDateTime lastMessageAt;
}

@Data
class ChatMessage {
    private String role; // "user" or "assistant"
    private String content;
    private LocalDateTime timestamp;
}
```

#### RoadMap (MongoDB Cache)

```java
@Document(collection = "RoadMap")
@Data
public class RoadMap extends BaseDocument {
    @Id
    private String id;

    private String title;
    private String description;
    private String scope;
    private String ownerId;

    private List<Node> nodes;
    private List<Edge> edges;

    private LocalDateTime cachedAt;
    private Long ttl; // Time to live in seconds
}
```

## Design Patterns

### 1. Builder Pattern

- Used extensively with Spring AI ChatClient
- Fluent API for constructing prompts and requests

```java
ChatClient chatClient = chatClientBuilder
    .defaultSystem("You are a helpful assistant")
    .build();

Prompt prompt = new Prompt.Builder()
    .systemMessage(systemMessage)
    .userMessage(userMessage)
    .build();
```

### 2. Strategy Pattern

- Different prompt strategies for different AI tasks
- Pluggable AI models (Groq, OpenAI, etc.)

### 3. Template Method Pattern

- Base prompt templates with customizable sections
- Reusable prompt structure

### 4. Repository Pattern

- MongoDB repositories for data access
- Clean separation of concerns

### 5. DTO Pattern

- Request/Response DTOs for API contracts
- Entity DTOs for database models

## AI Integration Architecture

### Spring AI Framework

```
┌─────────────────────────────────────────────────────────────┐
│                      Spring AI                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ ChatClient  │  │ Embedding   │  │ VectorStore         │ │
│  │             │  │  Model      │  │ (Pinecone)          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│         │                │                     │            │
│         ▼                ▼                     ▼            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Groq API    │  │ OpenAI API  │  │ Pinecone API        │ │
│  │ (LLM)       │  │ (Embeddings)│  │ (Vector Search)     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Configuration

```java
@Configuration
public class AIConfig {

    @Bean
    public ChatClient.Builder chatClientBuilder(ChatModel chatModel) {
        return ChatClient.builder(chatModel);
    }

    @Bean
    public VectorStore vectorStore(
        PineconeVectorStoreProperties properties,
        EmbeddingModel embeddingModel) {
        return new PineconeVectorStore(properties, embeddingModel);
    }
}
```

## RAG (Retrieval-Augmented Generation) Flow

```
User Question
     │
     ▼
[Create Embedding]
     │
     ▼
[Vector Search in Pinecone]
     │
     ▼
[Retrieve Top K Similar Documents]
     │
     ▼
[Build Context: Node Content + Similar Docs]
     │
     ▼
[Create Prompt with Context + Question]
     │
     ▼
[LLM generates Answer based on Context]
     │
     ▼
Response to User
```

## Performance Considerations

### Caching Strategy

```java
@Cacheable(value = "roadmaps", key = "#roadmapId")
public RoadMap fetchRoadmap(String roadmapId) {
    // Fetch from Roadmap Service
    // Cache in MongoDB for faster subsequent access
}
```

### Async Processing

```java
@Async
public CompletableFuture<GenerateRoadmapResponse> generateRoadmapAsync(
    GenerateRoadmapRequest request) {
    return CompletableFuture.completedFuture(generateRoadmap(request));
}
```

### Connection Pooling

- MongoDB connection pooling
- HTTP client connection pooling for external APIs

## Error Handling

### Custom Exceptions

```java
public class AIServiceException extends RuntimeException {
    public AIServiceException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class NodeNotFoundException extends RuntimeException {
    public NodeNotFoundException(String nodeId) {
        super("Node not found: " + nodeId);
    }
}

public class RateLimitExceededException extends RuntimeException {
    private final int retryAfterSeconds;

    public RateLimitExceededException(int retryAfterSeconds) {
        super("Rate limit exceeded. Retry after " + retryAfterSeconds + " seconds");
        this.retryAfterSeconds = retryAfterSeconds;
    }
}
```

### Global Exception Handler

```java
@ControllerAdvice
public class AIExceptionHandler {

    @ExceptionHandler(AIServiceException.class)
    public ResponseEntity<ErrorResponse> handleAIServiceException(AIServiceException ex) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("AI service error", ex.getMessage()));
    }

    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<ErrorResponse> handleRateLimitExceeded(
        RateLimitExceededException ex) {
        return ResponseEntity
            .status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", String.valueOf(ex.getRetryAfterSeconds()))
            .body(new ErrorResponse("Rate limit exceeded", ex.getMessage()));
    }
}
```

## Monitoring & Observability

### Logging

```java
@Slf4j
@Service
public class RoadmapGeneratorService {
    public GenerateRoadmapResponse generateRoadmap(GenerateRoadmapRequest request) {
        log.info("Generating roadmap for topic: {}", request.getTopic());

        Instant start = Instant.now();
        try {
            GenerateRoadmapResponse response = // ... generation logic

            long duration = Duration.between(start, Instant.now()).toMillis();
            log.info("Roadmap generated in {}ms", duration);

            return response;
        } catch (Exception e) {
            log.error("Error generating roadmap for topic: {}", request.getTopic(), e);
            throw new AIServiceException("Failed to generate roadmap", e);
        }
    }
}
```

### Metrics

- AI request count
- Average response time
- Token usage
- Error rate
- Cache hit rate

## Security

### API Key Management

- Environment variables for API keys
- Never commit keys to source control
- Rotation policy for keys

### Rate Limiting

- Implement rate limiting to prevent abuse
- Track API usage per user
- Graceful degradation when limits exceeded

## Cost Optimization

### Token Management

```java
private String optimizePrompt(String prompt) {
    // Remove unnecessary whitespace
    // Truncate overly long content
    // Use concise language
    return prompt.trim().replaceAll("\\s+", " ");
}
```

### Caching

- Cache AI responses for identical requests
- TTL-based cache expiration
- LRU eviction policy

### Model Selection

- Use cheaper models for simple tasks
- Reserve powerful models for complex generation

## Future Enhancements

1. **Streaming Responses**: Implement SSE for real-time streaming
2. **Multi-model Support**: Allow switching between different LLM providers
3. **Fine-tuning**: Fine-tune models on domain-specific data
4. **Advanced RAG**: Implement hybrid search (vector + keyword)
5. **Conversation Memory**: Maintain long-term conversation history
6. **Image Understanding**: Support multimodal inputs
7. **Code Generation**: Generate code snippets for programming roadmaps
8. **Personalization**: Tailor responses based on user profile and history
