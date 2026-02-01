# Identity Service Architecture

## Service Overview

The Identity Service is the authentication and authorization backbone of the MeMap platform. It follows a layered architecture pattern with clear separation of concerns.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│              (HTTP Requests with/without JWT)                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                     Controller Layer                            │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ UserController   │    │ AuthController   │                  │
│  │ - CRUD ops       │    │ - Login/Logout   │                  │
│  │ - User mgmt      │    │ - Token refresh  │                  │
│  └──────────────────┘    └──────────────────┘                  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Service Layer                             │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ UserService      │    │ AuthService      │                  │
│  │ - Business logic │    │ - JWT generation │                  │
│  │ - Validation     │    │ - Token validation│                 │
│  └──────────────────┘    └──────────────────┘                  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                    Repository Layer                             │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ UserRepository   │    │ TokenRepository  │                  │
│  │ - JPA interface  │    │ - Token storage  │                  │
│  └──────────────────┘    └──────────────────┘                  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Database Layer                            │
│                    MySQL (identity_service)                     │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │ users            │    │ tokens           │                  │
│  │ - credentials    │    │ - jwt_tokens     │                  │
│  │ - roles          │    │ - refresh_tokens │                  │
│  └──────────────────┘    └──────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### UserController

- **Responsibility**: Handle HTTP requests for user CRUD operations
- **Endpoints**:
  - `POST /users` - Create new user
  - `GET /users` - Get all users
  - `GET /users/{userId}` - Get user by ID
  - `PUT /users/{userId}` - Update user
  - `DELETE /users/{userId}` - Delete user

#### AuthController

- **Responsibility**: Handle authentication operations
- **Endpoints**:
  - `POST /auth/login` - User login
  - `POST /auth/logout` - User logout
  - `POST /auth/refresh` - Refresh access token
  - `POST /auth/introspect` - Validate token

### 2. Service Layer

#### UserService

- User creation and validation
- Password hashing (BCrypt)
- User profile management
- Role assignment

#### AuthService

- JWT token generation
- Token validation and parsing
- Refresh token handling
- OAuth2 integration (if configured)

### 3. Repository Layer

#### UserRepository

```java
public interface UserRepository extends JpaRepository<User, String> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

#### TokenRepository

```java
public interface TokenRepository extends JpaRepository<Token, String> {
    Optional<Token> findByRefreshToken(String refreshToken);
    void deleteByUserId(String userId);
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

    @Column(nullable = false)
    private String password;  // Hashed

    @ElementCollection(fetch = FetchType.EAGER)
    private Set<String> roles;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### Token Entity

```java
@Entity
@Table(name = "tokens")
public class Token {
    @Id
    private String tokenId;

    private String userId;
    private String refreshToken;
    private LocalDateTime expiresAt;
    private LocalDateTime createdAt;
}
```

## Security Architecture

### JWT Token Structure

```
Header:
{
  "alg": "HS512",
  "typ": "JWT"
}

Payload:
{
  "sub": "user-id",
  "username": "johndoe",
  "roles": ["STUDENT"],
  "iat": 1738149600,
  "exp": 1738150200
}

Signature:
HMACSHA512(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### Authentication Flow

```
┌──────────┐                                         ┌─────────────────┐
│  Client  │                                         │ Identity Service│
└────┬─────┘                                         └────────┬────────┘
     │                                                         │
     │ 1. POST /auth/login (username, password)               │
     │────────────────────────────────────────────────────────>│
     │                                                         │
     │                                   2. Validate credentials
     │                                   3. Generate JWT tokens
     │                                   4. Store refresh token
     │                                                         │
     │ 5. Response + Set-Cookie (access_token, refresh_token) │
     │<────────────────────────────────────────────────────────│
     │                                                         │
     │ 6. API Request + Cookie (access_token)                 │
     │────────────────────────────────────────────────────────>│
     │                                                         │
     │                                        7. Validate token
     │                                        8. Process request
     │                                                         │
     │ 9. Response                                             │
     │<────────────────────────────────────────────────────────│
     │                                                         │
```

### Token Refresh Flow

```
┌──────────┐                                         ┌─────────────────┐
│  Client  │                                         │ Identity Service│
└────┬─────┘                                         └────────┬────────┘
     │                                                         │
     │ 1. Access token expired                                │
     │                                                         │
     │ 2. POST /auth/refresh + Cookie (refresh_token)         │
     │────────────────────────────────────────────────────────>│
     │                                                         │
     │                                   3. Validate refresh token
     │                                   4. Generate new access token
     │                                                         │
     │ 5. Response + Set-Cookie (new access_token)            │
     │<────────────────────────────────────────────────────────│
     │                                                         │
```

## Configuration

### Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/login", "/auth/register").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### JWT Configuration

```yaml
jwt:
  secret: ${JWT_SECRET:your-secret-key}
  access-token-expiration: 600000 # 10 minutes
  refresh-token-expiration: 604800000 # 7 days
  issuer: identity-service
```

## Design Patterns

### 1. Repository Pattern

- Abstracts data access logic
- Provides clean interface for data operations

### 2. Service Layer Pattern

- Encapsulates business logic
- Separates concerns from controllers

### 3. DTO Pattern

- Data Transfer Objects for request/response
- Prevents exposure of entity internals

### 4. Builder Pattern

- Used in DTOs for flexible object creation
- Improves code readability

## Error Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InvalidCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleInvalidCredentials() {
        return ResponseEntity.status(401)
            .body(new ErrorResponse(401, "Invalid credentials"));
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound() {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(404, "User not found"));
    }
}
```

## Performance Considerations

### 1. Password Hashing

- Uses BCrypt with strength factor 12
- Async hashing for non-blocking operations

### 2. Token Validation

- In-memory cache for frequently validated tokens
- Redis integration for distributed caching (future)

### 3. Database Optimization

- Indexed username and email columns
- Connection pooling (HikariCP)

## Monitoring and Logging

### Logging Strategy

- **INFO**: Authentication success/failure
- **WARN**: Invalid token attempts
- **ERROR**: System failures
- **DEBUG**: Detailed request/response logs (dev only)

### Metrics

- Login success/failure rates
- Token generation time
- API response times
- Active user sessions
