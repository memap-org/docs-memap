# gRPC Troubleshooting Guide

This guide covers common issues and solutions for gRPC communication between the Learning Service and Roadmap Service.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│               Learning Service (Port 8089)                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              RoadmapGrpcClientImpl                       │   │
│  │  • getRoadmapName(roadmapId) → single lookup            │   │
│  │  • getRoadmapNames(roadmapIds) → batch lookup           │   │
│  └─────────────────────┬───────────────────────────────────┘   │
└────────────────────────┼────────────────────────────────────────┘
                         │ gRPC (HTTP/2, Port 9083)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│               Roadmap Service (Port 8083)                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              RoadmapGrpcService (Port 9083)              │   │
│  │  • GetRoadmapName(request) → roadmap name               │   │
│  │  • GetRoadmapNames(request) → map of names              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Common Issues and Solutions

### 1. Connection Refused

**Symptoms:**

```
io.grpc.StatusRuntimeException: UNAVAILABLE: io exception
Caused by: java.net.ConnectException: Connection refused
```

**Causes & Solutions:**

| Cause                       | Solution                                             |
| --------------------------- | ---------------------------------------------------- |
| Roadmap Service not running | Start the Roadmap Service: `./mvnw spring-boot:run`  |
| Wrong port                  | Verify gRPC server port is `9083` in Roadmap Service |
| Firewall blocking           | Allow port 9083 in firewall settings                 |
| Docker networking           | Use service name instead of `localhost` in Docker    |

**Verification Steps:**

```bash
# Check if Roadmap Service gRPC port is listening
netstat -an | findstr 9083    # Windows
netstat -an | grep 9083       # Linux/Mac

# Test gRPC connectivity with grpcurl
grpcurl -plaintext localhost:9083 list
```

### 2. Deadline Exceeded (Timeout)

**Symptoms:**

```
io.grpc.StatusRuntimeException: DEADLINE_EXCEEDED: deadline exceeded after 4.999s
```

**Causes & Solutions:**

| Cause                           | Solution                                  |
| ------------------------------- | ----------------------------------------- |
| Roadmap Service slow to respond | Check service logs for performance issues |
| Network latency                 | Increase deadline timeout                 |
| Large batch request             | Reduce batch size or increase timeout     |

**Configuration Fix:**

```yaml
# learning-service/application.yml
grpc:
  client:
    roadmap-service:
      deadline: 10000 # Increase to 10 seconds
```

### 3. Roadmap Name Returns Empty String

**Symptoms:**

- Quiz responses have `roadmapName: ""`
- No errors in logs

**Causes & Solutions:**

| Cause                       | Solution                                             |
| --------------------------- | ---------------------------------------------------- |
| Invalid roadmapId           | Verify the roadmapId exists in Roadmap Service       |
| gRPC graceful fallback      | Check logs for "Failed to get roadmap name" warnings |
| Roadmap Service unavailable | Ensure Roadmap Service is running and healthy        |

**Debug Steps:**

```java
// Add debug logging in RoadmapGrpcClientImpl
log.debug("Calling gRPC getRoadmapName for roadmapId: {}", roadmapId);
```

### 4. Channel Not Found

**Symptoms:**

```
io.grpc.StatusRuntimeException: Channel 'roadmap-service' not found
```

**Solution:**
Verify gRPC client configuration matches the channel name:

```yaml
# learning-service/application.yml
grpc:
  client:
    roadmap-service: # This name must match @GrpcClient("roadmap-service")
      address: static://localhost:9083
```

```java
// In RoadmapGrpcClientImpl
@GrpcClient("roadmap-service")  // Must match YAML key
private Channel channel;
```

### 5. Proto Compilation Errors

**Symptoms:**

```
Cannot find symbol: RoadmapLookupServiceGrpc
Cannot find symbol: GetRoadmapNameRequest
```

**Solution:**

```bash
# Recompile protobuf definitions
cd learning-serice
./mvnw clean compile

cd ../roadmap-service
./mvnw clean compile
```

**Verify proto file location:**

```
learning-serice/src/main/proto/roadmap/roadmap_lookup.proto
roadmap-service/src/main/proto/roadmap/roadmap_lookup.proto
```

### 6. SSL/TLS Issues

**Symptoms:**

```
io.grpc.StatusRuntimeException: UNAVAILABLE: Network closed for unknown reason
```

**Solution for Development:**

```yaml
# Use plaintext for local development
grpc:
  client:
    roadmap-service:
      negotiation-type: plaintext
```

**For Production:**

```yaml
grpc:
  client:
    roadmap-service:
      negotiation-type: tls
      security:
        trust-cert-collection: classpath:certs/ca.crt
```

## Health Check Endpoints

### Verify Roadmap Service gRPC Server

```bash
# Using grpcurl
grpcurl -plaintext localhost:9083 grpc.health.v1.Health/Check

# Using curl for REST health
curl http://localhost:8083/road-map/actuator/health
```

### Verify Learning Service gRPC Client

```bash
# Check Learning Service health
curl http://localhost:8089/learning/actuator/health

# Create a quiz with roadmapId and check if roadmapName is populated
curl -X POST http://localhost:8089/learning/api/v1/quizzes \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"title":"Test Quiz","roadmapId":"<valid-roadmap-id>"}'
```

## Configuration Reference

### Learning Service (gRPC Client)

```yaml
grpc:
  client:
    roadmap-service:
      address: static://localhost:9083 # or dns:///roadmap-service:9083
      negotiation-type: plaintext # or tls for production
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-timeout: 10s
      deadline: 5000 # milliseconds
```

### Roadmap Service (gRPC Server)

```yaml
grpc:
  server:
    port: 9083
    # security:
    #   enabled: false  # For development
```

## Docker Compose Configuration

When running in Docker, update the address to use service names:

```yaml
# docker-compose.yml
services:
  learning-service:
    environment:
      - GRPC_CLIENT_ROADMAP_ADDRESS=static://roadmap-service:9083
    depends_on:
      - roadmap-service

  roadmap-service:
    ports:
      - '8083:8083'
      - '9083:9083' # Expose gRPC port
```

## Logging Configuration

Enable gRPC debug logging for troubleshooting:

```yaml
# application.yml
logging:
  level:
    io.grpc: DEBUG
    net.devh.boot.grpc: DEBUG
    com.memap.learningservice.client.grpc: DEBUG
```

## Performance Tuning

### Connection Pooling

```yaml
grpc:
  client:
    roadmap-service:
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-timeout: 10s
      keep-alive-without-calls: true
```

### Batch Operations

For listing quizzes, the `getRoadmapNames()` batch operation is more efficient:

```java
// Instead of N individual calls:
for (Quiz quiz : quizzes) {
    String name = grpcClient.getRoadmapName(quiz.getRoadmapId()); // N calls
}

// Use batch operation:
List<String> roadmapIds = quizzes.stream().map(Quiz::getRoadmapId).toList();
Map<String, String> names = grpcClient.getRoadmapNames(roadmapIds); // 1 call
```

## Quick Diagnostics Checklist

- [ ] Roadmap Service is running on port 8083
- [ ] gRPC server is listening on port 9083
- [ ] Learning Service can reach Roadmap Service network
- [ ] Proto files are compiled (run `mvn compile`)
- [ ] gRPC client configuration matches server settings
- [ ] Firewall allows port 9083
- [ ] Correct channel name in `@GrpcClient` annotation

## Related Documentation

- [Learning Service README](./learning-service/README.md)
- [Roadmap Service README](./roadmap-service/README.md)
- [Architecture Overview](./ARCHITECTURE.md)
- [grpc-spring-boot-starter docs](https://yidongnan.github.io/grpc-spring-boot-starter/)
