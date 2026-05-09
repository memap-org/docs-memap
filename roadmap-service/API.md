# Roadmap Service API Documentation

> **Branch:** `main` | **Port:** 8083 | **Context path:** `/road-map` (via gateway rewrite from `/roadmap`)
> All endpoints require a valid Keycloak JWT unless noted otherwise.

---

## Base URLs

```
Via API Gateway:  http://localhost:8090/roadmap/...
Direct:           http://localhost:8083/road-map/...
Swagger UI:       http://localhost:8083/swagger-ui.html
```

---

## Roadmap Context (`com.techmap.roadmap`)

### Roadmap CRUD — `RoadMapController`

Base: `api/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap` | Create a new roadmap |
| POST | `/api/roadmap/ai-generated` | Create a roadmap from AI-generated structure |
| GET | `/api/roadmap/{roadmapId}` | Get roadmap by ID (with nodes + edges) |
| GET | `/api/roadmap/my-roadmap` | Get current user's roadmaps (paginated) |
| GET | `/api/roadmap/accessible` | Get all roadmaps the user can access (paginated) |
| DELETE | `/api/roadmap/{roadmapId}` | Delete a roadmap |
| PATCH | `/api/roadmap/node/{roadmapId}` | Update node positions/content |
| PATCH | `/api/roadmap/node/status/{roadmapId}` | Update node completion status |
| PATCH | `/api/roadmap/edge/{roadmapId}` | Update edges |

### Roadmap Permissions — `RoadMapAccessPermissionController`

Base: `api/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap/scope/user/{roadmapId}` | Add user access (generic permission) |
| DELETE | `/api/roadmap/scope/user/{roadmapId}` | Remove user access |
| POST | `/api/roadmap/{roadmapId}/teachers` | Add a teacher to the roadmap |
| POST | `/api/roadmap/{roadmapId}/students` | Add a student to the roadmap |
| GET | `/api/roadmap/{roadmapId}/permissions` | List all users and their roles |
| PATCH | `/api/roadmap/{roadmapId}/permissions/{userId}/role` | Change a user's role |
| DELETE | `/api/roadmap/{roadmapId}/teachers/{userId}` | Remove a teacher |
| DELETE | `/api/roadmap/{roadmapId}/students/{userId}` | Remove a student |

> Adding a teacher/student triggers a gRPC call to profile-service to resolve the sender's display name, then sends a notification.

### Roadmap Interactions — `RoadmapInteractionController`

Base: `api/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap/{roadmapId}/comments` | Add comment on a roadmap |
| POST | `/api/roadmap/{roadmapId}/nodes/{nodeId}/comments` | Add comment on a specific node |
| DELETE | `/api/roadmap/comments/{commentId}` | Delete a comment |
| GET | `/api/roadmap/comments/{commentId}` | Get a comment by ID |
| GET | `/api/roadmap/{roadmapId}/comments` | List roadmap comments (paginated) |
| GET | `/api/roadmap/{roadmapId}/nodes/{nodeId}/comments` | List node comments (paginated) |
| POST | `/api/roadmap/{roadmapId}/reactions` | Toggle reaction on roadmap |
| POST | `/api/roadmap/comments/{commentId}/reactions` | Toggle reaction on comment |
| GET | `/api/roadmap/{roadmapId}/reactions` | List all reactions on a roadmap |

> `toggleRoadmapReaction` calls profile-service via gRPC to resolve the reactor's display name.

### Roadmap Notes — `RoadmapNoteController`

Base: `api/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap/{roadmapId}/notes` | Create a personal note |
| GET | `/api/roadmap/{roadmapId}/notes` | Get the current user's note |
| GET | `/api/roadmap/{roadmapId}/notes/all` | Get all notes on a roadmap (paginated) |
| PATCH | `/api/roadmap/{roadmapId}/notes/{noteId}` | Update a note |
| DELETE | `/api/roadmap/{roadmapId}/notes/{noteId}` | Delete a note |

### Focus Tracking — `RoadmapFocusTrackingController`

Base: `api/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap/{roadmapId}/focus-tracking/sessions` | Start a focus session |
| PATCH | `/api/roadmap/focus-tracking/sessions/{sessionId}` | Update focused minutes |
| GET | `/api/roadmap/focus-tracking/sessions` | Get focus sessions (paginated) |
| GET | `/api/roadmap/focus-tracking/statistics` | Get focus statistics for current user |

### Progress Tracking — `RoadmapProgressTrackingController`

Base: `api/roadmap-tracking-progress`

| Method | Path | Description |
|---|---|---|
| GET | `/api/roadmap-tracking-progress/{roadmapId}` | Get full tracking state |
| GET | `/api/roadmap-tracking-progress/{roadmapId}/{nodeId}` | Get status of a single node |
| PATCH | `/api/roadmap-tracking-progress/{roadmapId}` | Update node tracking status |
| POST | `/api/roadmap-tracking-progress/reset/{roadmapId}` | Reset all progress |
| POST | `/api/roadmap-tracking-progress/done/{roadmapId}` | Mark all nodes as done |

### Roadmap Categories — `RoadmapCategoryController`

Base: `api/roadmap-category`

| Method | Path | Description |
|---|---|---|
| GET | `/api/roadmap-category` | List all categories |
| GET | `/api/roadmap-category/{roadmapCategoryId}` | Get category by ID |
| POST | `/api/roadmap-category` | Create category |
| PUT | `/api/roadmap-category/{roadmapCategoryId}` | Update category |
| DELETE | `/api/roadmap-category/{roadmapCategoryId}` | Delete category |

---

## Learning Context (`com.techmap.learning`)

### Quizzes — `QuizController`

Base: `/api/v1/quizzes` (also `/learning/api/v1/quizzes`)

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/quizzes` | Create a quiz (teacher) |
| PUT | `/api/v1/quizzes/{quizId}/visibility` | Toggle quiz visibility |
| DELETE | `/api/v1/quizzes/{quizId}` | Delete a quiz |
| GET | `/api/v1/quizzes/{quizId}` | Get quiz details |
| GET | `/api/v1/quizzes/{quizId}/questions` | List questions in a quiz |
| GET | `/api/v1/quizzes/my-quizzes` | Get quizzes created by current user |
| PUT | `/api/v1/quizzes/{quizId}` | Update a quiz |

### Student Quiz — `StudentQuizController`

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/quizzes/available` | Get quizzes available to the student |

### Questions — `QuestionController`

Base: `/api/v1/quizzes/{quizId}/questions`

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/quizzes/{quizId}/questions` | Create a question |
| GET | `/api/v1/quizzes/{quizId}/questions/{questionId}` | Get a question |
| PUT | `/api/v1/quizzes/{quizId}/questions/{questionId}` | Update a question |
| DELETE | `/api/v1/quizzes/{quizId}/questions/{questionId}` | Delete a question |
| PUT | `/api/v1/quizzes/{quizId}/questions/reorder` | Reorder questions |

### Quiz Attempts (Teacher view) — `QuizAttemptController`

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/quizzes/{quizId}/attempts` | Start attempt (teacher/admin) |
| PUT | `/api/v1/attempts/{attemptId}/submit` | Submit attempt |
| GET | `/api/v1/attempts/{attemptId}` | Get attempt by ID |
| GET | `/api/v1/quizzes/{quizId}/attempts` | List all attempts for a quiz (paginated) |
| GET | `/api/v1/quizzes/{quizId}/statistics` | Quiz statistics |

### Quiz Attempts (Student view) — `StudentQuizAttemptController`

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/quizzes/{quizId}/attempts/start` | Start a new attempt |
| GET | `/api/v1/quiz-attempts/{attemptId}/status` | Get attempt status |
| GET | `/api/v1/quiz-attempts/active` | Get active attempt by quiz ID |
| GET | `/api/v1/quiz-attempts/{attemptId}/active` | Get active attempt by attempt ID |
| PUT | `/api/v1/quiz-attempts/{attemptId}/auto-save` | Auto-save answers |
| POST | `/api/v1/quiz-attempts/{attemptId}/submit` | Submit attempt |
| GET | `/api/v1/quizzes/{quizId}/attempts/count` | Count attempts for student |

### Assignments (Teacher) — `TeacherAssignmentController`

| Method | Path | Description |
|---|---|---|
| POST | `/api/roadmap/{roadmapId}/assignments` | Create assignment on a roadmap |
| PUT | `/api/assignment/{id}` | Update assignment |
| PATCH | `/api/assignment/{id}/file-url` | Set file URL for assignment |
| DELETE | `/api/assignment/{id}` | Delete assignment |
| GET | `/api/assignment/my-assignments` | List assignments created by teacher (paginated) |
| GET | `/api/assignment/{id}/submissions` | List submissions for an assignment (paginated) |
| PATCH | `/api/assignment/{id}/submissions/{subId}/grade` | Grade a submission |

### Assignments (Student) — `StudentAssignmentController`

| Method | Path | Description |
|---|---|---|
| POST | `/api/assignment/{id}/submit` | Submit an assignment |
| GET | `/api/assignment/{id}/my-submission` | Get own submission |

### Assignment Queries — `AssignmentQueryController`

| Method | Path | Description |
|---|---|---|
| GET | `/api/roadmap/{roadmapId}/assignments` | List assignments for a roadmap (paginated) |
| GET | `/api/assignment/{id}` | Get assignment by ID |

---

## gRPC Server (port 9083)

Consumed by `roadmap-ai-service`.

| RPC | Purpose |
|---|---|
| `RoadmapLookupService.GetRoadmapName(roadmap_id)` | Returns `{name, found}` for a given roadmap ID |

---

## Response Envelope

All REST endpoints return:

```json
{
  "code": 1000,
  "message": null,
  "result": { ... }
}
```

Errors use non-1000 codes. See `docs/ERROR_CODES.md`.

---

## Caching

- Roadmap queries are cached with a **Caffeine + Redis** dual-layer cache.
- Eviction happens on write operations (create/update/delete roadmap).
