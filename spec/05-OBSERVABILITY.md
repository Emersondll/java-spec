# Observability - Logging, Metrics, Tracing

## Logging Standards (SLF4J + Logback)

### Configuration

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-logs/}spring.log}"/>
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>
    
    <!-- Development: Console with colors -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    
    <!-- Production: JSON file with rotation -->
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}</file>
        
        <!-- JSON format for ELK/Datadog ingestion -->
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeContext>true</includeContext>
            <includeMdcData>true</includeMdcData>
            <customFields>{"service":"user-service","environment":"${SPRING_PROFILE}"}
            </customFields>
        </encoder>
        
        <!-- Rotate logs daily or when 100MB reached -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
    </appender>
    
    <!-- Development profile -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="org.springframework" level="INFO"/>
        <logger name="org.springframework.data" level="DEBUG"/>
        <logger name="org.mongodb.driver" level="DEBUG"/>
    </springProfile>
    
    <!-- Production profile -->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="JSON_FILE"/>
        </root>
        <logger name="com.company" level="INFO"/>
        <logger name="org.springframework" level="WARN"/>
        <logger name="org.springframework.security" level="INFO"/>
    </springProfile>
</configuration>
```

### Logging in Services

```java
/**
 * Service with structured logging following best practices.
 * 
 * Logging Strategy:
 * - INFO: Business events (user created, payment processed)
 * - WARN: Recoverable errors (retry, fallback, degradation)
 * - ERROR: Unrecoverable errors (persistence failure, validation)
 * - DEBUG: Diagnostic information (variable values, method flow)
 * 
 * Never log:
 * - Passwords, secrets, PII without redaction
 * - System.out.println (use logger instead)
 * - Exceptions at INFO level (use WARN/ERROR)
 * 
 * Always include:
 * - Business context (userId, requestId, operation)
 * - Performance metrics (duration, count)
 * - Error root cause (exception class, message)
 */
@Service
@Slf4j
public class UserService {
    
    /**
     * Create user with comprehensive logging.
     * 
     * @param request user creation request
     * @return created user response
     */
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        // INFO: Log business operation start
        log.info("Creating user. email={}", request.email());
        
        long startTime = System.currentTimeMillis();
        
        try {
            // Validation
            if (userRepository.existsByEmail(request.email())) {
                log.warn("User creation failed: email already exists. email={}", request.email());
                throw new EmailAlreadyExistsException(request.email());
            }
            
            // Persistence
            User user = User.from(request);
            User saved = userRepository.save(user);
            
            long duration = System.currentTimeMillis() - startTime;
            
            // INFO: Log successful completion with metrics
            log.info("User created successfully. userId={}, email={}, duration={}ms", 
                saved.getId(), saved.getEmail(), duration);
            
            return UserResponse.fromDomain(saved);
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            
            // ERROR: Log failure with full context and exception
            log.error("User creation failed. email={}, duration={}ms", 
                request.email(), duration, e);
            
            // Re-throw or wrap exception
            throw new UserCreationException("Cannot create user: " + e.getMessage(), e);
        }
    }
    
    /**
     * Find user with DEBUG logging for diagnostics.
     * 
     * @param userId user identifier
     * @return user response
     */
    public UserResponse findUserById(String userId) {
        log.debug("Finding user. userId={}", userId);
        
        return userRepository.findById(userId)
            .map(user -> {
                log.debug("User found. userId={}, email={}, status={}", 
                    user.getId(), user.getEmail(), user.getStatus());
                return UserResponse.fromDomain(user);
            })
            .orElseThrow(() -> {
                log.warn("User not found. userId={}", userId);
                return new UserNotFoundException(userId);
            });
    }
}
```

### Request Context Logging (MDC)

> See also: **15-ERROR-RESPONSE.md** — `traceId` MDC key is included in every error response envelope
> See also: **14-SECURITY.md** — GatewayAuthFilter and JWT subject extraction

```java
/**
 * Filter adding request context to all logs.
 * MDC (Mapped Diagnostic Context) threads request-scoped attributes.
 * All logs from this request will include these attributes.
 *
 * MDC Variables:
 * - traceId: Distributed trace ID (also used by ErrorResponse envelope)
 * - requestId: Unique identifier for this request (uuid, kept for log backward-compat)
 * - userId: User making request (from JWT or session)
 * - method: HTTP method (GET, POST, etc)
 * - path: Request path
 * - duration: Total request duration
 */
@Component
@Slf4j
public class RequestContextFilter extends OncePerRequestFilter {
    
    private static final String REQUEST_ID = "requestId";
    private static final String USER_ID = "userId";
    private static final String METHOD = "method";
    private static final String PATH = "path";
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String requestId = UUID.randomUUID().toString();
        String userId = extractUserId(request);
        
        // Add to MDC (thread-local context)
        // traceId is the key used by ErrorResponse.of() — must be set here
        MDC.put("traceId", requestId);
        MDC.put(REQUEST_ID, requestId);  // kept for backward-compat log queries
        MDC.put(USER_ID, userId);
        MDC.put(METHOD, request.getMethod());
        MDC.put(PATH, request.getRequestURI());
        
        long startTime = System.currentTimeMillis();
        
        try {
            log.debug("Incoming request. requestId={}, method={}, path={}", 
                requestId, request.getMethod(), request.getRequestURI());
            
            filterChain.doFilter(request, response);
            
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request completed. status={}, duration={}ms", 
                response.getStatus(), duration);
            
        } finally {
            MDC.clear();  // Clean up to prevent memory leaks
        }
    }
    
    /**
     * Extract user ID from JWT or session.
     * Returns "anonymous" if not authenticated.
     * 
     * @param request HTTP request
     * @return user identifier or "anonymous"
     */
    private String extractUserId(HttpServletRequest request) {
        // Extract subject from JWT — see 14-SECURITY.md for GatewayAuthFilter JWT extraction
        return "anonymous";  // replaced at runtime by GatewayAuthFilter
    }
}
```

## Metrics with Micrometer

### Setup

> See also: **20-MAVEN-POM.md** — complete managed dependency list for observability

```xml
<!-- pom.xml — versions managed by Spring Boot BOM -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
    metrics:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s,2s
```

### Custom Metrics

```java
/**
 * Service exposing domain-specific metrics.
 * Metrics enable real-time monitoring of business operations.
 * 
 * Metrics exported:
 * - users.created (counter): Total users created since startup
 * - users.active (gauge): Currently active users
 * - user.creation.duration (timer): Time to create user with percentiles
 * - user.creation.failures (counter): Failed creation attempts with reason
 */
@Service
@Slf4j
public class UserService {
    
    private final MeterRegistry meterRegistry;
    private final AtomicInteger activeUsers;
    
    /**
     * Constructor initializing metrics.
     * 
     * @param meterRegistry Micrometer registry for metric registration
     */
    public UserService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Gauge: Current active user count
        this.activeUsers = meterRegistry.gauge(
            "users.active",
            new AtomicInteger(0)
        );
    }
    
    /**
     * Create user with performance metrics.
     * 
     * Metrics:
     * - user.creation.duration: Timer with p50, p95, p99 percentiles
     * - user.creation.success: Counter incremented on success
     * - user.creation.failure: Counter with reason tag
     * 
     * @param request user creation request
     * @return created user response
     */
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        // Timer.Sample captures start time for duration measurement
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // Business logic
            User user = User.from(request);
            User saved = userRepository.save(user);
            
            // Success counter
            meterRegistry.counter("user.creation.success").increment();
            
            activeUsers.incrementAndGet();
            
            return UserResponse.fromDomain(saved);
            
        } catch (EmailAlreadyExistsException e) {
            // Failure counter with reason tag
            meterRegistry.counter("user.creation.failure", "reason", "email_exists").increment();
            throw e;
            
        } finally {
            // Record duration timer with percentiles
            sample.stop(Timer.builder("user.creation.duration")
                .description("Time to create user")
                .publishPercentiles(0.5, 0.95, 0.99)
                .publishPercentileHistogram(true)
                .minimumExpectedValue(Duration.ofMillis(50))
                .maximumExpectedValue(Duration.ofSeconds(5))
                .slo(Duration.ofMillis(100), Duration.ofMillis(500))
                .register(meterRegistry));
        }
    }
    
    /**
     * Login user tracking active sessions.
     * Increments gauge of active users.
     * 
     * @param userId user identifier
     */
    public void loginUser(String userId) {
        log.info("User login. userId={}", userId);
        activeUsers.incrementAndGet();
        meterRegistry.counter("user.login").increment();
    }
    
    /**
     * Logout user decrementing active count.
     * 
     * @param userId user identifier
     */
    public void logoutUser(String userId) {
        log.info("User logout. userId={}", userId);
        activeUsers.decrementAndGet();
        meterRegistry.counter("user.logout").increment();
    }
}
```

### Prometheus Queries

```promql
# Rate of user creations per second (5-minute window)
rate(user_creation_success_total[5m])

# Latency: 95th percentile of user creation (50ms SLO)
histogram_quantile(0.95, user_creation_duration_seconds_bucket)

# Error rate: Failed user creations
rate(user_creation_failure_total[5m])

# Active users gauge
users_active

# HTTP requests slower than 500ms (SLO violation)
rate(http_server_requests_seconds_bucket{le="0.5"}[5m]) / rate(http_server_requests_seconds_count[5m])
```

## Distributed Tracing

### Micrometer Tracing + OTLP → Tempo

Spring Cloud Sleuth and Zipkin are deprecated. Use **Micrometer Tracing** with the OpenTelemetry bridge and OTLP exporter → Tempo.

```xml
<!-- pom.xml — managed by Spring Boot BOM -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.exporter</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0   # dev: 100% | prod: 0.1
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

### Manual Span Creation

```java
/**
 * Service with manual span instrumentation using Micrometer Tracing.
 * Spans track distributed operations across services.
 * 
 * Automatic tracing:
 * - HTTP requests/responses (Spring Boot)
 * - Database queries (Spring Data MongoDB)
 * - REST calls (OpenFeign)
 * 
 * Manual spans for custom operations.
 */
@Service
@Slf4j
public class UserService {
    
    private final Tracer tracer;
    
    /**
     * Create user with custom span for email verification.
     * 
     * Span hierarchy:
     * - Root span: POST /api/v1/users (created by Spring)
     *   - Child span: email-verification
     *   - Child span: user-persistence
     * 
     * @param request user creation request
     * @return created user response
     */
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        log.info("Creating user. email={}", request.email());
        
        // Manual span for email verification (Micrometer Tracing API)
        Span emailSpan = tracer.nextSpan().name("email-verification").start();
        try (Tracer.SpanInScope scope = tracer.withSpan(emailSpan)) {
            emailSpan.tag("email", request.email());
            validateEmail(request.email());
        } catch (Exception e) {
            emailSpan.error(e);
            throw e;
        } finally {
            emailSpan.end();
        }
        
        User user = User.from(request);
        User saved = userRepository.save(user);
        
        log.info("User created. userId={}", saved.getId());
        return UserResponse.fromDomain(saved);
    }
}
```

## Health Checks

Spring Boot Actuator auto-configures a MongoDB health indicator when `spring-boot-starter-data-mongodb` is on the classpath. No custom code needed for basic DB health.

```yaml
# application.yml — expose health endpoints for Docker HEALTHCHECK
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true   # enables /actuator/health/liveness and /readiness
```

```java
/**
 * Custom health indicator for external dependency (e.g., RabbitMQ broker).
 * MongoDB health is auto-configured by Spring Boot Actuator.
 * Exposed at /actuator/health endpoint.
 */
@Component
public class BrokerHealthIndicator implements HealthIndicator {

    private final RabbitTemplate rabbitTemplate;

    /**
     * Constructor injection.
     *
     * @param rabbitTemplate Spring AMQP template
     */
    public BrokerHealthIndicator(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = Objects.requireNonNull(rabbitTemplate);
    }

    /**
     * Check RabbitMQ broker health.
     *
     * @return UP if broker is reachable, DOWN otherwise
     */
    @Override
    public Health health() {
        try {
            rabbitTemplate.getConnectionFactory().createConnection().close();
            return Health.up()
                .withDetail("broker", "RabbitMQ")
                .withDetail("status", "reachable")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("broker", "RabbitMQ")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

Docker Compose health probe:

```yaml
# In docker-compose.yml service definition
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health/liveness"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 40s
```

## Observability Checklist

- [ ] Structured logging enabled (JSON in production)
- [ ] All INFO-level logs include business context
- [ ] Request ID (MDC) propagated through entire flow
- [ ] Metrics collected for critical operations
- [ ] Custom gauges for business metrics
- [ ] Distributed tracing enabled (trace ID in logs)
- [ ] Health checks implemented (/actuator/health)
- [ ] Prometheus scrape endpoint configured
- [ ] Tempo configured for trace visualization (via OTLP)
- [ ] Logs ship to Loki (via OTel Collector)
- [ ] Alerts configured for critical metrics
- [ ] SLO/SLA dashboards created in Grafana
