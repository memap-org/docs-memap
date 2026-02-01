# Learning Service API Documentation

## Base URL

```
http://localhost:8089/learning
```

## Authentication

All endpoints require authentication via JWT tokens issued by Keycloak and **TEACHER role**:

```
Authorization: Bearer {token}
```

---

## Endpoints

### Quiz Management

#### Create Quiz

```http
POST /api/v1/quizzes
Content-Type: application/json
Authorization: Bearer {token}

{
  "title": "Java OOP Basics Quiz",
  "description": "Test your understanding of Object-Oriented Programming in Java",
  "roadmapId": "roadmap-123",
  "questions": [
    {
      "questionText": "What is encapsulation in OOP?",
      "options": [
        {
          "optionText": "Hiding implementation details",
          "isCorrect": true
        },
        {
          "optionText": "Creating multiple classes",
          "isCorrect": false
        },
        {
          "optionText": "Using inheritance",
          "isCorrect": false
        },
        {
          "optionText": "Method overloading",
          "isCorrect": false
        }
      ]
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "code": 1000,
  "message": "Quiz created successfully",
  "result": {
    "quizId": "quiz-456",
    "title": "Java OOP Basics Quiz",
    "description": "Test your understanding of Object-Oriented Programming in Java",
    "roadmapId": "roadmap-123",
    "visible": false,
    "questionCount": 1,
    "createdAt": "2026-01-29T10:00:00Z",
    "teacherId": "teacher-123"
  }
}
```

---

#### Get Quiz by ID

```http
GET /api/v1/quizzes/{quizId}
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 1000,
  "message": "Quiz retrieved successfully",
  "result": {
    "quizId": "quiz-456",
    "title": "Java OOP Basics Quiz",
    "description": "Test your understanding...",
    "roadmapId": "roadmap-123",
    "visible": false,
    "teacherId": "teacher-123",
    "questions": [
      {
        "questionId": "q1",
        "questionText": "What is encapsulation in OOP?",
        "options": [
          {
            "optionId": "opt1",
            "optionText": "Hiding implementation details",
            "isCorrect": true
          },
          {
            "optionId": "opt2",
            "optionText": "Creating multiple classes",
            "isCorrect": false
          }
        ]
      }
    ],
    "createdAt": "2026-01-29T10:00:00Z",
    "updatedAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Get My Quizzes

```http
GET /api/v1/quizzes/my-quizzes?roadmapId=roadmap-123&visible=false&search=Java&page=0&size=10&sortBy=createdAt&sortDir=desc
Authorization: Bearer {token}
```

**Query Parameters:**

- `roadmapId` (optional): Filter by roadmap
- `visible` (optional): Filter by visibility (true/false)
- `search` (optional): Search in quiz titles
- `page` (optional, default: 0): Page number
- `size` (optional, default: 10): Page size
- `sortBy` (optional, default: createdAt): Sort field
- `sortDir` (optional, default: desc): Sort direction (asc/desc)

**Response (200 OK):**

```json
{
  "code": 1000,
  "message": "Quizzes retrieved successfully",
  "result": {
    "quizzes": [
      {
        "quizId": "quiz-456",
        "title": "Java OOP Basics Quiz",
        "description": "Test your understanding...",
        "roadmapId": "roadmap-123",
        "visible": false,
        "questionCount": 5,
        "createdAt": "2026-01-29T10:00:00Z"
      }
    ],
    "currentPage": 0,
    "totalPages": 1,
    "totalElements": 1,
    "pageSize": 10
  }
}
```

---

#### Update Quiz

```http
PUT /api/v1/quizzes/{quizId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "title": "Updated Quiz Title",
  "description": "Updated description",
  "questions": [
    {
      "questionText": "Updated question?",
      "options": [
        {
          "optionText": "Option A",
          "isCorrect": true
        },
        {
          "optionText": "Option B",
          "isCorrect": false
        }
      ]
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "code": 1000,
  "message": "Quiz updated successfully",
  "result": {
    "quizId": "quiz-456",
    "title": "Updated Quiz Title",
    "description": "Updated description",
    "visible": false,
    "questions": [...],
    "updatedAt": "2026-01-29T11:00:00Z"
  }
}
```

**Note**: Only draft quizzes (visible: false) can be updated.

---

#### Toggle Quiz Visibility

```http
PUT /api/v1/quizzes/{quizId}/visibility
Content-Type: application/json
Authorization: Bearer {token}

{
  "visible": true
}
```

**Response (200 OK):**

```json
{
  "code": 1000,
  "message": "Quiz visibility updated",
  "result": {
    "quizId": "quiz-456",
    "title": "Java OOP Basics Quiz",
    "visible": true,
    "updatedAt": "2026-01-29T11:00:00Z"
  }
}
```

---

#### Delete Quiz

```http
DELETE /api/v1/quizzes/{quizId}
Authorization: Bearer {token}
```

**Response (204 No Content)**

**Note**: Only draft quizzes (visible: false) can be deleted.

---

## Response Schema

### ApiResponse

```json
{
  "code": 1000,
  "message": "string",
  "result": {}
}
```

### QuizSummaryResponse

```json
{
  "quizId": "string",
  "title": "string",
  "description": "string",
  "roadmapId": "string",
  "visible": boolean,
  "questionCount": integer,
  "createdAt": "ISO 8601 datetime",
  "teacherId": "string"
}
```

### QuizDetailResponse

```json
{
  "quizId": "string",
  "title": "string",
  "description": "string",
  "roadmapId": "string",
  "visible": boolean,
  "teacherId": "string",
  "questions": [Question],
  "createdAt": "ISO 8601 datetime",
  "updatedAt": "ISO 8601 datetime"
}
```

### Question

```json
{
  "questionId": "string",
  "questionText": "string",
  "options": [Option]
}
```

### Option

```json
{
  "optionId": "string",
  "optionText": "string",
  "isCorrect": boolean
}
```

---

## Status Codes

- `200 OK`: Successful request
- `201 Created`: Resource created
- `204 No Content`: Successful deletion
- `400 Bad Request`: Invalid request
- `401 Unauthorized`: Missing or invalid token
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource not found
- `500 Internal Server Error`: Server error

---

## Error Response Format

```json
{
  "code": 1001,
  "message": "Validation error: Title is required"
}
```

---

## Business Rules

1. **Teacher-Only Access**: All endpoints require TEACHER role
2. **Draft Quizzes**: Only draft quizzes (visible: false) can be edited or deleted
3. **Published Quizzes**: Published quizzes (visible: true) are read-only
4. **Ownership**: Teachers can only access their own quizzes
5. **Validation**: At least one question required, each question must have 2-5 options
6. **Correct Answer**: Exactly one option must be marked as correct

---

## Validation Rules

### CreateQuizRequest

- `title`: Required, 3-200 characters
- `description`: Optional, max 1000 characters
- `roadmapId`: Required, valid UUID format
- `questions`: Required, at least 1 question

### Question

- `questionText`: Required, 10-500 characters
- `options`: Required, 2-5 options

### Option

- `optionText`: Required, 1-200 characters
- `isCorrect`: Required, exactly one option per question must be true

---

## Future Endpoints (Not Yet Implemented)

### Student Quiz Taking

```http
POST /api/v1/quizzes/{quizId}/attempts
GET /api/v1/quizzes/{quizId}/attempts/{attemptId}
POST /api/v1/quizzes/{quizId}/attempts/{attemptId}/submit
```

### Quiz Results

```http
GET /api/v1/quizzes/{quizId}/results
GET /api/v1/quizzes/{quizId}/analytics
```

---

## Usage Examples

### JavaScript (Fetch API)

```javascript
// Create Quiz
const createQuiz = async (quizData, token) => {
  const response = await fetch(
    'http://localhost:8089/learning/api/v1/quizzes',
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify(quizData),
    },
  );
  return await response.json();
};

// Get My Quizzes
const getMyQuizzes = async (token, filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(
    `http://localhost:8089/learning/api/v1/quizzes/my-quizzes?${params}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    },
  );
  return await response.json();
};

// Toggle Visibility
const toggleVisibility = async (quizId, visible, token) => {
  const response = await fetch(
    `http://localhost:8089/learning/api/v1/quizzes/${quizId}/visibility`,
    {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({ visible }),
    },
  );
  return await response.json();
};
```

### Python (requests)

```python
import requests

# Create Quiz
def create_quiz(quiz_data, token):
    url = 'http://localhost:8089/learning/api/v1/quizzes'
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {token}'
    }
    response = requests.post(url, json=quiz_data, headers=headers)
    return response.json()

# Get My Quizzes
def get_my_quizzes(token, filters=None):
    url = 'http://localhost:8089/learning/api/v1/quizzes/my-quizzes'
    headers = {'Authorization': f'Bearer {token}'}
    response = requests.get(url, params=filters, headers=headers)
    return response.json()

# Delete Quiz
def delete_quiz(quiz_id, token):
    url = f'http://localhost:8089/learning/api/v1/quizzes/{quiz_id}'
    headers = {'Authorization': f'Bearer {token}'}
    response = requests.delete(url, headers=headers)
    return response.status_code == 204
```
