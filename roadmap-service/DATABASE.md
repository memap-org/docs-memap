# Roadmap Service Database Schema

> **Database Type:** MongoDB (Spring Data MongoDB)
> **ORM Framework:** Spring Data MongoDB with `@Document` annotations

## Collections

### RoadMap

Main roadmap collection. Extends `BaseDocument` (id, created_date, updated_date, is_deleted).

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag, default false (from BaseDocument) |
| ownerId | String | Owner/user ID of the roadmap |
| name | String | Roadmap name/title |
| description | String | Roadmap description |
| nodes | List\<Node\> | Embedded graph nodes |
| edges | List\<Edge\> | Embedded graph edges |
| scope | Enum (GROUP, ...) | Access scope of the roadmap, default GROUP |
| permissions | List\<RoadMapAccessPermission\> | Embedded user permissions |
| imageUrl | String | Image URL for roadmap, default empty |
| creationSource | Enum (MANUAL, ...) | How the roadmap was created, default MANUAL |
| category | RoadmapCategory (DBRef) | Reference to roadmap category |
| resources | List\<RoadmapResource\> | Embedded resources attached to roadmap |

---

### RoadMapCategory

Category collection for organizing roadmaps. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| name | String | Category name (unique index) |
| description | String | Category description |

---

### Comment

Comments on roadmaps or specific nodes. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| roadmapId | String | ID of the roadmap this comment belongs to |
| nodeId | String | ID of the node (nullable, only for node-specific comments) |
| replyTo | String | ID of the comment being replied to (nullable) |
| userId | String | ID of the user who wrote the comment |
| content | String | Comment text content |

**Indexes:**
- `idx_comment_roadmap_active_created` on `roadmapId, is_deleted, created_date`
- `idx_comment_node_active_created` on `roadmapId, nodeId, is_deleted, created_date`

---

### Reaction

Reactions to roadmaps or comments. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| userId | String | ID of the user who reacted |
| content | Enum | Reaction type/content |
| roadmapId | String | ID of the roadmap (nullable) |
| commentId | String | ID of the comment (nullable) |

**Indexes:**
- `idx_reaction_roadmap_active_created` on `roadmapId, is_deleted, created_date`
- `idx_reaction_comment_active_created` on `commentId, is_deleted, created_date`
- `ux_reaction_user_roadmap` on `userId, roadmapId` (unique, sparse)
- `ux_reaction_user_comment` on `userId, commentId` (unique, sparse)

---

### RoadmapNote

User notes on a roadmap. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| roadmap_id | String | ID of the roadmap (indexed, not blank) |
| user_id | String | ID of the user (indexed, not blank) |
| content | String | Note content (max 50,000 characters, not blank) |

**Indexes:**
- `idx_roadmap_note_user_unique` on `roadmapId, userId` (unique)
- `idx_roadmap_note_list` on `roadmapId, is_deleted, created_date`

---

### RoadmapFocusTracking

Focus session tracking for roadmaps. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| userId | String | ID of the user |
| roadmapId | String | ID of the roadmap |
| claimTargetMinutes | Integer | Target minutes set by user, default 0 |
| focusedMinutes | Integer | Actual minutes focused, default 0 |

---

### NodeProgressTracking

Tracks user progress on roadmap nodes. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| nodeId | String | ID of the node |
| roadmapId | String | ID of the roadmap |
| userId | String | ID of the user |
| trackingStatus | Enum (PENDING, ...) | Progress status, default PENDING |

---

### ExcelImportJob

Tracks Excel import jobs for roadmaps.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| roadmap_id | String | ID of the roadmap (indexed) |
| requested_by | String | ID of user who requested import (indexed) |
| status | Enum (PENDING, ...) | Import job status, default PENDING |
| total_rows | int | Total number of rows to import |
| success_count | int | Number of successfully imported rows, default 0 |
| failure_count | int | Number of failed rows, default 0 |
| errors | List\<ImportError\> | List of import errors (rowIndex, email, errorMessage) |
| created_at | LocalDateTime | Auto-set on creation |
| completed_at | LocalDateTime | Job completion timestamp |

---

### Quiz

Quiz entity for learning module. Extends `BaseModel` (id, created_at, updated_at, deleted, deleted_at, deleted_by).

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseModel) |
| created_at | LocalDateTime | Auto-set on creation (from BaseModel) |
| updated_at | LocalDateTime | Auto-set on update (from BaseModel) |
| deleted | Boolean | Soft delete flag (from BaseModel) |
| deleted_at | LocalDateTime | Soft delete timestamp (from BaseModel) |
| deleted_by | String | User who deleted (from BaseModel) |
| title | String | Quiz title |
| description | String | Quiz description |
| teacher_id | String | ID of the teacher who created the quiz |
| roadmap_id | String | ID of the associated roadmap |
| visible | Boolean | Whether quiz is visible, default true |
| start_date | LocalDateTime | Quiz start date/time |
| end_date | LocalDateTime | Quiz end date/time |
| duration_minutes | Integer | Time limit in minutes |
| max_attempts | Integer | Maximum number of attempts allowed |
| questions | List\<Question\> | Embedded questions |

**Indexes:**
- `idx_quiz_teacher` on `teacher_id, deleted, created_at`
- `idx_quiz_roadmap` on `roadmap_id, visible, deleted`

---

### Question

Quiz questions. Extends `BaseModel`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseModel) |
| created_at | LocalDateTime | Auto-set on creation (from BaseModel) |
| updated_at | LocalDateTime | Auto-set on update (from BaseModel) |
| deleted | Boolean | Soft delete flag (from BaseModel) |
| deleted_at | LocalDateTime | Soft delete timestamp (from BaseModel) |
| deleted_by | String | User who deleted (from BaseModel) |
| quiz_id | String | ID of the parent quiz |
| question_text | String | The question text |
| question_type | Enum (MULTIPLE_CHOICE, TRUE_FALSE, SHORT_ANSWER) | Type of question |
| points | Integer | Points for this question, default 1 |
| order_index | Integer | Display order of the question |
| code_snippet | String | Optional code snippet for the question |
| image_url | String | Optional image URL for the question |
| explanation | String | Explanation for the correct answer |
| correct_answer | String | The correct answer |
| options | List\<Option\> | Embedded answer options |

**Indexes:**
- `idx_question_quiz` on `quiz_id, deleted, order_index`

---

### QuizAttempt

Student's quiz attempt record. Extends `BaseModel`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseModel) |
| created_at | LocalDateTime | Auto-set on creation (from BaseModel) |
| updated_at | LocalDateTime | Auto-set on update (from BaseModel) |
| deleted | Boolean | Soft delete flag (from BaseModel) |
| deleted_at | LocalDateTime | Soft delete timestamp (from BaseModel) |
| deleted_by | String | User who deleted (from BaseModel) |
| quiz_id | String | ID of the quiz |
| quiz | Quiz | Embedded quiz reference |
| student_id | String | ID of the student |
| student_name | String | Name of the student |
| score | BigDecimal | Score achieved |
| completion_time_seconds | Integer | Time taken in seconds |
| attempt_number | Integer | Which attempt this is |
| is_passed | Boolean | Whether the student passed |
| started_at | LocalDateTime | When the attempt started |
| submitted_at | LocalDateTime | When the attempt was submitted |
| saved_answers_json | String | JSON string of saved answers |
| flagged_question_ids_json | String | JSON string of flagged question IDs |
| current_question_index | Integer | Current question being answered, default 0 |
| answers | List\<AttemptAnswer\> | Embedded answers |

**Indexes:**
- `idx_attempt_quiz_student` on `quiz.id, student_id, deleted`
- `idx_attempt_student` on `student_id, deleted, started_at`

---

### Assignment

Assignments for learning module. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| roadmap_id | String | ID of the associated roadmap (indexed) |
| teacher_id | String | ID of the teacher who created it |
| title | String | Assignment title |
| description | String | Assignment description |
| type | Enum | Assignment type |
| visible | boolean | Whether assignment is visible, default false |
| start_date | Date | Assignment start date |
| end_date | Date | Assignment end date |
| file_url | String | URL to assignment file |

**Indexes:**
- `idx_assignment_roadmap_list` on `roadmap_id, is_deleted, created_date`
- `idx_assignment_teacher_list` on `teacher_id, is_deleted, created_date`

---

### AssignmentSubmission

Student assignment submissions. Extends `BaseDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseDocument) |
| created_date | Date | Auto-set on creation (from BaseDocument) |
| updated_date | Date | Auto-set on update (from BaseDocument) |
| is_deleted | Boolean | Soft delete flag (from BaseDocument) |
| assignment_id | String | ID of the assignment (indexed) |
| roadmap_id | String | ID of the roadmap |
| student_id | String | ID of the student |
| student_name | String | Name of the student |
| text_content | String | Text content of the submission |
| file_ids | List\<String\> | IDs of submitted files |
| file_urls | List\<String\> | Legacy file URLs |
| status | Enum (PENDING, ...) | Submission status, default PENDING |
| grade | Integer | Assigned grade |
| feedback | String | Teacher feedback |
| submitted_at | Date | Submission timestamp |

**Indexes:**
- `idx_submission_assignment_student_unique` on `assignment_id, student_id` (unique)
- `idx_submission_assignment_list` on `assignment_id, is_deleted, submitted_at`

## Embedded Types

### Node (embedded in RoadMap)

| Field | Type | Description |
|-------|------|-------------|
| nodeId | String | Unique identifier for the node |
| label | String | Display label |
| content | String | Node content/description |
| nodeType | Enum | Type of node |
| resources | List\<ResourceNode\> | Embedded resources |
| style | Map\<String, Object\> | Styling properties |

### Edge (embedded in RoadMap)

| Field | Type | Description |
|-------|------|-------------|
| edgeId | String | Unique identifier for the edge |
| source | String | Source node ID |
| target | String | Target node ID |
| style | Map\<String, Object\> | Styling properties |

### ResourceNode (embedded in Node)

| Field | Type | Description |
|-------|------|-------------|
| title | String | Resource title |
| url | String | Resource URL |

### RoadmapResource (embedded in RoadMap)

| Field | Type | Description |
|-------|------|-------------|
| id | String | Resource identifier |
| fileId | String | File ID reference |
| url | String | Resource URL |
| title | String | Resource title |
| type | String | Resource type |
| addedAt | Date | When resource was added |
| addedBy | String | User who added the resource |

### RoadMapAccessPermission (embedded in RoadMap)

| Field | Type | Description |
|-------|------|-------------|
| userId | String | User ID |
| permission | Enum (VIEW, ...) | Permission level, default VIEW |
| role | Enum | User role (nullable - null means both roles) |

### Option (embedded in Question)

Extends `BaseModel`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseModel) |
| created_at | LocalDateTime | Auto-set on creation (from BaseModel) |
| updated_at | LocalDateTime | Auto-set on update (from BaseModel) |
| deleted | Boolean | Soft delete flag (from BaseModel) |
| deleted_at | LocalDateTime | Soft delete timestamp (from BaseModel) |
| deleted_by | String | User who deleted (from BaseModel) |
| question_id | String | ID of the parent question |
| option_text | String | Option display text |
| is_correct | Boolean | Whether this is the correct answer, default false |
| order_index | Integer | Display order of the option |

### AttemptAnswer (embedded in QuizAttempt)

Extends `BaseModel`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseModel) |
| created_at | LocalDateTime | Auto-set on creation (from BaseModel) |
| updated_at | LocalDateTime | Auto-set on update (from BaseModel) |
| deleted | Boolean | Soft delete flag (from BaseModel) |
| deleted_at | LocalDateTime | Soft delete timestamp (from BaseModel) |
| deleted_by | String | User who deleted (from BaseModel) |
| question | Question | Embedded question reference |
| selected_option | Option | Selected option for multiple choice |
| text_answer | String | Text answer for short answer questions |
| is_correct | Boolean | Whether the answer is correct, default false |
