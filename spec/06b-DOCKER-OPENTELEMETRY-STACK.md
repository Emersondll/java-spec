# Docker & OpenTelemetry Stack - Complete Observability

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> See also: **05-OBSERVABILITY.md** — application-level instrumentation (MDC, metrics, logging)
> See also: **20-MAVEN-POM.md** — complete managed dependency list (never pin OTel versions manually)
> See also: **06-DOCKER-DEPLOYMENT.md** — base Docker and Docker Compose setup

## Complete Observability Stack Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Java Spring Boot App                        │
│  (Instrumented with OpenTelemetry SDK)                        │
└────────────────┬───────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┬─────────────┐
    ↓            ↓            ↓             ↓
┌────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Logs   │ │ Metrics  │ │ Traces   │ │ Events   │
└───┬────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
    │           │            │            │
    └───────────┼────────────┼────────────┘
                ↓
    ┌───────────────────────────────┐
    │ OpenTelemetry Collector       │
    │ (OTLP Protocol Receiver)      │
    └───┬───────────────┬───────────┘
        │               │
        ↓               ↓
    ┌────────┐    ┌─────────┐
    │  Loki  │    │ Tempo   │
    │ (Logs) │    │(Traces) │
    └────┬───┘    └────┬────┘
         │             │
    ┌────┴─────┬───────┴────┐
    ↓          ↓            ↓
┌──────────────────────────────────┐
│       Prometheus Scrape           │
│       (Metrics Collection)        │
└────┬─────────────────────────────┘
     │
     ↓
┌──────────────────────────────────┐
│   Grafana Dashboard              │
│   (Visualization & Alerting)     │
└──────────────────────────────────┘
```

---

## Docker Compose Configuration

### Complete docker-compose.yml

```yaml
# docker-compose.yml - Complete Observability Stack
version: '3.9'

services:
  # ==================== APPLICATION ====================
  
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: java-app
    depends_on:
      otel-collector:
        condition: service_healthy
      prometheus:
        condition: service_healthy
      grafana:
        condition: service_healthy
    environment:
      # OpenTelemetry Configuration
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_EXPORTER_OTLP_PROTOCOL: grpc
      OTEL_SERVICE_NAME: java-application
      OTEL_TRACES_EXPORTER: otlp
      OTEL_METRICS_EXPORTER: otlp
      OTEL_LOGS_EXPORTER: otlp
      
      # Spring Boot Configuration
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATA_MONGODB_URI: mongodb://mongo:27017/app
      
      # JVM Configuration
      JVM_OPTS: "-Xms512m -Xmx1024m -XX:+UseG1GC"
      
      # Logging
      LOGGING_LEVEL_ROOT: INFO
      LOGGING_LEVEL_COM_COMPANY: DEBUG
    ports:
      - "8080:8080"
      - "8081:8081"
    networks:
      - observability
    volumes:
      - ./logs:/var/log/app
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ==================== DATABASES ====================
  
  mongo:
    image: mongo:7.0
    container_name: mongo-db
    environment:
      MONGO_INITDB_DATABASE: app
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    networks:
      - observability
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==================== OBSERVABILITY STACK ====================
  
  # OpenTelemetry Collector - Central collection point
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    environment:
      GOGC: 80
    ports:
      # OTLP Receiver
      - "4317:4317"  # OTLP gRPC receiver
      - "4318:4318"  # OTLP HTTP receiver
      # Metrics
      - "8889:8889"  # Prometheus exporter
      - "8888:8888"  # Metrics endpoint
      # Traces
      - "13133:13133" # health check
      - "55679:55679" # zpages
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    command: 
      - "--config=/etc/otel-collector-config.yaml"
    depends_on:
      - prometheus
      - loki
      - tempo
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:13133"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Prometheus - Metrics time-series database
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    environment:
      TZ: UTC
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=7d'
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9090"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Loki - Log aggregation
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command:
      - -config.file=/etc/loki/local-config.yaml
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3100/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Tempo - Distributed tracing
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"  # HTTP API — Grafana accesses Tempo here
      # Port 4317 NOT exposed on host — otel-collector routes to tempo:4317 within Docker network
      # (otel-collector already binds 4317:4317; two containers cannot share the same host port)
    volumes:
      - ./tempo-config.yml:/etc/tempo-config.yaml
      - tempo_data:/var/tempo
    command:
      - -config.file=/etc/tempo-config.yaml
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3200"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Grafana - Visualization & Alerting
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: false
      GF_AUTH_BASIC_ENABLED: true
      GF_PATHS_PROVISIONING: /etc/grafana/provisioning
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      - prometheus
      - loki
      - tempo
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  observability:
    driver: bridge

volumes:
  mongo_data:
  prometheus_data:
  loki_data:
  tempo_data:
  grafana_data:
```

---

## Configuration Files

### otel-collector-config.yaml

```yaml
# OpenTelemetry Collector Configuration
receivers:
  # OTLP Protocol - Native OpenTelemetry Protocol
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Prometheus Scrape Configuration
  prometheus:
    config:
      scrape_configs:
        - job_name: 'java-app'
          static_configs:
            - targets: ['localhost:8080']
          metrics_path: '/actuator/prometheus'
          scrape_interval: 15s
          scrape_timeout: 10s

processors:
  # Batch Processing - Improve throughput
  batch:
    send_batch_size: 1000
    timeout: 10s
    send_batch_max_size: 2000

  # Attributes Processing
  attributes:
    actions:
      - key: service.version
        value: "1.0.0"
        action: insert
      - key: environment
        value: "development"
        action: insert

  # Sampling - Reduce volume
  tail_sampling:
    policies:
      # Always sample errors
      - name: error-traces
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      # Sample 10% of successful traces
      - name: probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  # Prometheus - Metrics Export
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"

  # Loki - Logs Export
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    labels:
      service: java-app
      environment: development

  # Tempo - Traces Export
  otlp:
    client:
      endpoint: tempo:4317
      tls:
        insecure: true

service:
  pipelines:
    # Traces Pipeline: Receive → Process → Export
    traces:
      receivers: [otlp]
      processors: [batch, attributes, tail_sampling]
      exporters: [otlp]
    
    # Metrics Pipeline
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, attributes]
      exporters: [prometheus]
    
    # Logs Pipeline
    logs:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [loki]

  telemetry:
    logs:
      level: info
```

### prometheus.yml

```yaml
# Prometheus Configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'local'
    environment: 'development'

scrape_configs:
  # Java Application Metrics
  - job_name: 'java-app'
    static_configs:
      - targets: ['app:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    scrape_timeout: 10s

  # OpenTelemetry Collector Metrics
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']

  # Prometheus Self
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

alerting:
  alertmanagers:
    - static_configs:
        - targets: []
```

### loki-config.yml

```yaml
# Loki Configuration for Log Aggregation
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  chunk_retain_period: 1m
  max_chunk_age: 1h
  chunk_encoding: snappy
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema:
        version: v11
        index:
          prefix: index_
          period: 24h

server:
  http_listen_port: 3100
  log_level: info

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks
```

### tempo-config.yml

```yaml
# Tempo Configuration for Distributed Tracing
auth_enabled: false

distributor:
  rate_limit_bytes: 10000000
  max_request_duration: 5m

receiver:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

ingester:
  lifecycler:
    ring:
      replication_factor: 1

storage:
  trace:
    backend: local
    wal:
      path: /var/tempo/wal
    local:
      path: /var/tempo/chunks

querier:
  max_search_bytes: 50000000
  search:
    max_pages: 2000

server:
  http_listen_port: 3200
  log_level: info
```

---

## Java Application Instrumentation

### Maven Dependencies

> **Dependencies:** All OTel and Micrometer dependencies are managed by the Spring Boot 4 BOM.
> See **20-MAVEN-POM.md** for the complete, version-managed dependency list.
> **Never** pin versions for Spring-managed dependencies — let the BOM resolve them.
>
> Key dependencies (no `<version>` tags needed):
> - `micrometer-tracing-bridge-otel` — Micrometer ↔ OTel bridge (auto-configures OTel SDK)
> - `opentelemetry-exporter-otlp` — OTLP gRPC/HTTP exporter
> - `micrometer-registry-prometheus` — Prometheus metrics endpoint
>
> **Do NOT add `opentelemetry-sdk` directly** — the bridge pulls it transitively and adding it
> manually can cause version conflicts with Spring Boot's auto-configuration.

### application.yml Configuration

```yaml
# Spring Boot Application Configuration with OpenTelemetry
spring:
  application:
    name: java-application
  
  profiles:
    active: dev
  
  data:
    mongodb:
      uri: mongodb://mongo:27017/app

management:
  # Actuator Endpoints — mandated set per CLAUDE.md
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus

  # Metrics Export (Prometheus)
  metrics:
    export:
      prometheus:
        enabled: true

  # OpenTelemetry Tracing — Spring Boot 4 / Micrometer correct path
  otlp:
    tracing:
      endpoint: http://otel-collector:4317/v1/traces
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (dev), use 0.1 for prod

logging:
  level:
    root: INFO
    com.company: DEBUG
  
  # JSON Logging for Loki
  pattern:
    console: '{"time":"%d{yyyy-MM-dd HH:mm:ss}","level":"%p","thread":"%t","logger":"%c","message":"%m"}%n'
```

### Java Code with Instrumentation

```java
/**
 * Service with OpenTelemetry instrumentation.
 * 
 * Instrumentation automatically captures:
 * - Method execution time
 * - Exceptions
 * - HTTP requests/responses
 * - Database queries
 * - Distributed context (trace ID)
 */
@Service
@Transactional
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final Tracer tracer;  // OpenTelemetry Tracer
    private final MeterRegistry meterRegistry;
    
    /**
     * Constructor injection.
     * 
     * @param userRepository user repository
     * @param tracer OpenTelemetry tracer
     * @param meterRegistry metrics registry
     */
    public UserService(
        UserRepository userRepository,
        Tracer tracer,
        MeterRegistry meterRegistry
    ) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.tracer = Objects.requireNonNull(tracer);
        this.meterRegistry = Objects.requireNonNull(meterRegistry);
    }
    
    /**
     * Create user with automatic instrumentation.
     * 
     * Automatically instrumented by Micrometer Tracing + OpenTelemetry:
     * - Trace ID: Propagated across services
     * - Span: Method execution tracked
     * - Metrics: Timing and counters
     * - Logs: Context data included
     * 
     * @param request user creation request
     * @return created user response
     */
    public UserResponse createUser(CreateUserRequest request) {
        // Trace ID automatically in logs and metrics
        log.info("Creating user. email={}", request.email());
        
        // Create custom span for specific operation
        Span span = tracer.spanBuilder("email-validation")
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            validateEmail(request.email());
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR);
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
        
        User user = createAndPersistUser(request);
        
        log.info("User created. userId={}", user.getId());
        meterRegistry.counter("user.created").increment();
        
        return UserResponse.fromDomain(user);
    }
}
```

---

## Grafana Datasources Configuration

### grafana/provisioning/datasources/datasources.yml

```yaml
apiVersion: 1

datasources:
  # Prometheus - Metrics
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    jsonData:
      timeInterval: 15s

  # Loki - Logs
  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy
    jsonData:
      derivedFields:
        - name: trace_id
          matcherRegex: 'trace_id=(\w+)'
          url: 'http://tempo:3200/api/traces/$${__value.raw}'
          datasourceName: Tempo

  # Tempo - Traces
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    access: proxy
    jsonData:
      tracesToMetrics:
        datasourceUid: prometheus_uid
      tracesToLogs:
        datasourceUid: loki_uid
      serviceMap:
        datasourceUid: prometheus_uid
```

---

## Quick Start Commands

### Start Full Stack

```bash
# Build and start all services
docker-compose up -d

# Follow logs
docker-compose logs -f app

# Stop all services
docker-compose down

# Clean everything (including data)
docker-compose down -v
```

### Access Services

```
Application:      http://localhost:8080
Grafana:          http://localhost:3000 (admin/admin)
Prometheus:       http://localhost:9090
Loki:             http://localhost:3100
Tempo:            http://localhost:3200
```

---

## Grafana Dashboard Queries

### Metrics (Prometheus)

```promql
# CPU Usage
rate(jvm_memory_usage_bytes{type="heap"}[5m])

# Request Rate
rate(http_server_requests_seconds_count[5m])

# Error Rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m])

# Response Time P95
histogram_quantile(0.95, http_server_requests_seconds_bucket)
```

### Logs (Loki)

```
# Application logs
{service="java-app"} | json

# Errors only
{service="java-app"} | json | level="ERROR"

# Specific user
{service="java-app"} | json | user_id="123"
```

### Traces (Tempo)

```
# All traces for java-app
{service.name="java-application"}

# Traces with errors
{service.name="java-application", status="error"}

# Slow traces (> 1s)
{service.name="java-application", duration>1s}
```

---

## Production Checklist

- [ ] OpenTelemetry SDK configured
- [ ] OTLP endpoint set to OTel Collector
- [ ] Sampling configured appropriately (10-50% for prod)
- [ ] Custom metrics implemented
- [ ] Traces exported to Tempo
- [ ] Logs aggregated in Loki
- [ ] Metrics scraped by Prometheus
- [ ] Grafana dashboards created
- [ ] Alerts configured in Prometheus
- [ ] Retention policies set (7-30 days)
- [ ] Performance impact monitored
- [ ] Security (TLS for OTLP, auth for Grafana)

