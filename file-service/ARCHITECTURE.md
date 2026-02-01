# File Service Architecture

## Service Overview

The File Service is a lightweight microservice for file storage and retrieval using MinIO object storage. It follows a simple layered architecture with clean separation between HTTP handling, business logic, and storage operations.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│              (Roadmap Service, Profile Service, etc.)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                     Controller Layer                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ FileController                                           │  │
│  │ - POST /file (upload)                                    │  │
│  │ - GET /file/download (download)                          │  │
│  │ - GET /file/download-link (presigned URL)               │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                       Service Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ FileService                                              │  │
│  │ - uploadFile(MultipartFile)                             │  │
│  │ - downloadFile(String fileName): InputStream            │  │
│  │ - getPresignedUrl(String fileName): String              │  │
│  │                                                          │  │
│  │ Business Logic:                                          │  │
│  │ - Validate file                                          │  │
│  │ - Generate unique filename                               │  │
│  │ - Store metadata                                         │  │
│  │ - Interact with MinIO                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                    Repository Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ FileRepository                                           │  │
│  │ - JPA Repository for File entity                         │  │
│  │ - findByFileName(String): Optional<File>                │  │
│  │ - findByUserId(String): List<File>                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                      Storage Layer                              │
│  ┌─────────────────────────┐    ┌──────────────────────────┐   │
│  │ MySQL (metadata)        │    │ MinIO (object storage)   │   │
│  │ ┌─────────────────────┐ │    │ ┌──────────────────────┐ │   │
│  │ │ files               │ │    │ │ Bucket:              │ │   │
│  │ │ - id                │ │    │ │ memap-file-service   │ │   │
│  │ │ - filename          │ │    │ │                      │ │   │
│  │ │ - original_name     │ │    │ │ Objects:             │ │   │
│  │ │ - content_type      │ │    │ │ - document.pdf       │ │   │
│  │ │ - size              │ │    │ │ - image.jpg          │ │   │
│  │ │ - user_id           │ │    │ │ - video.mp4          │ │   │
│  │ │ - created_at        │ │    │ └──────────────────────┘ │   │
│  │ └─────────────────────┘ │    └──────────────────────────┘   │
│  └─────────────────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### FileController

**Responsibility**: Handle HTTP requests for file operations

**Endpoints**:

```java
@RestController
@RequestMapping("/file")
public class FileController {

    @PostMapping
    public void upload(MultipartFile file) throws Exception;

    @GetMapping("/download")
    public ResponseEntity<Resource> download(@RequestParam String fileName);

    @GetMapping("/download-link")
    public ResponseEntity<String> getDownloadLink(@RequestParam String fileName);
}
```

**Key Features**:

- Multipart file upload handling
- Stream-based file download
- Presigned URL generation
- Error handling and response formatting

### 2. Service Layer

#### FileService / FileServiceImpl

**Key Methods**:

```java
public interface FileService {
    void uploadFile(MultipartFile file) throws Exception;
    InputStream downloadFile(String fileName) throws Exception;
    String getPresignedUrl(String fileName) throws Exception;
}
```

**Business Logic**:

##### Upload File

```java
public void uploadFile(MultipartFile file) throws Exception {
    // 1. Validate file
    if (file.isEmpty()) {
        throw new IllegalArgumentException("File is empty");
    }

    // 2. Get file metadata
    String originalFilename = file.getOriginalFilename();
    String contentType = file.getContentType();
    long size = file.getSize();

    // 3. Upload to MinIO
    minioClient.putObject(
        PutObjectArgs.builder()
            .bucket(bucketName)
            .object(originalFilename)
            .stream(file.getInputStream(), size, -1)
            .contentType(contentType)
            .build()
    );

    // 4. Save metadata to database
    File fileEntity = new File();
    fileEntity.setFileName(originalFilename);
    fileEntity.setContentType(contentType);
    fileEntity.setSize(size);
    fileEntity.setUserId(getCurrentUserId());
    fileRepository.save(fileEntity);
}
```

##### Download File

```java
public InputStream downloadFile(String fileName) throws Exception {
    // Get object from MinIO
    return minioClient.getObject(
        GetObjectArgs.builder()
            .bucket(bucketName)
            .object(fileName)
            .build()
    );
}
```

##### Get Presigned URL

```java
public String getPresignedUrl(String fileName) throws Exception {
    // Generate presigned URL (valid for 1 hour)
    return minioClient.getPresignedObjectUrl(
        GetPresignedObjectUrlArgs.builder()
            .bucket(bucketName)
            .object(fileName)
            .method(Method.GET)
            .expiry(3600) // 1 hour
            .build()
    );
}
```

### 3. Configuration

#### MinioConfig

```java
@Configuration
public class MinioConfig {
    @Value("${minio.endpoint}")
    private String endpoint;

    @Value("${minio.access-key}")
    private String accessKey;

    @Value("${minio.secret-key}")
    private String secretKey;

    @Value("${minio.bucket}")
    private String bucketName;

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
            .endpoint(endpoint)
            .credentials(accessKey, secretKey)
            .build();
    }

    @PostConstruct
    public void initializeBucket() throws Exception {
        // Create bucket if not exists
        boolean found = minioClient.bucketExists(
            BucketExistsArgs.builder().bucket(bucketName).build()
        );

        if (!found) {
            minioClient.makeBucket(
                MakeBucketArgs.builder().bucket(bucketName).build()
            );
        }
    }
}
```

### 4. Data Models

#### File Entity

```java
@Entity
@Table(name = "files")
public class File extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String fileName;

    private String originalName;

    @Column(nullable = false)
    private String contentType;

    @Column(nullable = false)
    private Long size;

    private String userId;

    private String storagePath;

    @Column(nullable = false, updatable = false)
    private LocalDateTime uploadedAt;
}
```

#### BaseEntity

```java
@MappedSuperclass
public abstract class BaseEntity {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

### 5. Repository Layer

#### FileRepository

```java
public interface FileRepository extends JpaRepository<File, String> {
    Optional<File> findByFileName(String fileName);
    List<File> findByUserId(String userId);
    List<File> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    @Query("SELECT SUM(f.size) FROM File f WHERE f.userId = :userId")
    Long getTotalSizeByUserId(@Param("userId") String userId);
}
```

## Design Patterns

### 1. Repository Pattern

- Clean separation between data access and business logic
- JPA repositories for database operations
- Type-safe query methods

### 2. Service Layer Pattern

- Business logic encapsulated in service classes
- Services use repositories for database access
- Services use MinioClient for object storage

### 3. Dependency Injection

- Spring's dependency injection for loose coupling
- Constructor injection preferred over field injection

### 4. Builder Pattern

- MinIO client operations use builder pattern
- Fluent API for constructing complex objects

## Storage Strategy

### File Organization

```
memap-file-service/
├── roadmaps/
│   └── {roadmapId}/
│       ├── images/
│       │   └── thumbnail.jpg
│       └── resources/
│           └── document.pdf
├── profiles/
│   └── {userId}/
│       └── avatar.jpg
├── quizzes/
│   └── {quizId}/
│       └── attachment.pdf
└── temp/
    └── {tempId}/
        └── temporary-file.ext
```

### Path Generation

```java
public String generateStoragePath(String category, String entityId, String fileName) {
    return String.format("%s/%s/%s", category, entityId, fileName);
}
```

## Security

### Authentication

- JWT-based authentication via Spring Security
- Token validation on every request
- User ID extracted from token claims

### Authorization

- Users can only access their own files
- Admin role can access all files
- Presigned URLs provide temporary public access

### File Validation

```java
public void validateFile(MultipartFile file) {
    // Check if file is empty
    if (file.isEmpty()) {
        throw new IllegalArgumentException("File is empty");
    }

    // Check file size
    if (file.getSize() > MAX_FILE_SIZE) {
        throw new IllegalArgumentException("File too large");
    }

    // Check content type
    String contentType = file.getContentType();
    if (!ALLOWED_CONTENT_TYPES.contains(contentType)) {
        throw new IllegalArgumentException("File type not allowed");
    }
}
```

## Database Schema

### Files Table

```sql
CREATE TABLE files (
    id VARCHAR(36) PRIMARY KEY,
    file_name VARCHAR(255) NOT NULL,
    original_name VARCHAR(255),
    content_type VARCHAR(100) NOT NULL,
    size BIGINT NOT NULL,
    user_id VARCHAR(36),
    storage_path VARCHAR(500),
    uploaded_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP
);

CREATE INDEX idx_file_name ON files(file_name);
CREATE INDEX idx_user_id ON files(user_id);
CREATE INDEX idx_uploaded_at ON files(uploaded_at);
```

## MinIO Integration

### Client Configuration

```yaml
minio:
  endpoint: http://localhost:9000
  access-key: admin123
  secret-key: admin123
  bucket: memap-file-service
```

### Bucket Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": ["*"] },
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::memap-file-service/public/*"]
    }
  ]
}
```

### Benefits of MinIO

1. **S3-Compatible**: Standard S3 API
2. **Scalable**: Horizontal scaling support
3. **High Performance**: Optimized for large files
4. **Erasure Coding**: Data protection and redundancy
5. **Versioning**: Object versioning support
6. **Encryption**: Server-side encryption

## Error Handling

### Exception Hierarchy

```java
public class FileStorageException extends RuntimeException {
    public FileStorageException(String message) {
        super(message);
    }

    public FileStorageException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class FileNotFoundException extends RuntimeException {
    public FileNotFoundException(String fileName) {
        super("File not found: " + fileName);
    }
}
```

### Global Exception Handler

```java
@ControllerAdvice
public class FileExceptionHandler {

    @ExceptionHandler(FileStorageException.class)
    public ResponseEntity<String> handleStorageException(FileStorageException ex) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body("Error storing file: " + ex.getMessage());
    }

    @ExceptionHandler(FileNotFoundException.class)
    public ResponseEntity<String> handleNotFoundException(FileNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ex.getMessage());
    }
}
```

## Performance Considerations

### Streaming

- Use streaming for file upload/download
- Avoid loading entire file into memory
- Support for large file transfers

### Connection Pooling

- MinIO client connection pooling
- Database connection pooling with HikariCP

### Async Operations

```java
@Async
public CompletableFuture<Void> uploadFileAsync(MultipartFile file) {
    uploadFile(file);
    return CompletableFuture.completedFuture(null);
}
```

## Monitoring & Logging

### Logging

```java
@Slf4j
@Service
public class FileServiceImpl implements FileService {

    public void uploadFile(MultipartFile file) {
        log.info("Uploading file: {}", file.getOriginalFilename());
        try {
            // Upload logic
            log.info("File uploaded successfully: {}", file.getOriginalFilename());
        } catch (Exception e) {
            log.error("Error uploading file: {}", file.getOriginalFilename(), e);
            throw new FileStorageException("Failed to upload file", e);
        }
    }
}
```

### Metrics

- File upload count
- File download count
- Storage usage per user
- Average file size
- Error rate

## Migration Management

### Flyway

Database schema migrations managed with Flyway:

```sql
-- V1__create_files_table.sql
CREATE TABLE files (
    id VARCHAR(36) PRIMARY KEY,
    file_name VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    size BIGINT NOT NULL,
    user_id VARCHAR(36),
    created_at TIMESTAMP NOT NULL
);
```

## Integration Points

### 1. Roadmap Service

- Upload roadmap thumbnail images
- Upload roadmap resource files
- Receive file URLs and metadata

### 2. Profile Service

- Upload user avatar images
- Update profile pictures

### 3. Learning Service

- Upload quiz attachments
- Store course materials

## Future Enhancements

1. **File Versioning**: Track multiple versions of the same file
2. **Thumbnail Generation**: Auto-generate thumbnails for images/videos
3. **File Compression**: Compress large files automatically
4. **CDN Integration**: Serve files via CDN for faster delivery
5. **Virus Scanning**: Integrate antivirus scanning
6. **File Sharing**: Share files with other users with permissions
7. **File Search**: Full-text search in file metadata
8. **Batch Upload**: Support uploading multiple files at once
9. **Resume Upload**: Support resumable uploads for large files
10. **Storage Analytics**: Detailed storage usage analytics and reporting
