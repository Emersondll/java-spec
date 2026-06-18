# Docker & Deployment - Java 26 Spring Boot

## Docker Image Construction

### Multi-Stage Build (Production Optimized)

```dockerfile
# Dockerfile - Multi-stage build for minimal image size
# Stage 1: Build stage - compiles and packages application

FROM maven:3.9.9-eclipse-temurin-26-alpine AS builder

WORKDIR /app

# Copy only pom.xml first for better Docker layer caching
# Maven dependencies cached in layer unless pom.xml changes
COPY pom.xml .

# Download dependencies (separate layer)
RUN mvn dependency:resolve-plugins dependency:resolve

# Copy source code
COPY src ./src

# Build application
RUN mvn clean package -DskipTests \
    && ls -la target/

# Stage 2: Runtime stage - minimal production image

FROM eclipse-temurin:26-jre-alpine

LABEL maintainer="Engineering Team"
LABEL version="1.0"
LABEL description="User Service - Spring Boot Java 26"

WORKDIR /app

# Create non-root user for security (Alpine uses addgroup/adduser)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy JAR from builder stage
COPY --from=builder /app/target/user-service-*.jar application.jar

# Change ownership to non-root user
RUN chown appuser:appgroup /app

# Switch to non-root user
USER appuser

# Health check using wget (available in Alpine by default)
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget -q -O /dev/null http://localhost:8080/actuator/health/liveness || exit 1

# Port exposure (application default)
EXPOSE 8080

# Memory settings for container
ENV JVM_OPTS="\
    -Xms512m \
    -Xmx1024m \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -Dfile.encoding=UTF-8"

ENV SPRING_PROFILES_ACTIVE=prod

# Start application
ENTRYPOINT ["java", "-jar", "application.jar"]
```

### Performance Optimized Layers

```dockerfile
# Dockerfile with layer optimization for smaller image

FROM eclipse-temurin:26-jre-alpine AS production

WORKDIR /app

# Install curl for healthchecks — Alpine uses apk, not apt-get
RUN apk add --no-cache curl

# Copy and verify JAR
COPY target/user-service-1.0.0.jar ./application.jar

# Create non-root user (Alpine)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

ENV JVM_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC"

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health/liveness || exit 1

ENTRYPOINT ["java", "-jar", "application.jar"]
```

## Docker Compose for Local Development

```yaml
# docker-compose.yml - Local development environment (Docker Compose — no Kubernetes)

version: '3.9'

services:
  # MongoDB Database (database-per-service pattern — one instance per service)
  mongo:
    image: mongo:7.0
    container_name: user-service-db
    environment:
      MONGO_INITDB_DATABASE: user_service
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - user-service-network

  # Spring Boot Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: user-service-app
    depends_on:
      mongo:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATA_MONGODB_URI: mongodb://mongo:27017/user_service
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health/liveness"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s
    networks:
      - user-service-network

  # Prometheus for metrics collection
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    environment:
      TZ: UTC
    volumes:
      - ./docker/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - user-service-network

  # Grafana for dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./docker/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./docker/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - user-service-network

networks:
  user-service-network:
    driver: bridge

volumes:
  mongo_data:
  prometheus_data:
  grafana_data:
```

## CI/CD Pipeline

> The complete CI/CD pipeline configuration (GitHub Actions, secrets management, Docker image build and push) is documented in **[24-CICD-SECRETS.md](24-CICD-SECRETS.md)**.
>
> Key steps covered there:
> - `mvn verify` with MongoDB service container
> - Docker multi-stage build and push to registry
> - Secret injection via GitHub Secrets (`${{ secrets.X }}`)
> - Image vulnerability scan (Trivy) before push
> - ArgoCD GitOps deployment flow

## Health Checks and Monitoring

### Docker Health Check

```dockerfile
# In Dockerfile — wget is available on Alpine by default; or use curl after apk add --no-cache curl
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget -q -O /dev/null http://localhost:8080/actuator/health/liveness || exit 1
```

### Docker Compose Health Probes

```yaml
# In docker-compose.yml service definition
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health/liveness"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 40s
```

## Environment Configuration

> See also: **[19-SPRING-PROFILES.md](19-SPRING-PROFILES.md)** — authoritative reference for all profile-specific YAML configuration (dev, test, prod) including MongoDB vs JPA variants.

```yaml
# application-prod.yml - Production overrides (no defaults for sensitive values)

spring:
  application:
    name: user-service
  data:
    mongodb:
      uri: ${MONGODB_URI}           # Required — no default in production
      auto-index-creation: false
  jackson:
    deserialization:
      fail-on-null-for-primitives: false

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true
  tracing:
    sampling:
      probability: 0.1   # 10% in prod
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

server:
  port: 8080
  compression:
    enabled: true
    min-response-size: 1024
```

## Deployment Checklist

- [ ] Dockerfile uses multi-stage build (`maven:3.9.9-eclipse-temurin-26-alpine` + `eclipse-temurin:26-jre-alpine`)
- [ ] Non-root user created with `addgroup -S` / `adduser -S` (Alpine syntax)
- [ ] `apk add` used for Alpine packages — never `apt-get`
- [ ] HEALTHCHECK uses `wget` or `curl` (after `apk add --no-cache curl`) against `/actuator/health/liveness`
- [ ] `SPRING_PROFILES_ACTIVE` (not `SPRING_PROFILE`) set in Dockerfile and Compose
- [ ] JVM memory settings optimized for container
- [ ] Health check endpoints: `/actuator/health/liveness` and `/actuator/health/readiness`
- [ ] Docker image size < 200MB
- [ ] Docker Compose service has `healthcheck` with `start_period: 40s`
- [ ] `MONGODB_URI` env var configured without default in prod (not `${URI:default-value}`)
- [ ] `GATEWAY_TOKEN` injected from environment (never hardcoded)
- [ ] CI/CD pipeline configured per `24-CICD-SECRETS.md`
- [ ] Graceful shutdown: `server.shutdown: graceful` in prod YAML

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
