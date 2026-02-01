# Getting Started Guide

## Prerequisites

Before you begin, ensure you have the following installed:

| Software | Version | Download                                  |
| -------- | ------- | ----------------------------------------- |
| Java JDK | 21+     | [Eclipse Temurin](https://adoptium.net/)  |
| Maven    | 3.8+    | [Apache Maven](https://maven.apache.org/) |
| Docker   | 24+     | [Docker Desktop](https://www.docker.com/) |
| Git      | 2.x     | [Git](https://git-scm.com/)               |

### Verify Installation

```bash
java -version    # Should show Java 21+
mvn -version     # Should show Maven 3.8+
docker --version # Should show Docker 24+
```

## Quick Start

### 1. Clone the Repository

```bash
git clone <repository-url>
cd spring-boot
```

### 2. Start Infrastructure Services

```bash
# Start Redis, RabbitMQ using Docker Compose
docker-compose up -d redis rabbitmq
```

Verify services are running:

```bash
docker ps
```

Expected output:

```
CONTAINER ID   IMAGE                           STATUS          PORTS
abc123         redis:6.0                       Up 2 minutes    0.0.0.0:6379->6379/tcp
def456         rabbitmq:4.1.6-management      Up 2 minutes    0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
```

### 3. Start MySQL (if not using Docker)

```bash
# Using Docker
docker run -d \
  --name mysql-memap \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=memap \
  -p 3306:3306 \
  mysql:8.0
```

### 4. Configure Environment

Each service requires environment variables. Create `.env` files:

**profile-service/.env**

```env
DB_HOST=localhost
DB_PORT=3306
DB_NAME=profile_db
DB_USERNAME=root
DB_PASSWORD=password
REDIS_HOST=localhost
REDIS_PORT=6379
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=admin
RABBITMQ_PASSWORD=admin
```

### 5. Run a Service

```bash
cd profile-service
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### 6. Verify Service is Running

```bash
curl http://localhost:8082/profile/actuator/health
```

Expected response:

```json
{
  "status": "UP"
}
```

## Running All Services

### Option 1: Docker Compose (Recommended)

```bash
# Build and run all services
docker-compose up -d --build

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### Option 2: Run Services Individually

Open multiple terminals and run each service:

**Terminal 1 - Identity Service:**

```bash
cd identity-service
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

**Terminal 2 - Profile Service:**

```bash
cd profile-service
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

**Terminal 3 - Roadmap Service:**

```bash
cd roadmap-service
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

## Service URLs

After starting services, access:

| Service  | Health Check                                   | Swagger UI                                     |
| -------- | ---------------------------------------------- | ---------------------------------------------- |
| Identity | http://localhost:8081/identity/actuator/health | http://localhost:8081/identity/swagger-ui.html |
| Profile  | http://localhost:8082/profile/actuator/health  | http://localhost:8082/profile/swagger-ui.html  |
| Roadmap  | http://localhost:8083/roadmap/actuator/health  | http://localhost:8083/roadmap/swagger-ui.html  |
| AI       | http://localhost:8084/ai/actuator/health       | http://localhost:8084/ai/swagger-ui.html       |
| File     | http://localhost:8085/file/actuator/health     | http://localhost:8085/file/swagger-ui.html     |

### Management UIs

| Tool                | URL                    | Credentials             |
| ------------------- | ---------------------- | ----------------------- |
| RabbitMQ Management | http://localhost:15672 | admin / admin           |
| MinIO Console       | http://localhost:9001  | minioadmin / minioadmin |

## Testing the API

### 1. Register a User

```bash
curl -X POST http://localhost:8082/profile/users/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "Password123!",
    "firstName": "Test",
    "lastName": "User"
  }'
```

### 2. Login

```bash
curl -X POST http://localhost:8081/identity/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "password": "Password123!"
  }' \
  -c cookies.txt
```

### 3. Get Profile (Authenticated)

```bash
curl http://localhost:8082/profile/users/my-profile \
  -b cookies.txt
```

## Development Workflow

### Making Changes

1. **Create feature branch:**

   ```bash
   git checkout -b feature/your-feature
   ```

2. **Make changes to service**

3. **Run tests:**

   ```bash
   cd profile-service
   ./mvnw test
   ```

4. **Build:**

   ```bash
   ./mvnw clean package -DskipTests
   ```

5. **Commit and push:**
   ```bash
   git add .
   git commit -m "feat: add new feature"
   git push origin feature/your-feature
   ```

### Hot Reload (Development)

Spring Boot DevTools enables hot reload:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

Changes to Java files will trigger automatic restart.

## Troubleshooting

### Service Won't Start

1. **Check if port is in use:**

   ```bash
   # Windows
   netstat -ano | findstr :8082

   # Linux/Mac
   lsof -i :8082
   ```

2. **Check database connection:**

   ```bash
   docker logs mysql-memap
   ```

3. **Check environment variables:**
   ```bash
   echo $DB_HOST
   ```

### Database Connection Issues

```bash
# Test MySQL connection
mysql -h localhost -u root -p

# Check MySQL is running
docker ps | grep mysql
```

### RabbitMQ Connection Issues

```bash
# Check RabbitMQ status
docker logs rabbitmq

# Access management UI
open http://localhost:15672
```

### Redis Connection Issues

```bash
# Test Redis connection
redis-cli ping
# Expected: PONG

# Check Redis logs
docker logs redis
```

## Next Steps

1. Read the [Architecture Documentation](ARCHITECTURE.md)
2. Explore individual service documentation
3. Review the [API Documentation](API.md)
4. Check the [Deployment Guide](DEPLOYMENT.md)
