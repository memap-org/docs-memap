# Roadmap Service Architecture

## Service Overview

The Roadmap Service is a comprehensive learning roadmap management system built with Spring Boot. It follows a layered architecture pattern with clear separation of concerns and integrates with multiple external services.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Layer                              │
│            (Frontend, Mobile Apps, External Services)               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │   API Gateway   │
                    │   (Port 8280)   │
                    └────────┬────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                      Controller Layer                               │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ RoadMapController│  │  ProgressTracking│  │   CategoryCtrl   │  │
│  │ - CRUD ops       │  │   Controller     │  │ - Categories     │  │
│  │ - Node/Edge mgmt │  │ - Track progress │  │                  │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ PermissionCtrl   │  │  ResourceCtrl    │  │  SocketHandler   │  │
│  │ - Access control │  │ - File upload    │  │ - WebSocket      │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                        Service Layer                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RoadMapService                                               │  │
│  │ - Roadmap business logic                                     │  │
│  │ - Node/Edge operations                                       │  │
│  │ - Permission validation                                      │  │
│  │ - File integration                                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ NodeProgressTrackingService                                  │  │
│  │ - Progress tracking logic                                    │  │
│  │ - Status updates (PENDING, IN_PROGRESS, DONE)               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RoadMapAccessPermissionService                               │  │
│  │ - Permission management                                      │  │
│  │ - Scope validation (PRIVATE, GROUP, PUBLIC)                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ FileService                                                  │  │
│  │ - File upload/download                                       │  │
│  │ - Integration with File Service                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                     Repository Layer                                │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ RoadMapRepository│  │ NodeTracking    │  │ CategoryRepo     │  │
│  │ - JPA interface  │  │  Repository     │  │                  │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌─────────────────┐                        │
│  │ PermissionRepo   │  │ MongoDB Repos   │                        │
│  │                  │  │ - File metadata │                        │
│  └──────────────────┘  └─────────────────┘                        │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────┐
│                       Database Layer                                │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐   │
│  │ MySQL (roadmap_service) │    │ MongoDB (file_metadata)      │   │
│  │ ┌─────────────────────┐ │    │ - File documents            │   │
│  │ │ roadmaps            │ │    └──────────────────────────────┘   │
│  │ │ nodes               │ │                                        │
│  │ │ edges               │ │    ┌──────────────────────────────┐   │
│  │ │ categories          │ │    │ Redis (Cache)                │   │
│  │ │ permissions         │ │    │ - Session cache              │   │
│  │ │ node_tracking       │ │    │ - Roadmap cache              │   │
│  │ └─────────────────────┘ │    └──────────────────────────────┘   │
│  └─────────────────────────┘                                        │
└─────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────┐
│                    External Services                                │
│  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │ Identity Service │  │ Profile Service │  │  File Service    │  │
│  │ - JWT validation │  │ - User profiles │  │ - File storage   │  │
│  │ - Keycloak       │  │                 │  │ - MinIO          │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
│  ┌──────────────────┐  ┌─────────────────┐                        │
│  │ RabbitMQ         │  │ Roadmap AI Svc  │                        │
│  │ - Async messages │  │ - AI generation │                        │
│  └──────────────────┘  └─────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### RoadMapController

**Responsibility**: Handle HTTP requests for roadmap CRUD operations and node/edge management

**Key Endpoints**:

- `POST /api/roadmap` - Create new roadmap
- `GET /api/roadmap/{id}` - Get roadmap by ID
- `GET /api/roadmap/my-roadmap` - Get user's roadmaps with pagination
- `GET /api/roadmap/accessible` - Get accessible roadmaps (owned + shared)
- `DELETE /api/roadmap/{id}` - Soft delete roadmap
- `PATCH /api/roadmap/node/{id}` - Update node
- `PATCH /api/roadmap/node/status/{id}` - Update node status
- `PATCH /api/roadmap/edge/{id}` - Update edge
- `PATCH /api/roadmap/info/{id}` - Update roadmap info
- `PATCH /api/roadmap/image/{id}` - Upload roadmap image
- `GET /api/roadmap/{id}/permissions` - Get permissions list

#### RoadmapProgressTrackingController

**Responsibility**: Handle progress tracking operations for users

**Key Endpoints**:

- `GET /api/roadmap-tracking-progress/{roadmapId}` - Get full progress
- `GET /api/roadmap-tracking-progress/{roadmapId}/{nodeId}` - Get node status
- `PATCH /api/roadmap-tracking-progress/{roadmapId}` - Update node tracking
- `POST /api/roadmap-tracking-progress/reset/{roadmapId}` - Reset progress
- `POST /api/roadmap-tracking-progress/done/{roadmapId}` - Mark all done

#### RoadmapCategoryController

**Responsibility**: Manage roadmap categories

**Key Endpoints**:

- `GET /api/roadmap-category` - Get all categories
- `POST /api/roadmap-category` - Create category

#### RoadMapAccessPermissionController

**Responsibility**: Manage roadmap access permissions and sharing

**Key Endpoints**:

- `PATCH /api/roadmap/scope/{id}` - Update roadmap scope
- `POST /api/roadmap/scope/user/{id}` - Add user permission
- `DELETE /api/roadmap/scope/user/{id}` - Remove user permission

#### RoadmapResourceController

**Responsibility**: Handle file uploads for roadmap resources

**Key Endpoints**:

- `POST /api/roadmap/resource/upload/file/{id}` - Upload file

#### SocketHandler

**Responsibility**: WebSocket handler for real-time collaborative editing

### 2. Service Layer

#### IRoadMapService / RoadMapServiceImpl

**Key Methods**:

```java
CreateRoadMapResponse create(CreateRoadMapRequest request);
RoadmapResponse getRoadmapById(String id);
PageResponse<RoadmapResponse> getMyRoadMap(int page, int size, EAccessStatusRoadmap scope, String search);
PageResponse<RoadmapResponse> getAccessibleRoadmaps(int page, int size, String search);
Boolean deleteRoadMap(String id);
Boolean updateANodeOfRoadmap(String roadmapId, NodeRequest nodeRequest);
Boolean updateStatusOfNode(String roadmapId, NodeStatusRequest request);
Boolean updateAEdgeOfRoadmap(String roadmapId, EdgeRequest edgeRequest);
Boolean updateInfomationOfRoadmap(String roadmapId, RoadmapInfoRequest request);
FileResponse uploadResourceForFile(String roadmapId, MultipartFile file);
FileResponse updateImageOfRoadmap(String roadmapId, MultipartFile file);
List<RoadMapAccessPermission> getAllPermissionOfRoadmap(String roadmapId);
```

**Business Logic**:

- Validate user permissions before operations
- Handle node and edge modifications
- Manage roadmap lifecycle
- Integrate with file storage
- Cache frequently accessed roadmaps

#### INodeProgressTrackingService

**Key Methods**:

```java
RoadmapTrackingProgressResponse getTracking(String roadmapId, String userId);
NodeTrackingResponse getStatusOfANode(String roadmapId, String userId, String nodeId);
Boolean updateTracking(String roadmapId, String userId, NodeTrackingUpdateRequest request);
Boolean resetTracking(String roadmapId, String userId);
Boolean markAllDone(String roadmapId, String userId);
```

**Business Logic**:

- Track individual node completion
- Calculate overall progress percentage
- Support status transitions (PENDING → IN_PROGRESS → DONE)
- Handle bulk operations (reset, mark all done)

#### IRoadMapAccessPermissionService

**Key Methods**:

```java
Boolean updateStatusRoadmapById(String roadmapId, UpdateStatusRoadmapRequest request);
UpdateRoadmapPermisisonResponse updateRoadMapPermission(String roadmapId, UpdateRoadmapPermissionRequest request);
RemoveRoadmapPermissionResponse removeRoadMapPermission(String roadmapId, RemoveRoadmapPermissionRequest request);
```

**Business Logic**:

- Validate scope changes (PRIVATE ↔ GROUP ↔ PUBLIC)
- Manage user permissions (OWNER, EDITOR, VIEWER)
- Enforce permission hierarchy
- Send notifications via RabbitMQ

### 3. Repository Layer

#### RoadMapRepository

```java
public interface RoadMapRepository extends JpaRepository<RoadMap, String> {
    Page<RoadMap> findByOwnerIdAndScopeAndDeletedFalse(String ownerId, EAccessStatusRoadmap scope, Pageable pageable);
    Page<RoadMap> findByOwnerIdAndDeletedFalse(String ownerId, Pageable pageable);
    List<RoadMap> findByOwnerIdOrPermissionsContaining(String ownerId, String userId);
    Optional<RoadMap> findByIdAndDeletedFalse(String id);
}
```

#### NodeProgressTrackingRepository

```java
public interface NodeProgressTrackingRepository extends JpaRepository<NodeProgressTracking, String> {
    List<NodeProgressTracking> findByRoadmapIdAndUserId(String roadmapId, String userId);
    Optional<NodeProgressTracking> findByRoadmapIdAndUserIdAndNodeId(String roadmapId, String userId, String nodeId);
    void deleteByRoadmapIdAndUserId(String roadmapId, String userId);
}
```

#### RoadMapAccessPermissionRepository

```java
public interface RoadMapAccessPermissionRepository extends JpaRepository<RoadMapAccessPermission, String> {
    List<RoadMapAccessPermission> findByRoadmapId(String roadmapId);
    Optional<RoadMapAccessPermission> findByRoadmapIdAndUserId(String roadmapId, String userId);
    void deleteByRoadmapIdAndUserId(String roadmapId, String userId);
}
```

### 4. Data Models

#### RoadMap Entity

```java
@Entity
@Table(name = "roadmaps")
public class RoadMap {
    @Id
    private String id;

    private String title;
    private String description;

    @Enumerated(EnumType.STRING)
    private EAccessStatusRoadmap scope; // PRIVATE, GROUP, PUBLIC

    private String ownerId;
    private String categoryId;
    private String imageUrl;

    @OneToMany(mappedBy = "roadmap", cascade = CascadeType.ALL)
    private List<Node> nodes;

    @OneToMany(mappedBy = "roadmap", cascade = CascadeType.ALL)
    private List<Edge> edges;

    @OneToMany(mappedBy = "roadmap", cascade = CascadeType.ALL)
    private List<RoadMapAccessPermission> permissions;

    private Boolean deleted = false;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

#### Node Entity

```java
@Entity
@Table(name = "nodes")
public class Node {
    @Id
    private String nodeId;

    private String label;
    private String type;

    @Embedded
    private Position position;

    @Embedded
    private NodeData data;

    @ManyToOne
    @JoinColumn(name = "roadmap_id")
    private RoadMap roadmap;
}
```

#### Edge Entity

```java
@Entity
@Table(name = "edges")
public class Edge {
    @Id
    private String edgeId;

    private String source;
    private String target;
    private String type;
    private String label;

    @ManyToOne
    @JoinColumn(name = "roadmap_id")
    private RoadMap roadmap;
}
```

#### NodeProgressTracking Entity

```java
@Entity
@Table(name = "node_tracking")
public class NodeProgressTracking {
    @Id
    private String id;

    private String roadmapId;
    private String userId;
    private String nodeId;

    @Enumerated(EnumType.STRING)
    private NodeStatus status; // PENDING, IN_PROGRESS, DONE, SKIPPED

    private LocalDateTime updatedAt;
}
```

#### RoadMapAccessPermission Entity

```java
@Entity
@Table(name = "roadmap_permissions")
public class RoadMapAccessPermission {
    @Id
    private String id;

    private String roadmapId;
    private String userId;

    @Enumerated(EnumType.STRING)
    private PermissionType permission; // OWNER, EDITOR, VIEWER

    private Boolean canEdit;
    private Boolean canView;

    @ManyToOne
    @JoinColumn(name = "roadmap_id")
    private RoadMap roadmap;
}
```

## Design Patterns

### 1. Repository Pattern

- JPA repositories for data access
- Clean separation between business logic and data access
- Support for complex queries with Spring Data JPA

### 2. Service Layer Pattern

- Business logic encapsulated in service classes
- Services use repositories for data access
- Transaction management with `@Transactional`

### 3. DTO Pattern

- Request DTOs for input validation
- Response DTOs for consistent API responses
- Mapper utilities for entity ↔ DTO conversion

### 4. Builder Pattern

- Used in DTOs and response objects
- Lombok `@Builder` annotation
- Fluent API for object construction

### 5. Soft Delete Pattern

- `deleted` flag instead of physical deletion
- Preserve data integrity and audit trail
- Filters in queries to exclude deleted records

## Security

### Authentication

- OAuth2 Resource Server with Spring Security
- JWT tokens issued by Keycloak
- Token validation on every request

### Authorization

- Role-based access control (RBAC)
- Permission checks in service layer
- Owner, Editor, Viewer permission levels

### Permission Validation Logic

```java
public boolean hasEditPermission(String roadmapId, String userId) {
    RoadMap roadmap = getRoadmap(roadmapId);

    // Owner always has edit permission
    if (roadmap.getOwnerId().equals(userId)) {
        return true;
    }

    // Check explicit permissions for GROUP scope
    if (roadmap.getScope() == EAccessStatusRoadmap.GROUP) {
        RoadMapAccessPermission permission = getPermission(roadmapId, userId);
        return permission != null && permission.getCanEdit();
    }

    // PUBLIC roadmaps: only owner can edit
    return false;
}
```

## Caching Strategy

### Redis Caching

- **Roadmap Cache**: Cache frequently accessed roadmaps
- **Category Cache**: Cache all categories (rarely changes)
- **Permission Cache**: Cache user permissions for quick validation
- **TTL**: 1 hour for roadmaps, 24 hours for categories

### Cache Invalidation

- Invalidate on update/delete operations
- Event-based invalidation via RabbitMQ

## Integration Points

### 1. File Service Integration

- Upload roadmap images
- Store resource files
- Generate presigned URLs for downloads

### 2. Profile Service Integration

- Fetch user profiles for permission management
- Validate user existence before granting permissions

### 3. Identity Service Integration

- JWT token validation
- Extract user ID from token claims

### 4. RabbitMQ Integration

- Publish events: RoadmapCreated, RoadmapUpdated, PermissionGranted
- Subscribe to events: UserDeleted (cleanup permissions)

### 5. Roadmap AI Service Integration

- Receive AI-generated roadmaps
- Import nodes and edges from AI service

## Database Schema

### Key Tables

- `roadmaps` - Main roadmap data
- `nodes` - Roadmap nodes with position and data
- `edges` - Connections between nodes
- `roadmap_categories` - Category definitions
- `roadmap_permissions` - User access permissions
- `node_tracking` - User progress tracking
- `file_metadata` (MongoDB) - File metadata

### Relationships

- RoadMap 1:N Nodes
- RoadMap 1:N Edges
- RoadMap 1:N Permissions
- RoadMap N:1 Category
- User M:N RoadMap (via permissions)

## Performance Considerations

### Pagination

- All list endpoints support pagination
- Default page size: 10
- Maximum page size: 100

### Query Optimization

- Indexed columns: ownerId, categoryId, scope
- Composite indexes for common queries
- Fetch JOIN for related entities

### Caching

- Redis for frequently accessed data
- Cache-aside pattern
- 5-minute TTL for volatile data

## Error Handling

### Exception Hierarchy

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(RoadmapNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse handleNotFound(RoadmapNotFoundException ex);

    @ExceptionHandler(PermissionDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ApiResponse handlePermissionDenied(PermissionDeniedException ex);

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse handleValidation(ValidationException ex);
}
```

## WebSocket Support

### Real-time Collaboration

- WebSocket endpoint: `ws://localhost:8083/road-map/ws`
- Broadcast node/edge updates to connected clients
- Support for concurrent editing
- Conflict resolution with last-write-wins

## Monitoring & Logging

### Logging

- SLF4J with Logback
- Log levels: DEBUG for development, INFO for production
- Structured logging with correlation IDs

### Metrics

- Spring Boot Actuator
- Custom metrics for roadmap operations
- Performance monitoring

## Future Enhancements

1. **Versioning**: Track roadmap versions and allow rollback
2. **Comments**: Add commenting system for nodes
3. **Templates**: Roadmap templates for common learning paths
4. **Export/Import**: Export roadmaps as JSON/PDF
5. **Analytics**: Detailed progress analytics and insights
6. **Notifications**: Real-time notifications for shared roadmaps
