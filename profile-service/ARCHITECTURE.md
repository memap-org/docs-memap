# Profile Service Architecture

## Service Overview

The Profile Service manages user profiles and personal information with integration to caching (Redis), messaging (RabbitMQ), and file storage services. It follows a microservices architecture with clear separation of concerns.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│                  (HTTP Requests + JWT Cookie)                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                     Controller Layer                            │
│  ┌────────────────────────────────────────────────────┐         │
│  │              UserController                        │         │
│  │  - Registration    - Profile CRUD                  │         │
│  │  - Password mgmt   - Avatar upload                 │         │
│  │  - User search     - Bulk operations               │         │
│  └────────────────────────────────────────────────────┘         │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Service Layer                             │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ UserService      │    │ EmailService     │                  │
│  │ - Profile logic  │───>│ - Send emails    │                  │
│  │ - Validation     │    │ - Templates      │                  │
│  └────────┬─────────┘    └──────────────────┘                  │
│           │                                                      │
│           ├──────────────────────────────┐                      │
│           │                              │                      │
│  ┌────────▼─────────┐         ┌─────────▼────────┐             │
│  │ MessageProducer  │         │ FileServiceClient│             │
│  │ - RabbitMQ       │         │ - Avatar upload  │             │
│  │ - User events    │         │ - Feign client   │             │
│  └──────────────────┘         └──────────────────┘             │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                    Repository Layer                             │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │ UserRepository   │         │ RedisTemplate    │             │
│  │ - JPA CRUD       │         │ - Cache ops      │             │
│  └──────────────────┘         └──────────────────┘             │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Data Layer                                │
│  ┌──────────────────┐         ┌──────────────────┐             │
│  │     MySQL        │         │      Redis       │             │
│  │ - User profiles  │         │ - Profile cache  │             │
│  │ - User metadata  │         │ - Session data   │             │
│  └──────────────────┘         └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                  External Services                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Identity Service │  │  File Service    │  │   RabbitMQ    │ │
│  │ - Auth/JWT       │  │ - Avatar storage │  │ - Events      │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### UserController

- **Package**: `com.techmap.profileservice.controller`
- **Endpoints**:
  - `POST /users/register` - User registration
  - `GET /users/my-profile` - Get current user profile
  - `PUT /users` - Update profile
  - `PATCH /users/remove/{id}` - Soft delete user
  - `POST /users/change-password` - Change password
  - `POST /users/forgot-password` - Forgot password
  - `POST /users/update-avatar` - Upload avatar
  - `GET /users` - Search users with pagination
  - `POST /users/bulk` - Get multiple users

### 2. Service Layer

#### UserService

```java
@Service
public class UserService {
    // Core operations
    UserResponse register(UserRequest request);
    UserResponse getMyProfile();
    UserResponse updateUser(UpdateProfileRequest request);
    void removeUser(String userId);

    // Password management
    void changePassword(String oldPassword, String newPassword);
    void forgotPassword(String email);

    // Avatar management
    String uploadAvatar(MultipartFile file);

    // Search operations
    PageResponse<UserEmailResponse> findUserByEmailAndUserId(
        int page, int size, String email, String userId
    );
    List<UserEmailResponse> getUserByIds(List<String> userIds);
}
```

### 3. Integration Clients

#### IdentityClient (Feign)

```java
@FeignClient(name = "identity-service", url = "${identity.service.url}")
public interface IdentityClient {
    @PostMapping("/auth/token")
    TokenResponse getToken(@RequestBody TokenRequest request);

    @PostMapping("/users")
    void createUser(@RequestBody UserCreationRequest request);

    @PutMapping("/users/{userId}")
    void updateUser(@PathVariable String userId,
                   @RequestBody UserUpdateRequest request);
}
```

#### FileRepository (Feign)

```java
@FeignClient(name = "file-service", url = "${file.service.url}")
public interface FileRepository {
    @PostMapping(value = "/media/upload",
                consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    FileUploadResponse uploadFile(@RequestPart("file") MultipartFile file);
}
```

## Data Models

### User Entity

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    private String userId;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(unique = true, nullable = false)
    private String email;

    private String firstName;
    private String lastName;

    @Enumerated(EnumType.STRING)
    private Gender gender;

    private LocalDate dateOfBirth;
    private String phoneNumber;

    @Column(length = 500)
    private String bio;

    private String avatarUrl;

    @Column(columnDefinition = "boolean default false")
    private boolean deleted;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

## Caching Strategy

### Redis Configuration

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

### Cache Usage

```java
@Cacheable(value = "users", key = "#userId")
public UserResponse getUserById(String userId) {
    // Database query only on cache miss
}

@CacheEvict(value = "users", key = "#userId")
public void updateUser(String userId, UpdateProfileRequest request) {
    // Cache invalidation on update
}
```

## Message Queue Integration

### RabbitMQ Configuration

```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public Queue userRegisteredQueue() {
        return new Queue("user.registered", true);
    }

    @Bean
    public Queue userUpdatedQueue() {
        return new Queue("user.updated", true);
    }

    @Bean
    public TopicExchange userExchange() {
        return new TopicExchange("user.events");
    }
}
```

### Event Publishing

```java
@Service
public class UserEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishUserRegistered(UserResponse user) {
        UserRegisteredEvent event = new UserRegisteredEvent(
            user.getUserId(),
            user.getEmail(),
            user.getUsername()
        );
        rabbitTemplate.convertAndSend(
            "user.events",
            "user.registered",
            event
        );
    }
}
```

## Security Configuration

### JWT Integration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/users/register",
                               "/users/forgot-password").permitAll()
                .requestMatchers("/users/**")
                    .hasAnyAuthority("STUDENT", "TEACHER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt())
            .build();
    }
}
```

## Error Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiResponse<?>> handleUserNotFound(
            UserNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(ApiResponse.builder()
                .code(1004)
                .message("User not found")
                .build());
    }

    @ExceptionHandler(EmailAlreadyExistsException.class)
    public ResponseEntity<ApiResponse<?>> handleEmailExists(
            EmailAlreadyExistsException ex) {
        return ResponseEntity.status(409)
            .body(ApiResponse.builder()
                .code(1005)
                .message("Email already exists")
                .build());
    }
}
```

## Performance Optimizations

### 1. Database Indexing

```sql
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_deleted ON users(deleted);
```

### 2. Redis Caching

- User profiles cached for 1 hour
- Cache invalidation on updates
- Distributed caching for scalability

### 3. Pagination

- Default page size: 10
- Maximum page size: 100
- Efficient offset pagination

### 4. Bulk Operations

- Batch user retrieval
- Single database query for multiple users
- Reduced network overhead

## Monitoring and Logging

### Logging Strategy

```java
@Slf4j
@Service
public class UserService {

    public UserResponse register(UserRequest request) {
        log.info("Registering new user: {}", request.getEmail());
        try {
            // Registration logic
            log.info("User registered successfully: {}", user.getUserId());
        } catch (Exception e) {
            log.error("Failed to register user: {}", request.getEmail(), e);
            throw e;
        }
    }
}
```

### Metrics

- Registration success/failure rates
- Profile update frequency
- Avatar upload success rates
- Cache hit/miss ratios
- API response times

## Design Patterns

### 1. Repository Pattern

- Data access abstraction
- JPA repository interfaces

### 2. Service Layer Pattern

- Business logic encapsulation
- Transaction management

### 3. DTO Pattern

- Request/Response separation
- Entity protection

### 4. Event-Driven Pattern

- Asynchronous communication
- Service decoupling

### 5. Circuit Breaker Pattern (Future)

- Resilient external service calls
- Fallback mechanisms
