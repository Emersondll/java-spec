# Java Spring Boot Refactoring Guidelines - Complete Specification

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)

## 📂 Complete Documentation Package in `/spec` Folder

This documentation package contains **27 specifications** for building enterprise-grade Java 26 / Spring Boot 4.0.7 microservices with MongoDB or PostgreSQL/MySQL.

### 📋 Core Architecture & Design

| File | Purpose | Key Topics |
|------|---------|-----------|
| **01-ARCHITECTURE.md** | Spring Boot layered architecture | Project structure, DI, design patterns, API versioning, exception handling |
| **01b-HEXAGONAL-ARCHITECTURE.md** | Hexagonal architecture conventions | Domain, ports, adapters, dependency flow, wiring |
| **02-CODE-QUALITY.md** | Code standards and quality enforcement | JavaDoc 100% mandatory, null-safety, static analysis (SonarQube, SpotBugs) |
| **07-JAVA26-RECORDS-JAVADOC.md** | Java 26 features and complete documentation | Records (mandatory for DTOs), sealed classes, pattern matching, JavaDoc templates |

### 🧪 Testing & Validation

| File | Purpose | Key Topics |
|------|---------|-----------|
| **03-TESTING-STRATEGY.md** | Test pyramid and coverage requirements | Unit tests (90%), integration tests, test containers, JaCoCo (80% minimum) |

### ⚡ Performance & Resilience

| File | Purpose | Key Topics |
|------|---------|-----------|
| **04-PERFORMANCE.md** | Performance optimization strategies | N+1 query elimination, caching, connection pools, JVM tuning |
| **10-CIRCUIT-BREAKER.md** | Circuit breaker pattern with Resilience4j | Fail-fast, fallback strategies, error handling, metrics |

### 📊 Observability & Monitoring

| File | Purpose | Key Topics |
|------|---------|-----------|
| **05-OBSERVABILITY.md** | Logging, metrics, and distributed tracing | Structured logging, SLF4J, Micrometer, Prometheus, OTLP → Tempo |
| **06b-DOCKER-OPENTELEMETRY-STACK.md** | Complete observability stack | OpenTelemetry, Prometheus, Loki, Tempo, Grafana - Full stack |

### 🐳 Deployment & DevOps

| File | Purpose | Key Topics |
|------|---------|-----------|
| **06-DOCKER-DEPLOYMENT.md** | Docker deployment | Dockerfile multi-stage (temurin:26), Docker Compose, health checks |

### 🔧 Code Patterns & Best Practices

| File | Purpose | Key Topics |
|------|---------|-----------|
| **09-SYNC-ASYNC-COMMUNICATION.md** | Synchronous vs asynchronous patterns | REST endpoints, @Async, CompletableFuture, message queues, timeouts |
| **11-IMPORT-GUIDELINES.md** | Import statement standards | NO wildcards, explicit imports, organization, IDE config |
| **12-MANDATORY-ANNOTATIONS.md** | Spring stereotype annotations | @Repository, @Service, @RestController, @Component, @Configuration |

### 🤖 AI Refactoring Automation

| File | Purpose | Key Topics |
|------|---------|-----------|
| **08-REFACTORING-CHECKLIST.md** | Practical AI refactoring guide | Step-by-step process, before/after examples, validation checklist |

### 🌐 API Reference

| File | Purpose | Key Topics |
|------|---------|-----------|
| **13-OPENAPI-REST-REFERENCE.md** | Complete REST API reference | **Mandatory static openapi.yaml** — zero annotations in code, all endpoints/schemas/examples in YAML |

### 🔐 Security & Contracts

| File | Purpose | Key Topics |
|------|---------|-----------|
| **14-SECURITY.md** | Security standards | Trust boundary, GatewayAuthFilter, IDOR, secrets management |
| **15-ERROR-RESPONSE.md** | Standard HTTP error envelope | `ErrorResponse` record, `GlobalExceptionHandler`, `VALIDATION_ERROR`, `INTERNAL_ERROR` |
| **21-HTTP-IDEMPOTENCY.md** | HTTP idempotency for mutation endpoints | `Idempotency-Key` header, `IdempotencyFilter`, TTL index, SHA-256 body hash |

### 📨 Messaging & Events

| File | Purpose | Key Topics |
|------|---------|-----------|
| **16-RABBITMQ-SAGA.md** | RabbitMQ & SAGA event patterns | `DomainEvent<T>` envelope, exchanges/queues/DLQ, idempotency, SAGA choreography |

### 📄 API Contracts & Patterns

| File | Purpose | Key Topics |
|------|---------|-----------|
| **17-PAGINATION.md** | Pagination standard for list endpoints | `PageResponse<T>`, `@PageableDefault`, size limits, sort conventions |

### 🗄️ Data & Configuration

| File | Purpose | Key Topics |
|------|---------|-----------|
| **18-MONGODB-INDEXES.md** | MongoDB indexing conventions | `@Indexed`, `@CompoundIndex`, TTL, naming, Mongock migrations |
| **18b-RELATIONAL-INDEXES-MIGRATIONS.md** | JPA & Relational DB migrations | JPA `@Index`, Flyway, `ddl-auto: validate` |
| **19-SPRING-PROFILES.md** | Spring profiles per environment | `application-dev/test/prod.yml`, MongoDB + JPA variants, fail-fast vars |
| **20-MAVEN-POM.md** | Standard Maven POM structure | Spring Boot parent, Spring Cloud BOM, JaCoCo, Surefire, Failsafe, SpotBugs, Checkstyle |

### 📝 Documentation Standards

| File | Purpose | Key Topics |
|------|---------|-----------|
| **22-DOCUMENTATION.md** | Project documentation structure | README standards, supplementary docs, AI generation guidelines, cross-reference rules |

### 🔀 Version Control & CI/CD

| File | Purpose | Key Topics |
|------|---------|-----------|
| **23-GIT-BRANCHING.md** | GitHub branching strategy | Branch naming, PR workflow, Conventional Commits, never push to main, hotfix flow, SemVer tags |
| **24-CICD-SECRETS.md** | CI/CD pipelines, secrets & orchestration | GitHub Actions, .env, HashiCorp Vault, Kubernetes Secrets/Deployments, ArgoCD GitOps, Trivy scan |

---

## Technology Stack

**Required Versions:**
- Java 26 (mandatory)
- Spring Boot 4.0.7
- Spring Cloud 2025.1.0
- Maven 3.9.9 (Maven Wrapper or SDKMAN)

**Key Dependencies:**
- **Database:** Spring Data MongoDB **or** Spring Data JPA + PostgreSQL/MySQL (detect from `pom.xml`)
- Spring Security (stateless JWT)
- Spring Cloud OpenFeign (BFF only)
- Lombok: **context-aware** — `@Slf4j` always allowed; `@Data`/`@Builder`/`@Getter`/`@Setter` only if project already uses them on entity classes
- Jackson 4 (JSON — configure `fail-on-null-for-primitives: false`)
- JUnit 5 + Mockito (testing)
- Micrometer Tracing + OTLP exporter (NOT Spring Cloud Sleuth / Zipkin — both deprecated)

---

## Mandatory Requirements

### 1. Java 26 Features
- ✅ **Records ONLY** for all DTOs/Requests/Responses (no POJOs)
- ✅ **Sealed classes** for restricted inheritance
- ✅ **Pattern matching** for type-safe code
- ✅ Source/Target: Java 26 in build configuration (`<java.version>26</java.version>`)
- ✅ Docker image: `eclipse-temurin:26-jre-alpine` / builder: `maven:3.9.9-eclipse-temurin-26-alpine`

### 2. Complete JavaDoc (100% Required)
- ✅ Every public class MUST have class-level JavaDoc
- ✅ Every public method MUST have method-level JavaDoc
- ✅ Every record MUST have record-level JavaDoc
- ✅ Every record field MUST be documented
- ✅ Documentation includes: purpose, parameters, return value, exceptions

### 3. Spring Boot Architecture
- ✅ **Dependency Injection**: Constructor-based only (no field @Autowired)
- ✅ **Layered Architecture**: Controller → Service → Repository → Database
- ✅ **Exception Handling**: Centralized @RestControllerAdvice
- ✅ **Transactional Consistency**: `@Transactional` at service layer (JPA projects); MongoDB standalone does not support multi-document transactions — `@Transactional` has no effect on a single node

### 4. Code Quality Standards
- ✅ Methods: < 30 lines maximum
- ✅ Classes: < 300 lines maximum
- ✅ Parameters: < 3 per method (see 02-CODE-QUALITY.md)
- ✅ Duplication: < 3%
- ✅ Null-safety: Use Optional or validation
- ✅ Logging: INFO for business events, WARN/ERROR for issues

### 5. Test Coverage
- ✅ Minimum 80% code coverage
- ✅ Services: 90% coverage
- ✅ Controllers: 70% coverage
- ✅ Unit, integration, and E2E test pyramid

### 6. HTTP REST Standards
- ✅ 201 CREATED for POST with Location header
- ✅ 200 OK for GET
- ✅ 204 NO CONTENT for DELETE/successful operations
- ✅ 400 BAD REQUEST for validation errors
- ✅ 404 NOT FOUND for missing resources
- ✅ 409 CONFLICT for uniqueness violations
- ✅ 500 INTERNAL SERVER ERROR for unexpected failures

### 7. OpenAPI Documentation — Static File (MANDATORY)
- ✅ **`openapi.yaml` in `src/main/resources/static/`** — versioned documentation independent from code
- ✅ springdoc configured to serve the static file (`swagger-ui.url: /openapi.yaml`)
- ✅ **FORBIDDEN** to use annotations in code: `@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition`
- ✅ Swagger UI accessible at `/swagger-ui.html`
- ✅ All endpoints, schemas and examples documented in YAML

---

## Using This Documentation for AI Refactoring

### For ChatGPT / Claude / Other LLMs:

1. **Understand the Context**: Read 01-ARCHITECTURE.md for system design
2. **Learn the Standards**: Read 02-CODE-QUALITY.md for code rules
3. **Refactoring Process**: Follow 08-REFACTORING-CHECKLIST.md step-by-step
4. **Verify Quality**: Check against provided quality gates

### Example Prompt for AI:

```
You are a Java Spring Boot architect helping refactor legacy code to enterprise standards.

Project: User Service (Java 26, Spring Boot 4.0.7)
Target Standards: 
- Java 26 Records for all DTOs
- Complete JavaDoc on every class/method
- Constructor injection only
- 80%+ test coverage
- Clean code (methods <30 lines, classes <300 lines)

Guidelines: 
- Refer to spec files 01-ARCHITECTURE through 24-CICD-SECRETS
- Ensure 100% JavaDoc coverage
- Convert all POJOs to Records with compact constructors
- Replace field @Autowired with constructor injection
- Validate HTTP status codes

File to refactor:
[Paste your Java code here]

Please:
1. Refactor code to meet standards
2. Add complete JavaDoc
3. List changes made
4. Provide validation checklist
```

---

## Refactoring Process Flow

```
Start
  ↓
[1. Analyze Code] → Identify DTOs, missing JavaDoc, field injection
  ↓
[2. Convert DTOs] → POJO → Records with compact constructors
  ↓
[3. Add JavaDoc] → Class, method, field-level documentation
  ↓
[4. Fix Injection] → Replace @Autowired fields with constructor
  ↓
[5. Extract Methods] → Break methods >30 lines
  ↓
[6. Fix HTTP Status] → Correct status codes for REST endpoints
  ↓
[7. Run Tests] → Ensure >80% coverage
  ↓
[8. Quality Checks] → SonarQube, Checkstyle, SpotBugs
  ↓
[9. Final Review] → Verify checklist
  ↓
End ✅
```

---

## Quick Reference Checklist

Before submitting refactored code:

### Structure & Architecture
- [ ] Project structure follows src/main/java/(domain|application|infrastructure|presentation)
- [ ] Dependency injection via constructor (no field @Autowired)
- [ ] Services return Records/DTOs, never raw entities
- [ ] Exception handling centralized (@RestControllerAdvice)
- [ ] Transactional consistency at service layer

### Java 26 & Records
- [ ] All requests/responses are Records (no POJOs)
- [ ] Records have compact constructors with validation
- [ ] Records have complete JavaDoc on class and fields
- [ ] Sealed classes used for restricted polymorphism
- [ ] Pattern matching used instead of if-else chains

### JavaDoc Coverage (100% Required)
- [ ] Every public class has class-level JavaDoc
- [ ] Every public method has method-level JavaDoc
- [ ] Every record has field documentation
- [ ] @param documented for all parameters
- [ ] @return documented for all return values
- [ ] @throws documented for all exceptions
- [ ] @since, @author, @version for classes
- [ ] Code examples in {@code} blocks where helpful

### Code Quality
- [ ] No methods > 30 lines (extract helpers)
- [ ] No classes > 300 lines (split classes)
- [ ] No methods with > 3 parameters (use DTOs)
- [ ] No null pointer risks (Optional or validation)
- [ ] No System.out.println() (use logger)
- [ ] No TODO/FIXME without ticket reference

### REST API Standards
- [ ] POST returns 201 CREATED with Location header
- [ ] GET returns 200 OK
- [ ] DELETE returns 204 NO CONTENT
- [ ] Bad request returns 400 BAD REQUEST
- [ ] Resource not found returns 404 NOT FOUND
- [ ] Duplicate returns 409 CONFLICT
- [ ] Server error returns 500 INTERNAL SERVER ERROR
- [ ] All endpoints have error handling

### Testing
- [ ] Test coverage > 80% overall
- [ ] Services tested at 90% coverage
- [ ] Controllers tested at 70% coverage
- [ ] All public methods have test cases
- [ ] Happy path and error paths covered
- [ ] Edge cases (nulls, boundaries) tested
- [ ] Test names follow should[Behavior]When[Condition]
- [ ] JaCoCo report generated successfully

### Static Analysis
- [ ] Zero SonarQube critical issues
- [ ] Zero Checkstyle violations
- [ ] Zero SpotBugs critical issues
- [ ] JavaDoc generation succeeds
- [ ] No code duplication > 3%
- [ ] Code smell = 0 critical

### Build & Deployment
- [ ] Application builds with `mvn clean package`
- [ ] All tests pass: `mvn verify` (green before any commit)
- [ ] No compiler warnings
- [ ] Dockerfile builds successfully (Alpine: `apk add`, non-root user, HEALTHCHECK via curl/wget)
- [ ] Docker image size < 200MB
- [ ] Docker image runs and responds to health checks (`/actuator/health/liveness`)
- [ ] `docker compose up` starts all services healthy

### Version Control & CI/CD
- [ ] Branch follows naming convention `feat/*/fix/*/refactor/*` — see **23-GIT-BRANCHING.md**
- [ ] PR opened to `main` with what/why/how description
- [ ] CI green (`mvn verify`) before requesting review
- [ ] No secrets or credentials committed (`.env` in `.gitignore`)
- [ ] `SPRING_PROFILES_ACTIVE` used (never `SPRING_PROFILE`)

---

## Common Refactoring Patterns

### Convert POJO to Record
```java
// Before: POJO with Lombok
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class UserDTO { private String id; private String email; }

// After: Java 26 Record
public record UserDTO(String id, String email) {
    public UserDTO { 
        Objects.requireNonNull(id);
        Objects.requireNonNull(email);
    }
}
```

### Add Constructor Injection
```java
// Before: Field injection (anti-pattern)
@Service public class UserService {
    @Autowired private UserRepository repo;
}

// After: Constructor injection
@Service public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) {
        this.repo = Objects.requireNonNull(repo);
    }
}
```

### Add Complete JavaDoc
```java
// Before: Missing documentation
public UserResponse createUser(CreateUserRequest request) { }

// After: Complete JavaDoc
/**
 * Creates a new user account with email verification.
 * [Detailed description...]
 * 
 * @param request user creation request (non-null)
 * @return UserResponse with created user details
 * @throws EmailAlreadyExistsException if email exists
 */
public UserResponse createUser(CreateUserRequest request) { }
```

---

## Performance Baselines

**Service SLAs:**
- Single user lookup: < 10ms (cached) / < 50ms (database)
- User creation: < 200ms p95
- List users (paginated): < 100ms p95
- Database query: < 50ms p95
- HTTP endpoint: < 200ms p95
- Page load: < 500ms p95

**Resource Limits:**
- JVM Heap: 512MB minimum, 1024MB typical, 2048MB maximum
- Database connections: 5-20 (connection pool size)
- Thread pool: 10-50 (depends on CPU cores)
- Container memory: 512MB-1GB

---

## Getting Help

When refactoring with AI:

1. **Read relevant section** of documentation first
2. **Provide concrete examples** of code to refactor
3. **Specify error messages** if validation fails
4. **Ask for validation checklist** after refactoring
5. **Request explanation** of changes made

---

## Document Maintenance

This documentation is current as of:
- **Java**: 26
- **Spring Boot**: 4.0.7
- **Spring Cloud**: 2025.1.0
- **MongoDB**: 7.0
- **Last Updated**: 2026-06-18

For updates or corrections, please refer to:
- [Spring Boot Official Docs](https://spring.io/projects/spring-boot)
- [Java 26 Feature Guide](https://openjdk.org/projects/jdk/26/)
- [Spring Data MongoDB Docs](https://docs.spring.io/spring-data/mongodb/reference/)

---

## Maintainer

| Name | GitHub |
|------|--------|
| **Emerson Lima** | [github.com/Emersondll](https://github.com/Emersondll) |

---

## License and Usage

This documentation is provided as guidance for code refactoring.
- ✅ Use in your projects
- ✅ Share with team members
- ✅ Adapt to your standards
- ✅ Reference in code reviews

---

## Quick Start for New Developer

1. **Start Here**: Read `01-ARCHITECTURE.md` — project structure and design patterns
2. **Learn Standards**: Read `02-CODE-QUALITY.md` — code rules, Lombok policy, static analysis
3. **Write Tests**: Read `03-TESTING-STRATEGY.md` — JaCoCo gates, @DataMongoTest / @DataJpaTest
4. **Handle Errors**: Read `15-ERROR-RESPONSE.md` — standard envelope and error codes
5. **Secure APIs**: Read `14-SECURITY.md` — GatewayAuthFilter, IDOR prevention
6. **Deploy Safely**: Read `06-DOCKER-DEPLOYMENT.md` + `24-CICD-SECRETS.md` — Docker and CI/CD
7. **Work on Git**: Read `23-GIT-BRANCHING.md` — never push to main, PR workflow
8. **Refactor Code**: Follow `08-REFACTORING-CHECKLIST.md` — step-by-step execution

**Time Investment**: ~6 hours reading + hands-on practice

---

**Happy Refactoring! 🚀**

Remember: Enterprise-quality code is not about perfection,
it's about maintainability, scalability, and team efficiency.

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
