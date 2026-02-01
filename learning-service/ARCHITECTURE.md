# Learning Service Architecture

## Service Overview

The Learning Service manages quizzes and assessments for the MeMap platform. It provides a RESTful API for teachers to create, manage, and publish quizzes linked to roadmaps. The service is currently in active development with student quiz-taking features planned.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│                   (Frontend, Mobile Apps)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                     Controller Layer                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ QuizController (@PreAuthorize("hasRole('TEACHER')"))     │  │
│  │ - POST /quizzes (create)                                 │  │
│  │ - GET /quizzes/{id} (get by ID)                          │  │
│  │ - GET /quizzes/my-quizzes (list with filters)            │  │
│  │ - PUT /quizzes/{id} (update)                             │  │
│  │ - PUT /quizzes/{id}/visibility (toggle)                  │  │
│  │ - DELETE /quizzes/{id} (delete)                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ AuthTestController                                       │  │
│  │ - Test authentication endpoints                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Service Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ QuizService                                              │  │
│  │ - createQuiz(request, teacherId): QuizSummaryResponse    │  │
│  │ - getQuizById(quizId, teacherId): QuizDetailResponse     │  │
│  │ - getMyQuizzes(teacherId, filters, pageable): List      │  │
│  │ - updateQuiz(quizId, request, teacherId): Response       │  │
│  │ - toggleVisibility(quizId, visible, teacherId)           │  │
│  │ - deleteQuiz(quizId, teacherId)                          │  │
│  │                                                          │  │
│  │ Business Logic:                                          │  │
│  │ - Validate quiz ownership                                │  │
│  │ - Enforce draft-only editing rule                        │  │
│  │ - Validate questions and options                         │  │
│  │ - Handle cascading operations                            │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                    Repository Layer                             │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │ QuizRepository   │  │ QuestionRepo    │  │ OptionRepo   │  │
│  │                  │  │                 │  │              │  │
│  └──────────────────┘  └─────────────────┘  └──────────────┘  │
│  ┌──────────────────┐  ┌─────────────────┐                    │
│  │ QuizAttemptRepo  │  │ AttemptAnswerRepo│                    │
│  │                  │  │                 │                    │
│  └──────────────────┘  └─────────────────┘                    │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                       Database Layer                            │
│              MySQL (learning_service database)                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Tables:                                                  │   │
│  │ - quizzes (id, title, description, roadmap_id, visible) │   │
│  │ - questions (id, quiz_id, question_text)                │   │
│  │ - options (id, question_id, option_text, is_correct)    │   │
│  │ - quiz_attempts (id, quiz_id, student_id, score)        │   │
│  │ - attempt_answers (id, attempt_id, question_id, ...)    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### QuizController

**Role-Based Access**: `@PreAuthorize("hasRole('TEACHER')")` applied at class level

**Key Endpoints**:

```java
@PostMapping
public ApiResponse<QuizSummaryResponse> createQuiz(
    @Valid @RequestBody CreateQuizRequest request,
    @AuthenticationPrincipal Jwt jwt);

@GetMapping("/{quizId}")
public ApiResponse<QuizDetailResponse> getQuizById(
    @PathVariable String quizId,
    @AuthenticationPrincipal Jwt jwt);

@GetMapping("/my-quizzes")
public ApiResponse<QuizListResponse> getMyQuizzes(
    @RequestParam(required = false) String roadmapId,
    @RequestParam(required = false) Boolean visible,
    @RequestParam(required = false) String search,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String sortDir,
    @AuthenticationPrincipal Jwt jwt);

@PutMapping("/{quizId}")
public ApiResponse<QuizDetailResponse> updateQuiz(
    @PathVariable String quizId,
    @Valid @RequestBody UpdateQuizRequest request,
    @AuthenticationPrincipal Jwt jwt);

@PutMapping("/{quizId}/visibility")
public ApiResponse<QuizSummaryResponse> toggleVisibility(
    @PathVariable String quizId,
    @Valid @RequestBody ToggleVisibilityRequest request,
    @AuthenticationPrincipal Jwt jwt);

@DeleteMapping("/{quizId}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void deleteQuiz(
    @PathVariable String quizId,
    @AuthenticationPrincipal Jwt jwt);
```

### 2. Service Layer

#### QuizService

**Business Logic**:

```java
public QuizSummaryResponse createQuiz(CreateQuizRequest request, String teacherId) {
    // 1. Validate request
    validateQuizRequest(request);

    // 2. Create quiz entity
    Quiz quiz = new Quiz();
    quiz.setTitle(request.getTitle());
    quiz.setDescription(request.getDescription());
    quiz.setRoadmapId(request.getRoadmapId());
    quiz.setTeacherId(teacherId);
    quiz.setVisible(false); // Default to draft

    // 3. Add questions and options
    for (QuestionRequest qReq : request.getQuestions()) {
        Question question = new Question();
        question.setQuestionText(qReq.getQuestionText());
        question.setQuiz(quiz);

        for (OptionRequest oReq : qReq.getOptions()) {
            Option option = new Option();
            option.setOptionText(oReq.getOptionText());
            option.setIsCorrect(oReq.getIsCorrect());
            option.setQuestion(question);
            question.addOption(option);
        }

        quiz.addQuestion(question);
    }

    // 4. Save to database
    Quiz savedQuiz = quizRepository.save(quiz);

    // 5. Convert to response DTO
    return mapToSummaryResponse(savedQuiz);
}

public QuizDetailResponse updateQuiz(String quizId, UpdateQuizRequest request, String teacherId) {
    // 1. Find quiz and verify ownership
    Quiz quiz = quizRepository.findById(quizId)
        .orElseThrow(() -> new QuizNotFoundException(quizId));

    if (!quiz.getTeacherId().equals(teacherId)) {
        throw new UnauthorizedException("You don't own this quiz");
    }

    // 2. Check if quiz is draft
    if (quiz.getVisible()) {
        throw new IllegalStateException("Cannot edit published quiz");
    }

    // 3. Update quiz details
    quiz.setTitle(request.getTitle());
    quiz.setDescription(request.getDescription());

    // 4. Clear existing questions and add new ones
    quiz.getQuestions().clear();

    for (QuestionRequest qReq : request.getQuestions()) {
        Question question = new Question();
        question.setQuestionText(qReq.getQuestionText());
        question.setQuiz(quiz);

        for (OptionRequest oReq : qReq.getOptions()) {
            Option option = new Option();
            option.setOptionText(oReq.getOptionText());
            option.setIsCorrect(oReq.getIsCorrect());
            option.setQuestion(question);
            question.addOption(option);
        }

        quiz.addQuestion(question);
    }

    // 5. Save and return
    Quiz updatedQuiz = quizRepository.save(quiz);
    return mapToDetailResponse(updatedQuiz);
}
```

### 3. Data Models

#### Quiz Entity

```java
@Entity
@Table(name = "quizzes")
public class Quiz extends BaseModel {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(length = 1000)
    private String description;

    @Column(nullable = false)
    private String roadmapId;

    @Column(nullable = false)
    private String teacherId;

    @Column(nullable = false)
    private Boolean visible = false;

    @OneToMany(mappedBy = "quiz", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Question> questions = new ArrayList<>();

    @OneToMany(mappedBy = "quiz", cascade = CascadeType.ALL)
    private List<QuizAttempt> attempts = new ArrayList<>();
}
```

#### Question Entity

```java
@Entity
@Table(name = "questions")
public class Question extends BaseModel {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false, length = 500)
    private String questionText;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "quiz_id", nullable = false)
    private Quiz quiz;

    @OneToMany(mappedBy = "question", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Option> options = new ArrayList<>();
}
```

#### Option Entity

```java
@Entity
@Table(name = "options")
public class Option extends BaseModel {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false, length = 200)
    private String optionText;

    @Column(nullable = false)
    private Boolean isCorrect = false;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "question_id", nullable = false)
    private Question question;
}
```

#### QuizAttempt Entity (Future)

```java
@Entity
@Table(name = "quiz_attempts")
public class QuizAttempt extends BaseModel {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "quiz_id", nullable = false)
    private Quiz quiz;

    @Column(nullable = false)
    private String studentId;

    private Integer score;
    private Integer maxScore;

    @Column(nullable = false)
    private LocalDateTime startedAt;

    private LocalDateTime completedAt;

    @OneToMany(mappedBy = "attempt", cascade = CascadeType.ALL)
    private List<AttemptAnswer> answers = new ArrayList<>();
}
```

## Design Patterns

### 1. Repository Pattern

- Spring Data JPA repositories
- Type-safe queries
- Automatic CRUD implementation

### 2. Service Layer Pattern

- Business logic in service classes
- Transaction management
- DTO mapping

### 3. DTO Pattern

- Request DTOs for input validation
- Response DTOs for API contracts
- Separation from entity models

### 4. Builder Pattern

- Lombok `@Builder` for DTOs
- Fluent API for object construction

## Security Architecture

### OAuth2 Resource Server

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter =
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);

        return jwtAuthenticationConverter;
    }
}
```

### Role-Based Access Control

- `@PreAuthorize("hasRole('TEACHER')")` at controller level
- Teacher ID extracted from JWT token
- Ownership validation in service layer

## Database Schema

### Entity Relationships

```
Quiz 1---N Question 1---N Option
Quiz 1---N QuizAttempt 1---N AttemptAnswer
```

### Indexes

```sql
CREATE INDEX idx_quiz_teacher_id ON quizzes(teacher_id);
CREATE INDEX idx_quiz_roadmap_id ON quizzes(roadmap_id);
CREATE INDEX idx_quiz_visible ON quizzes(visible);
CREATE INDEX idx_question_quiz_id ON questions(quiz_id);
CREATE INDEX idx_option_question_id ON options(question_id);
```

## Migration Management

### Flyway Configuration

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    validate-on-migrate: true
```

### Migration Files

- `V1__create_quiz_tables.sql` - Initial schema
- `V2__add_indexes.sql` - Performance indexes
- `V3__add_quiz_attempts.sql` - Student attempts tables

## Error Handling

### Exception Hierarchy

```java
@ControllerAdvice
public class LearningExceptionHandler {

    @ExceptionHandler(QuizNotFoundException.class)
    public ResponseEntity<ApiResponse<?>> handleQuizNotFound(QuizNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.builder()
                .code(1002)
                .message(ex.getMessage())
                .build());
    }

    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ApiResponse<?>> handleUnauthorized(UnauthorizedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(ApiResponse.builder()
                .code(1003)
                .message(ex.getMessage())
                .build());
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<ApiResponse<?>> handleIllegalState(IllegalStateException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.builder()
                .code(1001)
                .message(ex.getMessage())
                .build());
    }
}
```

## Future Architecture Enhancements

### Student Quiz Taking

```
Student → Take Quiz → Record Attempt → Submit Answers → Calculate Score → Store Results
```

### Grading System

- Automatic grading for multiple choice
- Manual grading for essay questions
- Partial credit support

### Analytics

- Quiz performance metrics
- Student progress tracking
- Question difficulty analysis
- Answer distribution statistics

## Integration Points

### Roadmap Service

- Validate roadmap existence
- Link quizzes to specific nodes
- Fetch roadmap metadata

### AI Service

- Generate quiz questions automatically
- Suggest improvements
- Create question variations

## Performance Considerations

### Pagination

- Default page size: 10
- Maximum page size: 100
- Efficient offset pagination

### Lazy Loading

- `@ManyToOne(fetch = FetchType.LAZY)`
- Load related entities only when needed

### Caching (Future)

- Cache published quizzes
- Redis integration
- TTL-based expiration
