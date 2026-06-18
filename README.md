# Java Spring Boot Microservices Specification

> Enterprise-grade refactoring and standardization guidelines for Java 26 and Spring Boot 4.0.7 microservices.

![Java](https://img.shields.io/badge/Java-26-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0.7-green)
![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2025.1.0-blue)
![Maven](https://img.shields.io/badge/Maven-3.9.9-red)
![License](https://img.shields.io/badge/License-MIT-brightgreen)

## Overview

This repository contains a complete **27-specification suite** for building, refactoring, and maintaining enterprise-grade Java 26 microservices using Spring Boot 4.0.7. It serves as the authoritative standard for:

- **Architecture & Design**: Layered architecture, hexagonal patterns, dependency injection
- **Code Quality**: JavaDoc 100% mandatory, Java 26 Records, sealed classes, clean code principles
- **Testing & Coverage**: JaCoCo minimum 80%, test pyramid strategy
- **Security**: Trust boundaries, GatewayAuthFilter, IDOR prevention, secrets management
- **Observability**: OpenTelemetry, Micrometer, Prometheus, Grafana, structured logging
- **DevOps**: Multi-stage Docker builds, CI/CD pipelines, Kubernetes, ArgoCD GitOps
- **Database**: MongoDB or PostgreSQL/MySQL (per-service pattern) with proper indexing
- **Messaging**: RabbitMQ, SAGA event choreography, DLQ handling
- **API Documentation**: Static OpenAPI 3.0 YAML (zero annotations in code)

## Quick Navigation

### 📋 Core Architecture & Design
- [**01-ARCHITECTURE.md**](spec/01-ARCHITECTURE.md) — Spring Boot layered architecture, project structure, dependency injection, design patterns
- [**01b-HEXAGONAL-ARCHITECTURE.md**](spec/01b-HEXAGONAL-ARCHITECTURE.md) — Hexagonal architecture conventions, ports, adapters
- [**02-CODE-QUALITY.md**](spec/02-CODE-QUALITY.md) — Code standards, null-safety, static analysis (SonarQube, SpotBugs, Checkstyle)
- [**07-JAVA26-RECORDS-JAVADOC.md**](spec/07-JAVA26-RECORDS-JAVADOC.md) — Java 26 features, Records, sealed classes, complete JavaDoc templates

### 🧪 Testing & Validation
- [**03-TESTING-STRATEGY.md**](spec/03-TESTING-STRATEGY.md) — Test pyramid, JaCoCo gates (80% minimum), unit/integration/E2E tests, test containers

### ⚡ Performance & Resilience
- [**04-PERFORMANCE.md**](spec/04-PERFORMANCE.md) — N+1 query elimination, caching strategies, connection pools, JVM tuning, performance baselines
- [**10-CIRCUIT-BREAKER.md**](spec/10-CIRCUIT-BREAKER.md) — Resilience4j, circuit breaker pattern, fallback strategies, error handling

### 📊 Observability & Monitoring
- [**05-OBSERVABILITY.md**](spec/05-OBSERVABILITY.md) — Structured logging, SLF4J, Micrometer, Prometheus, OpenTelemetry, OTLP
- [**06b-DOCKER-OPENTELEMETRY-STACK.md**](spec/06b-DOCKER-OPENTELEMETRY-STACK.md) — Complete observability stack (Prometheus, Loki, Tempo, Grafana)

### 🐳 Deployment & DevOps
- [**06-DOCKER-DEPLOYMENT.md**](spec/06-DOCKER-DEPLOYMENT.md) — Multi-stage Dockerfile, health checks, Docker Compose
- [**24-CICD-SECRETS.md**](spec/24-CICD-SECRETS.md) — GitHub Actions, secrets management, Vault, Kubernetes, ArgoCD GitOps, Trivy scanning

### 🔧 Code Patterns & Best Practices
- [**09-SYNC-ASYNC-COMMUNICATION.md**](spec/09-SYNC-ASYNC-COMMUNICATION.md) — Synchronous vs asynchronous patterns, @Async, CompletableFuture
- [**11-IMPORT-GUIDELINES.md**](spec/11-IMPORT-GUIDELINES.md) — Import standards (NO wildcards), organization
- [**12-MANDATORY-ANNOTATIONS.md**](spec/12-MANDATORY-ANNOTATIONS.md) — Spring stereotype annotations, @Repository, @Service, @RestController

### 🤖 AI Refactoring Automation
- [**08-REFACTORING-CHECKLIST.md**](spec/08-REFACTORING-CHECKLIST.md) — Step-by-step AI refactoring guide, before/after examples, validation checklists

### 🌐 API Reference
- [**13-OPENAPI-REST-REFERENCE.md**](spec/13-OPENAPI-REST-REFERENCE.md) — Static OpenAPI 3.0 YAML standard, zero annotations in code

### 🔐 Security & Contracts
- [**14-SECURITY.md**](spec/14-SECURITY.md) — Trust boundaries, GatewayAuthFilter, IDOR prevention, secrets management
- [**15-ERROR-RESPONSE.md**](spec/15-ERROR-RESPONSE.md) — Standard HTTP error envelope, GlobalExceptionHandler, error codes
- [**21-HTTP-IDEMPOTENCY.md**](spec/21-HTTP-IDEMPOTENCY.md) — Idempotency-Key header pattern, distributed idempotency

### 📨 Messaging & Events
- [**16-RABBITMQ-SAGA.md**](spec/16-RABBITMQ-SAGA.md) — RabbitMQ architecture, SAGA event choreography, DLQ handling, idempotency

### 📄 API Contracts & Patterns
- [**17-PAGINATION.md**](spec/17-PAGINATION.md) — Pagination standard, PageResponse<T>, size limits, sorting conventions

### 🗄️ Data & Configuration
- [**18-MONGODB-INDEXES.md**](spec/18-MONGODB-INDEXES.md) — MongoDB indexing conventions, TTL, Mongock migrations
- [**18b-RELATIONAL-INDEXES-MIGRATIONS.md**](spec/18b-RELATIONAL-INDEXES-MIGRATIONS.md) — JPA & Flyway migrations, index naming
- [**19-SPRING-PROFILES.md**](spec/19-SPRING-PROFILES.md) — Spring profiles (dev, test, prod), application-{profile}.yml patterns
- [**20-MAVEN-POM.md**](spec/20-MAVEN-POM.md) — Standard Maven POM structure, plugins, BOM, dependency management

### 📝 Documentation Standards
- [**22-DOCUMENTATION.md**](spec/22-DOCUMENTATION.md) — Project documentation structure, README standards, supplementary docs, AI generation guidelines

### 🔀 Version Control & CI/CD
- [**23-GIT-BRANCHING.md**](spec/23-GIT-BRANCHING.md) — GitHub branching strategy, PR workflow, Conventional Commits, SemVer tagging
- [**00-INDEX.md**](spec/00-INDEX.md) — Complete index and getting started guide

## Required Technology Stack

| Technology | Version | Purpose |
|---|---|---|
| Java | 26 (mandatory) | Runtime environment |
| Spring Boot | 4.0.7 | Framework foundation |
| Spring Cloud | 2025.1.0 | Cloud patterns & service discovery |
| Maven | 3.9.9 | Build & dependency management |
| **Database** | MongoDB 7.0 **OR** PostgreSQL 16/MySQL 8.0 | Persistent storage (per-service) |
| RabbitMQ | 3.x | Asynchronous messaging & SAGA events |
| Docker | Latest | Containerization & orchestration |
| Kubernetes | 1.28+ | Container orchestration (optional) |

## Key Requirements Summary

### ✅ Mandatory Standards

**Java 26 Features:**
- All DTOs/Requests/Responses MUST be `record` types (not POJOs)
- Sealed classes for restricted polymorphism
- Pattern matching for type-safe code
- Source/Target: Java 26 in build configuration

**Code Quality (100% Required):**
- Every public class: complete class-level JavaDoc
- Every public method: complete method-level JavaDoc
- Every record field: mandatory field documentation
- Methods: < 30 lines maximum
- Classes: < 300 lines maximum
- No null pointer risks: use Optional or validation

**Spring Boot Architecture:**
- Constructor-based dependency injection ONLY (no field `@Autowired`)
- Layered architecture: Controller → Service → Repository → Database
- Centralized exception handling: `@RestControllerAdvice`
- Configuration: `application.yml` ONLY (never `application.properties`)

**Testing & Coverage:**
- Minimum 80% code coverage (JaCoCo gate)
- Services: 90% coverage target
- Controllers: 70% coverage target
- JUnit 5 + Mockito framework

**API Documentation:**
- Static `openapi.yaml` at `src/main/resources/static/openapi.yaml`
- **FORBIDDEN annotations** in code: `@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition`
- Swagger UI serves the static file

**REST Status Codes:**
- `201 CREATED` for POST with Location header
- `200 OK` for GET
- `204 NO CONTENT` for DELETE/successful operations
- `400 BAD REQUEST` for validation errors
- `404 NOT FOUND` for missing resources
- `409 CONFLICT` for uniqueness violations
- `500 INTERNAL SERVER ERROR` for unexpected failures

## Documentation Structure for Microservices

Every microservice repository built using these specifications MUST include:

```
microservice-root/
├── README.md                                    ← Main entry point
├── src/main/resources/
│   ├── static/
│   │   └── openapi.yaml                         ← API contract (static, no annotations)
│   ├── application.yml                          ← Base configuration
│   ├── application-dev.yml                      ← Dev profile
│   ├── application-test.yml                     ← Test profile
│   ├── application-prod.yml                     ← Prod profile
│   └── docs/
│       ├── README.md                            ← Supplementary docs index
│       ├── architecture.md                      ← Internal architecture details
│       ├── context.md                           ← System context and boundaries
│       ├── configuration.md                     ← Env vars, profiles, feature flags
│       ├── deployment.md                        ← Environments, pipeline, rollback
│       ├── observability.md                     ← Logs, metrics, tracing, dashboards
│       ├── troubleshooting.md                   ← Common issues & resolution
│       ├── events.md                            ← Published/consumed async events
│       └── runbooks/
│           ├── README.md                        ← Runbook index
│           └── incident-template.md             ← Incident response template
├── Dockerfile                                   ← Multi-stage build
├── docker-compose.yml                           ← Local dev environment
├── pom.xml                                      ← Maven configuration
└── spec/                                        ← Link to this repository or reference
```

## Using This Specification

### For New Microservices

1. **Read [spec/00-INDEX.md](spec/00-INDEX.md)** — Overview of all specifications
2. **Read [spec/01-ARCHITECTURE.md](spec/01-ARCHITECTURE.md)** — Understand layered architecture
3. **Read [spec/02-CODE-QUALITY.md](spec/02-CODE-QUALITY.md)** — Learn code quality standards
4. **Read [spec/07-JAVA26-RECORDS-JAVADOC.md](spec/07-JAVA26-RECORDS-JAVADOC.md)** — Java 26 & JavaDoc rules
5. **Read [spec/22-DOCUMENTATION.md](spec/22-DOCUMENTATION.md)** — Create project documentation
6. **Read [spec/06-DOCKER-DEPLOYMENT.md](spec/06-DOCKER-DEPLOYMENT.md)** — Docker & deployment

**Time Investment:** ~4 hours reading + hands-on setup

### For AI-Assisted Refactoring

1. **Load this specification** into your AI tool (Claude, ChatGPT, etc.)
2. **Follow [spec/08-REFACTORING-CHECKLIST.md](spec/08-REFACTORING-CHECKLIST.md)** — Step-by-step process
3. **Reference [spec/02-CODE-QUALITY.md](spec/02-CODE-QUALITY.md)** — Before refactoring code
4. **Run `mvn verify`** — Must pass before any commit
5. **Use [spec/22-DOCUMENTATION.md](spec/22-DOCUMENTATION.md)** — For documentation generation

### For Code Review

**Pre-commit validation:**
```bash
# Must pass before any commit
mvn clean verify

# Check specific rules
mvn jacoco:report           # Coverage report (target/site/jacoco/index.html)
mvn sonar:sonar             # SonarQube analysis
mvn checkstyle:check        # Code style violations
mvn com.github.spotbugs:spotbugs-maven-plugin:spotbugs  # Bug detection
```

## Refactoring Examples

### Convert POJO to Java 26 Record

**Before (Anti-pattern: POJO with Lombok):**
```java
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class UserDTO {
    private String id;
    private String email;
}
```

**After (Correct: Java 26 Record):**
```java
/**
 * Data transfer object for user information.
 * 
 * @param id unique identifier (non-null)
 * @param email user email address (non-null)
 */
public record UserDTO(String id, String email) {
    public UserDTO {
        Objects.requireNonNull(id, "id must not be null");
        Objects.requireNonNull(email, "email must not be null");
    }
}
```

### Convert Field Injection to Constructor Injection

**Before (Anti-pattern: Field @Autowired):**
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public UserResponse createUser(CreateUserRequest request) { }
}
```

**After (Correct: Constructor Injection):**
```java
/**
 * Service for user management operations.
 */
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = Objects.requireNonNull(userRepository);
    }
    
    /**
     * Creates a new user account.
     * 
     * @param request user creation request (non-null)
     * @return UserResponse with created user details
     * @throws EmailAlreadyExistsException if email is already registered
     */
    public UserResponse createUser(CreateUserRequest request) { }
}
```

## Lombok Policy (Context-Aware)

This specification uses **context-aware Lombok rules**:

### Always Allowed
- `@Slf4j` — logging on ANY class (does not count as "project uses Lombok")

### Conditional (based on existing code)
- **If** your project already uses Lombok on entity/object classes (`@Data`, `@Builder`, `@Getter`, `@Setter`):
  - ALL non-record classes MUST follow the same pattern
  - Prefer: `@Builder` + `@Getter` + `@NoArgsConstructor` + `@AllArgsConstructor`
- **If** your project does NOT use Lombok on entity/object classes:
  - Use manual getters/setters only
  - No Lombok except `@Slf4j`

**Detection Rule:** Scan existing entity/model/domain classes — if ≥1 uses `@Data`/`@Builder`/`@Getter`/`@Setter` → project uses Lombok.

## OpenAPI Documentation Standard

**Key Rules:**

1. **Single Source of Truth:** Static YAML file at `src/main/resources/static/openapi.yaml`
2. **Zero Annotations in Code:** Forbidden → `@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition`
3. **Swagger UI Configuration:**
   ```yaml
   springdoc:
     api-docs:
       enabled: false
     swagger-ui:
       url: /openapi.yaml
   ```
4. **All Documentation in YAML:** Endpoints, schemas, examples, error responses

**Rationale:** Separating API documentation from code prevents annotation clutter, reduces dependencies, and creates versioned documentation independent from code changes.

## Mandatory application.yml Configuration

Every microservice MUST include this configuration:

```yaml
spring:
  jackson:
    deserialization:
      fail-on-null-for-primitives: false   # Required in Boot 4 / Jackson 4

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0

security:
  gateway:
    token: ${GATEWAY_TOKEN:local-dev-gateway-token}
```

## Performance Baselines (SLAs)

Target performance for properly optimized services:

| Operation | P95 Latency |
|---|---|
| Single resource lookup (cached) | < 10ms |
| Single resource lookup (database) | < 50ms |
| Resource creation | < 200ms |
| Paginated list (10-50 items) | < 100ms |
| Database query (optimized) | < 50ms |
| HTTP endpoint round-trip | < 200ms |

**Resource Limits:**
- JVM Heap: 512MB minimum, 1024MB typical, 2048MB maximum
- Database connections: 5-20 per service
- Thread pool: 10-50 (depends on CPU cores)
- Container memory: 512MB-1GB

## Common Pitfalls to Avoid

❌ **Don't:**
- Use POJOs for DTOs (use Records)
- Use `@Autowired` field injection (use constructor injection)
- Write methods > 30 lines (extract helpers)
- Write classes > 300 lines (split classes)
- Skip JavaDoc on public classes/methods
- Use `application.properties` (use `application.yml`)
- Put Swagger annotations in code (use static `openapi.yaml`)
- Use `LocalDateTime` (use `Instant`)
- Use `System.out.println()` (use logger)
- Hardcode configuration values (use environment variables)

✅ **Do:**
- Use Records for all DTOs/Requests/Responses
- Use constructor injection for dependencies
- Keep methods focused and < 30 lines
- Write complete JavaDoc (100% coverage)
- Use `application.yml` with Spring profiles
- Maintain static API documentation
- Use `java.time.Instant` for timestamps
- Use `@Slf4j` and structured logging
- Use environment variables for configuration
- Run `mvn verify` before every commit

## Contributing

This specification is maintained by **Emerson Lima** ([github.com/Emersondll](https://github.com/Emersondll)).

**Contribution Guidelines:**
1. All specifications are in English
2. Follow Conventional Commits: `docs: update architecture section`
3. Reference spec files when adding new rules
4. Validate changes against existing specifications
5. Update the index (00-INDEX.md) when adding new specs
6. Run validation: `mvn verify` must be green

## License & Usage

This documentation is provided as guidance for enterprise Java microservice development.

✅ **You may:**
- Use in your projects
- Share with team members
- Adapt specifications to your standards
- Reference in code reviews
- Use as training material

## Quick Reference Checklists

### Pre-Commit Validation
- [ ] `mvn clean verify` passes (green)
- [ ] JaCoCo coverage ≥ 80%
- [ ] Zero SonarQube critical issues
- [ ] Zero Checkstyle violations
- [ ] Zero SpotBugs critical issues
- [ ] All public classes/methods have complete JavaDoc
- [ ] No hardcoded credentials or secrets
- [ ] All methods < 30 lines
- [ ] All classes < 300 lines

### Code Review Focus Areas
1. **Records vs POJOs** — All DTOs must be Records
2. **Dependency Injection** — Constructor-based only
3. **JavaDoc Coverage** — 100% on public members
4. **Method Length** — < 30 lines maximum
5. **Class Cohesion** — < 300 lines maximum
6. **Error Handling** — Centralized with @RestControllerAdvice
7. **API Contracts** — Static openapi.yaml (no annotations in code)
8. **Test Coverage** — Services 90%, Controllers 70%

## Support & Getting Help

### Learning Resources
1. **New to the project?** Start with [spec/00-INDEX.md](spec/00-INDEX.md)
2. **Need architecture guidance?** Read [spec/01-ARCHITECTURE.md](spec/01-ARCHITECTURE.md)
3. **Refactoring with AI?** Follow [spec/08-REFACTORING-CHECKLIST.md](spec/08-REFACTORING-CHECKLIST.md)
4. **Creating a new microservice?** Use [spec/22-DOCUMENTATION.md](spec/22-DOCUMENTATION.md) as template

### External References
- [Spring Boot 4.0.7 Documentation](https://spring.io/projects/spring-boot)
- [Java 26 Feature Guide](https://openjdk.org/projects/jdk/26/)
- [OpenAPI 3.0 Specification](https://spec.openapis.org/oas/v3.0.3)
- [MongoDB 7.0 Documentation](https://docs.mongodb.com/manual/)
- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)

## Version Information

This specification is current as of:

| Component | Version | Last Updated |
|---|---|---|
| Java | 26 | 2026-06-18 |
| Spring Boot | 4.0.7 | 2026-06-18 |
| Spring Cloud | 2025.1.0 | 2026-06-18 |
| Maven | 3.9.9 | 2026-06-18 |
| MongoDB | 7.0 | 2026-06-18 |

---

**Maintained by [Emerson Lima](https://github.com/Emersondll)**

**Happy Refactoring! 🚀**

Remember: Enterprise-quality code is not about perfection — it's about **maintainability, scalability, and team efficiency**.
