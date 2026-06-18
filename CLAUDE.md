# Java Coding Rules

These rules apply to **all** microservices.
Detailed documentation is in `spec/`.

---

## Required Stack

- Java 26 · Spring Boot 4.0.7 · Spring Cloud 2025.1.0
- Maven 3.9.9 · Database per-service (MongoDB **or** PostgreSQL/MySQL) · RabbitMQ (SAGA events)
- Configuration: **only** `application.yml` — `application.properties` is forbidden

## Commits — Absolute Rules

- **Never** `Co-Authored-By: Claude` or any mention of Claude
- Fixed author: `Emerson Lima <emersondll@outlook.com>` — do not change the global git config
- Conventional Commits in English: `feat:` `fix:` `refactor:` `test:` `docs:` `chore:`
- `mvn verify` must be green before any commit

## Code Structure

- DTOs: `record` with JavaDoc on 100% of fields
- Entities (non-record): **Lombok is context-aware** (see below)
- Constructor injection (no `@Autowired`)
- Methods < 30 lines · Classes < 300 lines
- Repeated string/numeric literals → extract to `private static final` constants

## Lombok Policy — Context-Aware

- `@Slf4j`: **always allowed** on any class — does NOT count as "project uses Lombok"
- **If** the project already uses Lombok on entity/object classes (`@Data`/`@Builder`/`@Getter`/`@Setter`):
  **all** non-record classes MUST follow the same Lombok pattern, prefer `@Builder` + `@Getter` + `@NoArgsConstructor` + `@AllArgsConstructor`
- **If** the project does NOT use Lombok on entity/object classes:
  manual getters/setters — no Lombok except `@Slf4j`
- **Detection rule:** scan existing entity/model/domain classes — if ≥1 uses `@Data`/`@Builder`/`@Getter`/`@Setter` → the project uses Lombok

## Database — MongoDB or Relational

- **MongoDB project:** entity IDs = `String` · repository extends `MongoRepository` · `@Document` / `@Indexed`
- **Relational project (JPA):** entity IDs = `Long` · repository extends `JpaRepository` · `@Entity` / `@Table` / `@Column`
- Detect from `pom.xml`: `spring-boot-starter-data-mongodb` vs `spring-boot-starter-data-jpa`

## OpenAPI

- **Static** file: `src/main/resources/static/openapi.yaml`
- **Forbidden** in code: `@Tag` `@Operation` `@ApiResponse` `@Schema` `@OpenAPIDefinition`
- springdoc: `api-docs.enabled: false` + `swagger-ui.url: /openapi.yaml`

## Tests and Coverage

- JUnit 5 + Mockito · JaCoCo gate >= 80%
- Services >= 90% · Controllers >= 70%
- `@SpringBootTest` requires database running:
  - MongoDB: `docker run -d --rm -p 27017:27017 mongo:7.0`
  - PostgreSQL: `docker run -d --rm -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:16-alpine`
- Exclude from coverage: `*Config`, `*Application`, `*Filter` (via `<exclude>` in JaCoCo)

## Mandatory configuration in every application.yml

```yaml
spring:
  jackson:
    deserialization:
      fail-on-null-for-primitives: false   # required in Boot 4 / Jackson 4

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

## Refactoring Skills (Claude Code — `/skill <name>`)

| Skill | Purpose |
|---|---|
| `java-refactor-context` | Compliance audit before any refactoring |
| `java-flow-refactor` | Full refactoring of a controller→service→repository flow |
| `java-tdd-unit-test` | Test generation up to JaCoCo gate 80%/90% |
| `java-javadoc` | Add Javadoc to public classes and methods |
| `java-method-extraction` | Extract long methods (> 30 lines) |
| `java-guard-clauses` | Apply early-return to nested if-else blocks |
| `java-solid-dip` | Convert field injection to constructor injection |
| `java-controller-lean` | Remove business logic from controllers |
| `java-dry-extraction` | Extract duplicated code into shared utilities |

## Detailed Spec (spec/ folder)

| File | Content |
|---|---|
| `00-INDEX.md` | Index and mandatory requirements |
| `01-ARCHITECTURE.md` | Microservices architecture |
| `01b-HEXAGONAL-ARCHITECTURE.md` | Hexagonal architecture conventions |
| `02-CODE-QUALITY.md` | Code quality and standards |
| `03-TESTING-STRATEGY.md` | Testing strategy |
| `04-PERFORMANCE.md` | Performance optimization, N+1, caching |
| `05-OBSERVABILITY.md` | OpenTelemetry, Prometheus, Grafana |
| `06-DOCKER-DEPLOYMENT.md` | Multi-stage Dockerfile, Docker Compose |
| `06b-DOCKER-OPENTELEMETRY-STACK.md` | Full observability stack |
| `07-JAVA26-RECORDS-JAVADOC.md` | Records, JavaDoc, sealed classes |
| `08-REFACTORING-CHECKLIST.md` | Pre-commit checklist |
| `09-SYNC-ASYNC-COMMUNICATION.md` | Synchronous vs asynchronous communication |
| `10-CIRCUIT-BREAKER.md` | Resilience4j, circuit breaker, fallback |
| `11-IMPORT-GUIDELINES.md` | Import standards (no wildcards) |
| `12-MANDATORY-ANNOTATIONS.md` | @Repository, @Service, @RestController |
| `13-OPENAPI-REST-REFERENCE.md` | REST and static OpenAPI reference |
| `14-SECURITY.md` | Trust boundary, GatewayAuthFilter, IDOR |
| `15-ERROR-RESPONSE.md` | Standard HTTP error envelope |
| `16-RABBITMQ-SAGA.md` | SAGA events, exchanges, DLQ, idempotency |
| `17-PAGINATION.md` | List pagination standard |
| `18-MONGODB-INDEXES.md` | MongoDB index conventions |
| `18b-RELATIONAL-INDEXES-MIGRATIONS.md` | JPA indexes and Flyway migration conventions |
| `19-SPRING-PROFILES.md` | Spring profiles (dev, test, prod) |
| `20-MAVEN-POM.md` | Standard pom.xml structure, plugins, BOM |
| `21-HTTP-IDEMPOTENCY.md` | Idempotency-Key header pattern for mutation endpoints |
| `22-DOCUMENTATION.md` | Project documentation structure, README standards, AI generation guidelines |
| `23-GIT-BRANCHING.md` | GitHub branching strategy, PR workflow, Conventional Commits, never push to main |
| `24-CICD-SECRETS.md` | CI/CD pipelines, secrets management, .env, Vault, Kubernetes, ArgoCD |
