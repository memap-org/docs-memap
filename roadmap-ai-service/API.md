# Roadmap AI Service API Documentation

## Base URL

```
http://localhost:8085
```

## Authentication

Authentication is optional for AI service endpoints, but recommended for production use. If authentication is required, include JWT token:

```
Authorization: Bearer {token}
```

---

## Endpoints

### Roadmap Generation

#### Generate Roadmap (POST)

Generate a complete roadmap structure with nodes and edges based on the provided topic and options.

```http
POST /roadmap/generate
Content-Type: application/json

{
  "topic": "Java Backend Developer",
  "description": "Lộ trình học Java từ cơ bản đến làm việc thực tế",
  "maxNodes": 15,
  "difficulty": "BEGINNER"
}
```

**Request Body:**

- `topic` (required, string): The main topic for the roadmap
- `description` (optional, string): Additional description or context
- `maxNodes` (optional, integer, default: 15): Maximum number of nodes to generate
- `difficulty` (optional, string, default: "INTERMEDIATE"): Difficulty level
  - Options: `BEGINNER`, `INTERMEDIATE`, `ADVANCED`

**Response (200 OK):**

```json
{
  "name": "Lộ trình Java Backend Developer",
  "description": "Lộ trình chi tiết từ cơ bản đến nâng cao cho Java Backend Developer",
  "nodes": [
    {
      "nodeId": "node_1",
      "label": "Giới thiệu Java",
      "content": "Tìm hiểu về lịch sử Java, đặc điểm của ngôn ngữ, và cách cài đặt môi trường phát triển. Học các khái niệm cơ bản về JDK, JRE, và JVM.",
      "nodeType": "TOPIC_NODE",
      "resources": [
        {
          "title": "Java Tutorial - Oracle",
          "url": "https://docs.oracle.com/javase/tutorial/"
        },
        {
          "title": "Java Programming - W3Schools",
          "url": "https://www.w3schools.com/java/"
        }
      ]
    },
    {
      "nodeId": "node_2",
      "label": "Cú pháp Java cơ bản",
      "content": "Học cú pháp Java: biến, kiểu dữ liệu, toán tử, câu lệnh điều kiện, vòng lặp. Thực hành viết các chương trình đơn giản.",
      "nodeType": "TOPIC_NODE",
      "resources": []
    },
    {
      "nodeId": "node_3",
      "label": "Lập trình hướng đối tượng",
      "content": "Hiểu về OOP trong Java: class, object, inheritance, polymorphism, encapsulation, abstraction. Thực hành tạo các class và object.",
      "nodeType": "TOPIC_NODE",
      "resources": [
        {
          "title": "OOP Concepts in Java",
          "url": "https://www.baeldung.com/java-oop"
        }
      ]
    }
  ],
  "edges": [
    {
      "edgeId": "edge_1",
      "source": "node_1",
      "target": "node_2"
    },
    {
      "edgeId": "edge_2",
      "source": "node_2",
      "target": "node_3"
    }
  ]
}
```

**Error Response (400 Bad Request):**

```json
{
  "error": "Invalid request",
  "message": "Topic is required"
}
```

**Error Response (500 Internal Server Error):**

```json
{
  "error": "AI service error",
  "message": "Không thể tạo roadmap. Vui lòng thử lại sau."
}
```

---

#### Generate Roadmap (GET)

Quick generate a roadmap using query parameters.

```http
GET /roadmap/generate?topic=React Frontend Developer&maxNodes=15&difficulty=INTERMEDIATE
```

**Query Parameters:**

- `topic` (required, string): The topic for the roadmap
- `maxNodes` (optional, integer, default: 15): Maximum number of nodes
- `difficulty` (optional, string, default: "INTERMEDIATE"): Difficulty level

**Response:**
Same as POST endpoint above.

**Example:**

```bash
curl -X GET "http://localhost:8085/roadmap/generate?topic=Python%20Data%20Science&maxNodes=12&difficulty=BEGINNER"
```

---

### Node AI Operations

#### Summarize Node Content (POST)

Get a node from Roadmap Service and generate an AI summary of its content.

```http
POST /node/summary
Content-Type: application/json

{
  "roadmapId": "roadmap-123",
  "nodeId": "node-456"
}
```

**Request Body:**

- `roadmapId` (required, string): ID of the roadmap
- `nodeId` (required, string): ID of the node within the roadmap

**Response (200 OK):**

```json
{
  "roadmapId": "roadmap-123",
  "nodeId": "node-456",
  "nodeLabel": "Lập trình hướng đối tượng",
  "summary": {
    "mainPoints": [
      "OOP là phương pháp lập trình tổ chức code thành các đối tượng",
      "4 nguyên lý chính: Encapsulation, Inheritance, Polymorphism, Abstraction",
      "Giúp code dễ bảo trì, mở rộng và tái sử dụng",
      "Java hỗ trợ đầy đủ các tính năng OOP"
    ],
    "keyTakeaways": [
      "Class là bản thiết kế, Object là thực thể cụ thể",
      "Inheritance cho phép kế thừa thuộc tính và phương thức",
      "Polymorphism cho phép một đối tượng có nhiều hình thái"
    ]
  },
  "generatedAt": "2026-01-29T10:00:00Z"
}
```

**Error Response (404 Not Found):**

```json
{
  "error": "Not found",
  "message": "Node or Roadmap not found"
}
```

---

#### Summarize Node Content (GET)

Alternative endpoint using path parameters.

```http
GET /node/summary/{roadmapId}/{nodeId}
```

**Path Parameters:**

- `roadmapId` (required): ID of the roadmap
- `nodeId` (required): ID of the node

**Response:**
Same as POST endpoint above.

**Example:**

```bash
curl -X GET "http://localhost:8085/node/summary/roadmap-123/node-456"
```

---

#### Ask Question About Node

Ask a question about specific node content and get an AI-powered answer.

```http
POST /node/ask
Content-Type: application/json

{
  "roadmapId": "roadmap-123",
  "nodeId": "node-456",
  "question": "Sự khác biệt giữa Abstract class và Interface trong Java là gì?"
}
```

**Request Body:**

- `roadmapId` (required, string): ID of the roadmap
- `nodeId` (required, string): ID of the node
- `question` (required, string): Question to ask about the node content

**Response (200 OK):**

```json
{
  "roadmapId": "roadmap-123",
  "nodeId": "node-456",
  "question": "Sự khác biệt giữa Abstract class và Interface trong Java là gì?",
  "answer": "Abstract class và Interface có những điểm khác biệt chính:\n\n1. **Phương thức**: Abstract class có thể có cả phương thức abstract và concrete, trong khi Interface (trước Java 8) chỉ có abstract methods.\n\n2. **Biến**: Abstract class có thể có biến instance, còn Interface chỉ có constants (public static final).\n\n3. **Kế thừa**: Một class chỉ có thể extends một Abstract class, nhưng có thể implements nhiều Interfaces.\n\n4. **Constructor**: Abstract class có constructor, Interface không có.\n\n5. **Access modifiers**: Abstract class có thể có các access modifiers khác nhau, Interface mặc định là public.",
  "context": "Based on the node content about OOP in Java",
  "generatedAt": "2026-01-29T10:00:00Z"
}
```

---

### Quiz Generation

#### Generate Quiz Questions

Generate multiple-choice quiz questions based on a topic.

```http
POST /chat
Content-Type: application/json

{
  "title": "Java OOP Fundamentals",
  "numberOfQuestions": 5
}
```

**Request Body:**

- `title` (required, string): Topic or title for quiz generation
- `numberOfQuestions` (optional, integer, default: 5): Number of questions to generate

**Response (200 OK):**

```json
{
  "title": "Java OOP Fundamentals",
  "totalQuestions": 5,
  "questions": [
    {
      "questionId": "q1",
      "questionText": "Nguyên lý nào của OOP cho phép che giấu dữ liệu bên trong class?",
      "options": [
        {
          "optionId": "A",
          "text": "Encapsulation"
        },
        {
          "optionId": "B",
          "text": "Inheritance"
        },
        {
          "optionId": "C",
          "text": "Polymorphism"
        },
        {
          "optionId": "D",
          "text": "Abstraction"
        }
      ],
      "correctAnswer": "A",
      "explanation": "Encapsulation (đóng gói) là nguyên lý cho phép che giấu dữ liệu bên trong class và chỉ cho phép truy cập thông qua các phương thức public."
    },
    {
      "questionId": "q2",
      "questionText": "Trong Java, một class có thể kế thừa từ bao nhiêu class khác?",
      "options": [
        {
          "optionId": "A",
          "text": "Không giới hạn"
        },
        {
          "optionId": "B",
          "text": "Tối đa 2 class"
        },
        {
          "optionId": "C",
          "text": "Chỉ 1 class"
        },
        {
          "optionId": "D",
          "text": "Không thể kế thừa"
        }
      ],
      "correctAnswer": "C",
      "explanation": "Java chỉ hỗ trợ single inheritance, một class chỉ có thể kế thừa từ một class duy nhất nhưng có thể implements nhiều interfaces."
    }
  ],
  "generatedAt": "2026-01-29T10:00:00Z"
}
```

**Error Response (400 Bad Request):**

```json
{
  "error": "Invalid request",
  "message": "Title is required"
}
```

---

## Response Schemas

### NodeResource

```json
{
  "title": "string",
  "url": "string"
}
```

### Node

```json
{
  "nodeId": "string",
  "label": "string",
  "content": "string",
  "nodeType": "TOPIC_NODE | NO_TOPIC_NODE",
  "resources": [NodeResource]
}
```

### Edge

```json
{
  "edgeId": "string",
  "source": "string",
  "target": "string"
}
```

### GenerateRoadmapResponse

```json
{
  "name": "string",
  "description": "string",
  "nodes": [Node],
  "edges": [Edge]
}
```

### NodeSummaryResponse

```json
{
  "roadmapId": "string",
  "nodeId": "string",
  "nodeLabel": "string",
  "summary": {
    "mainPoints": ["string"],
    "keyTakeaways": ["string"]
  },
  "generatedAt": "ISO 8601 datetime"
}
```

### QuizOption

```json
{
  "optionId": "string",
  "text": "string"
}
```

### QuizQuestion

```json
{
  "questionId": "string",
  "questionText": "string",
  "options": [QuizOption],
  "correctAnswer": "string",
  "explanation": "string"
}
```

### QuestionListResponseDto

```json
{
  "title": "string",
  "totalQuestions": "integer",
  "questions": [QuizQuestion],
  "generatedAt": "ISO 8601 datetime"
}
```

---

## Error Handling

### Error Response Format

```json
{
  "error": "Error type",
  "message": "Detailed error message",
  "timestamp": "2026-01-29T10:00:00Z"
}
```

### HTTP Status Codes

- `200 OK`: Successful request
- `400 Bad Request`: Invalid request parameters
- `404 Not Found`: Resource not found (roadmap/node)
- `500 Internal Server Error`: AI service error, database error
- `503 Service Unavailable`: External service unavailable

---

## Rate Limiting

The service is subject to rate limits from external AI providers:

- **Groq API**: 30 requests/minute, 14400 requests/day
- **OpenAI Embeddings**: 3000 requests/minute

If rate limits are exceeded, you'll receive:

```json
{
  "error": "Rate limit exceeded",
  "message": "Too many requests. Please try again later.",
  "retryAfter": 60
}
```

---

## Best Practices

1. **Cache Results**: Cache AI-generated content when possible
2. **Error Handling**: Implement retry logic for transient failures
3. **Request Validation**: Validate inputs before making API calls
4. **Timeout Handling**: Set appropriate timeouts (AI generation can take 5-30 seconds)
5. **Fallback Strategy**: Have fallback content when AI service is unavailable
6. **Monitoring**: Monitor API usage and costs

---

## Usage Examples

### JavaScript (Fetch API)

```javascript
// Generate Roadmap
const generateRoadmap = async (topic, difficulty = 'INTERMEDIATE') => {
  const response = await fetch('http://localhost:8085/roadmap/generate', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      topic,
      difficulty,
      maxNodes: 15,
    }),
  });

  return await response.json();
};

// Summarize Node
const summarizeNode = async (roadmapId, nodeId) => {
  const response = await fetch('http://localhost:8085/node/summary', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      roadmapId,
      nodeId,
    }),
  });

  return await response.json();
};

// Generate Quiz
const generateQuiz = async (title, numQuestions = 5) => {
  const response = await fetch('http://localhost:8085/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      title,
      numberOfQuestions: numQuestions,
    }),
  });

  return await response.json();
};
```

### Python (requests)

```python
import requests

# Generate Roadmap
def generate_roadmap(topic, difficulty='INTERMEDIATE', max_nodes=15):
    url = 'http://localhost:8085/roadmap/generate'
    payload = {
        'topic': topic,
        'difficulty': difficulty,
        'maxNodes': max_nodes
    }
    response = requests.post(url, json=payload)
    return response.json()

# Summarize Node
def summarize_node(roadmap_id, node_id):
    url = 'http://localhost:8085/node/summary'
    payload = {
        'roadmapId': roadmap_id,
        'nodeId': node_id
    }
    response = requests.post(url, json=payload)
    return response.json()

# Generate Quiz
def generate_quiz(title, num_questions=5):
    url = 'http://localhost:8085/chat'
    payload = {
        'title': title,
        'numberOfQuestions': num_questions
    }
    response = requests.post(url, json=payload)
    return response.json()
```

---

## Integration with Roadmap Service

The AI Service integrates with Roadmap Service to fetch node content:

```java
// Internal API call to fetch node data
GET http://roadmap-service:8083/road-map/api/roadmap/{roadmapId}

// Extract node content for summarization or Q&A
Node node = roadmap.getNodes().stream()
    .filter(n -> n.getNodeId().equals(nodeId))
    .findFirst()
    .orElseThrow(() -> new NodeNotFoundException());
```

---

## Future Enhancements

1. **Streaming Responses**: Stream AI responses for better UX
2. **Multi-language Support**: Support English and other languages
3. **Custom Models**: Allow users to choose different AI models
4. **Fine-tuning**: Fine-tune models on domain-specific data
5. **Conversation History**: Maintain chat history across sessions
6. **Advanced RAG**: Implement more sophisticated RAG with document chunking
7. **Image Generation**: Generate diagrams and visualizations
8. **Voice Integration**: Support voice input/output
