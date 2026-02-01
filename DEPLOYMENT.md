# Deployment Guide

## Table of Contents

- [Local Development](#local-development)
- [Docker Deployment](#docker-deployment)
- [Production Deployment](#production-deployment)
- [CI/CD Pipeline](#cicd-pipeline)

---

## Local Development

### Prerequisites

- Java 21
- Maven 3.8+
- Docker Desktop
- IDE (IntelliJ IDEA / VS Code)

### Setup Steps

1. **Start infrastructure:**

   ```bash
   docker-compose up -d redis rabbitmq
   ```

2. **Start MySQL:**

   ```bash
   docker run -d --name mysql-memap \
     -e MYSQL_ROOT_PASSWORD=password \
     -e MYSQL_DATABASE=memap \
     -p 3306:3306 \
     mysql:8.0
   ```

3. **Run service:**
   ```bash
   cd profile-service
   ./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
   ```

---

## Docker Deployment

### Build Images

**Single service:**

```bash
cd profile-service
docker build -t memap/profile-service:latest .
```

**All services:**

```bash
docker-compose build
```

### Docker Compose Configuration

**docker-compose.yml:**

```yaml
version: '3.9'

services:
  # Application Services
  identity-service:
    build: ./identity-service
    container_name: identity-service
    ports:
      - '8081:8081'
    env_file:
      - ./identity-service/.env
    depends_on:
      - mysql
      - redis
    restart: unless-stopped
    networks:
      - memap-network

  profile-service:
    build: ./profile-service
    container_name: profile-service
    ports:
      - '8082:8082'
    env_file:
      - ./profile-service/.env
    depends_on:
      - mysql
      - redis
      - rabbitmq
    restart: unless-stopped
    networks:
      - memap-network

  roadmap-service:
    build: ./roadmap-service
    container_name: roadmap-service
    ports:
      - '8083:8083'
    env_file:
      - ./roadmap-service/.env
    depends_on:
      - mysql
      - rabbitmq
    restart: unless-stopped
    networks:
      - memap-network

  roadmap-ai-service:
    build: ./roadmap-ai-service
    container_name: roadmap-ai-service
    ports:
      - '8084:8084'
    env_file:
      - ./roadmap-ai-service/.env
    depends_on:
      - mongodb
    restart: unless-stopped
    networks:
      - memap-network

  file-service:
    build: ./file-service
    container_name: file-service
    ports:
      - '8085:8085'
    env_file:
      - ./file-service/.env
    depends_on:
      - mysql
      - minio
    restart: unless-stopped
    networks:
      - memap-network

  # Infrastructure Services
  mysql:
    image: mysql:8.0
    container_name: memap-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-password}
    ports:
      - '3306:3306'
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - memap-network

  mongodb:
    image: mongo:6.0
    container_name: memap-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    ports:
      - '27017:27017'
    volumes:
      - mongo-data:/data/db
    networks:
      - memap-network

  redis:
    image: redis:6.0
    container_name: memap-redis
    ports:
      - '6379:6379'
    command: ['redis-server', '--appendonly', 'yes']
    volumes:
      - redis-data:/data
    networks:
      - memap-network

  rabbitmq:
    image: rabbitmq:3-management
    container_name: memap-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
    ports:
      - '5672:5672'
      - '15672:15672'
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - memap-network

  minio:
    image: minio/minio:latest
    container_name: memap-minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - '9000:9000'
      - '9001:9001'
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data
    networks:
      - memap-network

networks:
  memap-network:
    driver: bridge

volumes:
  mysql-data:
  mongo-data:
  redis-data:
  rabbitmq-data:
  minio-data:
```

### Running with Docker Compose

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f profile-service

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

---

## Production Deployment

### Environment Variables

Create secure `.env` files for production:

```env
# Database
DB_HOST=prod-mysql.example.com
DB_PORT=3306
DB_NAME=memap_prod
DB_USERNAME=memap_user
DB_PASSWORD=<secure-password>

# Redis
REDIS_HOST=prod-redis.example.com
REDIS_PORT=6379
REDIS_PASSWORD=<secure-password>

# RabbitMQ
RABBITMQ_HOST=prod-rabbitmq.example.com
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=memap_user
RABBITMQ_PASSWORD=<secure-password>

# JWT
JWT_SECRET=<256-bit-secret>
JWT_EXPIRY=600

# AI Service
OPENAI_API_KEY=<api-key>
```

### JVM Configuration

```bash
java -jar \
  -Xms512m \
  -Xmx1g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Dspring.profiles.active=prod \
  app.jar
```

### Kubernetes Deployment

**deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: profile-service
  namespace: memap
spec:
  replicas: 3
  selector:
    matchLabels:
      app: profile-service
  template:
    metadata:
      labels:
        app: profile-service
    spec:
      containers:
        - name: profile-service
          image: memap/profile-service:latest
          ports:
            - containerPort: 8082
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: 'prod'
          envFrom:
            - secretRef:
                name: profile-service-secrets
          resources:
            requests:
              memory: '512Mi'
              cpu: '250m'
            limits:
              memory: '1Gi'
              cpu: '500m'
          livenessProbe:
            httpGet:
              path: /profile/actuator/health/liveness
              port: 8082
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /profile/actuator/health/readiness
              port: 8082
            initialDelaySeconds: 30
            periodSeconds: 5
```

**service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: profile-service
  namespace: memap
spec:
  selector:
    app: profile-service
  ports:
    - port: 8082
      targetPort: 8082
  type: ClusterIP
```

**ingress.yaml:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: memap-ingress
  namespace: memap
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.memap.com
      http:
        paths:
          - path: /profile(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: profile-service
                port:
                  number: 8082
          - path: /roadmap(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: roadmap-service
                port:
                  number: 8083
```

---

## CI/CD Pipeline

### GitHub Actions Example

**.github/workflows/deploy.yml:**

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build with Maven
        run: |
          cd profile-service
          mvn clean package -DskipTests

      - name: Build Docker image
        run: |
          cd profile-service
          docker build -t memap/profile-service:${{ github.sha }} .

      - name: Push to Registry
        run: |
          docker push memap/profile-service:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/profile-service \
            profile-service=memap/profile-service:${{ github.sha }} \
            -n memap
```

---

## Health Checks

### Endpoints

| Endpoint                     | Purpose              |
| ---------------------------- | -------------------- |
| `/actuator/health`           | Overall health       |
| `/actuator/health/liveness`  | Kubernetes liveness  |
| `/actuator/health/readiness` | Kubernetes readiness |

### Monitoring Commands

```bash
# Check service health
curl http://localhost:8082/profile/actuator/health

# Check all services
for port in 8081 8082 8083 8084 8085; do
  echo "Port $port:"
  curl -s http://localhost:$port/actuator/health | jq .status
done
```

---

## Rollback Procedure

### Docker Compose

```bash
# Stop current version
docker-compose down

# Pull previous version
docker pull memap/profile-service:previous-tag

# Update docker-compose.yml or use specific tag
docker-compose up -d
```

### Kubernetes

```bash
# Rollback to previous deployment
kubectl rollout undo deployment/profile-service -n memap

# Rollback to specific revision
kubectl rollout undo deployment/profile-service --to-revision=2 -n memap
```
