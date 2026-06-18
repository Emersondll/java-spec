# 19 - Spring Profiles

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory profile convention for all microservices.
> Configuration separation by environment is required — never mix production credentials into the base `application.yml`.

> **Detection:** Check `pom.xml` to determine database type:
> - `spring-boot-starter-data-mongodb` → use MongoDB configuration blocks
> - `spring-boot-starter-data-jpa` → use JPA/Relational configuration blocks

---

## 1. Required Profiles

| Profile | File | When active |
|---|---|---|
| (base) | `application.yml` | Always — shared settings and non-sensitive defaults |
| `dev` | `application-dev.yml` | Local development |
| `test` | `application-test.yml` | Automated tests (`@ActiveProfiles("test")`) |
| `prod` | `application-prod.yml` | Production — no defaults, all values via env var |

---

## 2. `application.yml` — Shared Base

Contains **only** what is identical across all environments:

### MongoDB projects

```yaml
spring:
  application:
    name: ${SERVICE_NAME}
  jackson:
    deserialization:
      fail-on-null-for-primitives: false
  data:
    mongodb:
      auto-index-creation: false        # always false; each profile defines the URI

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
  tracing:
    sampling:
      probability: 1.0

springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    url: /openapi.yaml

security:
  gateway:
    token: ${GATEWAY_TOKEN}             # no default — fail fast if not provided
```

### JPA/Relational projects

```yaml
spring:
  application:
    name: ${SERVICE_NAME}
  jackson:
    deserialization:
      fail-on-null-for-primitives: false
  datasource:
    url: ${DATASOURCE_URL}
    username: ${DATASOURCE_USERNAME}
    password: ${DATASOURCE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate                # NEVER create/update — Flyway manages schema
    open-in-view: false                 # prevent lazy-loading outside transaction
    show-sql: false
  flyway:
    enabled: true
    locations: classpath:db/migration

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
  tracing:
    sampling:
      probability: 1.0

springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    url: /openapi.yaml

security:
  gateway:
    token: ${GATEWAY_TOKEN}             # no default — fail fast if not provided
```

**Forbidden in the base file:**
- Database or broker credentials (even as prod defaults)
- Hardcoded production URLs
- `auto-index-creation: true` (MongoDB) or `ddl-auto: create` (JPA)

---

## 3. `application-dev.yml` — Local Development

### MongoDB projects

```yaml
spring:
  profiles:
    active: dev
  data:
    mongodb:
      uri: mongodb://localhost:27017/${spring.application.name}-dev
      auto-index-creation: true         # convenient for dev
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

security:
  gateway:
    token: local-dev-gateway-token      # safe local default for dev only

logging:
  level:
    root: INFO
    com.yourcompany: DEBUG
    org.springframework.data.mongodb: DEBUG

management:
  tracing:
    sampling:
      probability: 1.0                  # 100% in dev to simplify debugging
```

> Docker: `docker run -d --rm -p 27017:27017 mongo:7.0`

### JPA/Relational projects

```yaml
spring:
  profiles:
    active: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/${spring.application.name}_dev
    username: dev
    password: dev
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
  flyway:
    enabled: true
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

security:
  gateway:
    token: local-dev-gateway-token

logging:
  level:
    root: INFO
    com.yourcompany: DEBUG
    org.springframework.orm.jpa: DEBUG

management:
  tracing:
    sampling:
      probability: 1.0
```

> Docker: `docker run -d --rm -p 5432:5432 -e POSTGRES_PASSWORD=dev -e POSTGRES_USER=dev -e POSTGRES_DB=myservice_dev postgres:16-alpine`

---

## 4. `application-test.yml` — Automated Tests

### MongoDB projects

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/${spring.application.name}-test
      auto-index-creation: true
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

security:
  gateway:
    token: test-gateway-token

logging:
  level:
    root: WARN
    com.yourcompany: INFO

management:
  tracing:
    sampling:
      probability: 0.0                  # disable tracing in tests
```

**How to activate in tests:**

```java
@SpringBootTest
@ActiveProfiles("test")
class UserServiceIntegrationTest {
    // MongoDB must be running: docker run -d --rm -p 27017:27017 mongo:7.0
}
```

### JPA/Relational projects

```yaml
spring:
  datasource:
    url: ${TEST_DATASOURCE_URL:jdbc:tc:postgresql:16-alpine:///testdb}
  jpa:
    hibernate:
      ddl-auto: create-drop             # TestContainers manages lifecycle
  flyway:
    enabled: false                      # ddl-auto handles schema in tests

security:
  gateway:
    token: test-gateway-token

logging:
  level:
    root: WARN
    com.yourcompany: INFO

management:
  tracing:
    sampling:
      probability: 0.0
```

> Use the `jdbc:tc:postgresql:...` Testcontainers JDBC URL or `@DynamicPropertySource` + `PostgreSQLContainer` to inject the datasource URL at runtime.

**How to activate in tests:**

```java
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceIntegrationTest {
    // PostgreSQL started automatically by Testcontainers JDBC URL
}
```

---

## 5. `application-prod.yml` — Production

### MongoDB projects

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}               # REQUIRED via env var
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USER}
    password: ${RABBITMQ_PASSWORD}
    virtual-host: ${RABBITMQ_VHOST:/}

logging:
  level:
    root: WARN
    com.yourcompany: INFO
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%p","service":"${spring.application.name}","traceId":"%X{traceId}","message":"%m"}%n'

management:
  tracing:
    sampling:
      probability: 0.1                  # 10% in production (adjust per load)
  endpoint:
    health:
      show-details: never               # do not expose details to unauthenticated callers
```

### JPA/Relational projects

```yaml
spring:
  datasource:
    url: ${DATASOURCE_URL}              # Required — no default
    username: ${DATASOURCE_USERNAME}    # Required — no default
    password: ${DATASOURCE_PASSWORD}    # Required — no default
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USER}
    password: ${RABBITMQ_PASSWORD}
    virtual-host: ${RABBITMQ_VHOST:/}

logging:
  level:
    root: WARN
    com.yourcompany: INFO
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%p","service":"${spring.application.name}","traceId":"%X{traceId}","message":"%m"}%n'

management:
  tracing:
    sampling:
      probability: 0.1
  endpoint:
    health:
      show-details: never
```

**Production rules (both database types):**
- **No default values** for credentials — `${VAR}` without `: default`
- Application MUST fail to start if a required env var is absent
- Structured JSON logging for Loki collection
- `auto-index-creation: false` (MongoDB) / `ddl-auto: validate` (JPA) — schema changes via migrations only

---

## 6. Activation per Environment

### Docker Compose (dev)
```yaml
services:
  user-service:
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - GATEWAY_TOKEN=local-dev-gateway-token
```

### Docker Compose (prod — MongoDB)
```yaml
services:
  user-service:
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - GATEWAY_TOKEN=${GATEWAY_TOKEN}
      - MONGODB_URI=${MONGODB_URI}
      - RABBITMQ_HOST=${RABBITMQ_HOST}
      - RABBITMQ_USER=${RABBITMQ_USER}
      - RABBITMQ_PASSWORD=${RABBITMQ_PASSWORD}
```

### Docker Compose (prod — JPA/Relational)
```yaml
services:
  user-service:
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - GATEWAY_TOKEN=${GATEWAY_TOKEN}
      - DATASOURCE_URL=${DATASOURCE_URL}
      - DATASOURCE_USERNAME=${DATASOURCE_USERNAME}
      - DATASOURCE_PASSWORD=${DATASOURCE_PASSWORD}
      - RABBITMQ_HOST=${RABBITMQ_HOST}
      - RABBITMQ_USER=${RABBITMQ_USER}
      - RABBITMQ_PASSWORD=${RABBITMQ_PASSWORD}
```

### CI/CD (GitHub Actions)
```yaml
- name: Run tests
  run: mvn verify -P test
  env:
    SPRING_PROFILES_ACTIVE: test
    GATEWAY_TOKEN: test-gateway-token
```

---

## 7. Fail-Fast for Required Variables

In production, validate required variables at context startup:

### MongoDB projects

```java
/**
 * Security configuration validated at context initialization.
 * Fails immediately if required environment variables are absent.
 */
@Configuration
public class AppConfig {

    /**
     * Gateway token obtained mandatorily from the environment.
     * Absence of this variable causes a startup failure.
     *
     * @param gatewayToken internal gateway authentication token
     * @throws IllegalArgumentException if the token is null or blank
     */
    @Bean
    public String gatewayToken(@Value("${security.gateway.token}") String gatewayToken) {
        if (gatewayToken == null || gatewayToken.isBlank()) {
            throw new IllegalArgumentException("GATEWAY_TOKEN env var is required and must not be blank");
        }
        return gatewayToken;
    }
}
```

### JPA/Relational projects

```java
/**
 * Startup validation for required JPA datasource environment variables.
 * Application fails fast rather than reaching a broken runtime state.
 */
@Configuration
public class AppConfig {

    @Bean
    public String gatewayToken(@Value("${security.gateway.token}") String gatewayToken) {
        if (gatewayToken == null || gatewayToken.isBlank()) {
            throw new IllegalArgumentException("GATEWAY_TOKEN env var is required and must not be blank");
        }
        return gatewayToken;
    }

    @Bean
    public String datasourceValidation(
            @Value("${spring.datasource.url:}") String datasourceUrl,
            @Value("${spring.datasource.username:}") String datasourceUsername,
            @Value("${spring.datasource.password:}") String datasourcePassword) {
        if (datasourceUrl.isBlank()) {
            throw new IllegalArgumentException("DATASOURCE_URL env var is required");
        }
        if (datasourceUsername.isBlank()) {
            throw new IllegalArgumentException("DATASOURCE_USERNAME env var is required");
        }
        if (datasourcePassword.isBlank()) {
            throw new IllegalArgumentException("DATASOURCE_PASSWORD env var is required");
        }
        return "datasource-validated";
    }
}
```

---

## 8. Per-Microservice Checklist

- [ ] `application.yml` contains no credentials or production URLs
- [ ] `application-dev.yml` with local defaults (MongoDB localhost or PostgreSQL dev container)
- [ ] `application-test.yml` with `@ActiveProfiles("test")` on integration tests
- [ ] `application-prod.yml` with all credentials as `${VAR}` without defaults
- [ ] `SPRING_PROFILES_ACTIVE` set in the production Dockerfile/Compose
- [ ] MongoDB: `auto-index-creation: false` in base profile
- [ ] JPA: `ddl-auto: validate` in base and prod; Flyway enabled
- [ ] Structured JSON logging enabled only in the `prod` profile
- [ ] Tracing with `probability: 0.1` in prod and `1.0` in dev/test
- [ ] `.env` files in `.gitignore`
- [ ] Fail-fast documented for all required variables

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
