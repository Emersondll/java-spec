# 20 - Maven POM Standard

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory `pom.xml` structure for every microservice.
> Copy this template and replace `<artifactId>` and `<name>` — everything else is standard.

---

## 1. Parent POM

Every service MUST inherit from the Spring Boot parent. Version is pinned here; never override starter versions manually.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.7</version>
    <relativePath/>
</parent>
```

---

## 2. Properties Block

```xml
<properties>
    <java.version>26</java.version>
    <spring-cloud.version>2025.1.0</spring-cloud.version>
    <maven.compiler.source>26</maven.compiler.source>
    <maven.compiler.target>26</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

---

## 3. Spring Cloud BOM

Declare in `<dependencyManagement>` so all Spring Cloud starters resolve the correct version automatically.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 4. Dependencies

### 4.1 Core — Always Present

| Dependency | GroupId | ArtifactId | Notes |
|---|---|---|---|
| Web | `org.springframework.boot` | `spring-boot-starter-web` | REST layer |
| MongoDB | `org.springframework.boot` | `spring-boot-starter-data-mongodb` | For MongoDB projects — detect via `pom.xml` (see CLAUDE.md database section) |
| Security | `org.springframework.boot` | `spring-boot-starter-security` | GatewayAuthFilter |
| Validation | `org.springframework.boot` | `spring-boot-starter-validation` | Bean Validation (`@Valid`) |
| Actuator | `org.springframework.boot` | `spring-boot-starter-actuator` | Health, metrics, prometheus |
| Tracing | `io.micrometer` | `micrometer-tracing-bridge-otel` | OTLP tracing bridge |
| OTLP Exporter | `io.opentelemetry` | `opentelemetry-exporter-otlp` | Exports traces to Tempo |
| Prometheus | `io.micrometer` | `micrometer-registry-prometheus` | Metrics scraping |
| Lombok | `org.projectlombok` | `lombok` | `provided` scope — context-aware (see `02-CODE-QUALITY.md`) |
| OpenAPI UI | `org.springdoc` | `springdoc-openapi-starter-webmvc-ui` | Serves static `openapi.yaml` |

```xml
<dependencies>
    <!-- Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MongoDB (for MongoDB projects — for JPA projects use spring-boot-starter-data-jpa instead) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>

    <!-- Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Tracing bridge (OpenTelemetry) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>

    <!-- OTLP exporter → Tempo -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>

    <!-- Prometheus metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Lombok — context-aware: @Slf4j always allowed; @Builder/@Data only if project already uses them (see 02-CODE-QUALITY.md) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>

    <!-- Static OpenAPI / Swagger UI -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.8.8</version>
    </dependency>
</dependencies>
```

### 4.2 JPA/Relational Alternative — Use Instead of MongoDB When `spring-boot-starter-data-jpa` Detected

```xml
<!-- JPA (for PostgreSQL/MySQL projects — replaces spring-boot-starter-data-mongodb) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- Flyway for schema migrations (required when ddl-auto: validate) -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

> For JPA test scope, use `spring-boot-starter-test` + `org.testcontainers:postgresql` (see `03-TESTING-STRATEGY.md`).

### 4.3 Messaging — Add When Publishing or Consuming Events

```xml
<!-- RabbitMQ / SAGA events -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 4.4 Resilience — Add When Calling External Services

```xml
<!-- Circuit breaker — version managed by Spring Cloud BOM -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- Resilience4j Micrometer bridge for metrics -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-micrometer</artifactId>
</dependency>
```

### 4.5 Test Scope

```xml
<!-- Spring Boot test slice + JUnit 5 + Mockito -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Security test support (MockMvc with auth) -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>

<!--
    MongoDB for tests — choose ONE:

    Option A: Embedded MongoDB (fast, no Docker required, use for unit/slice tests)
-->
<dependency>
    <groupId>de.flapdoodle.embed</groupId>
    <artifactId>de.flapdoodle.embed.mongo.spring3x</artifactId>
    <version>4.18.0</version>
    <scope>test</scope>
</dependency>

<!--
    Option B: TestContainers (RECOMMENDED for integration tests — real MongoDB 7.0)
    Add to test scope alongside Option A or replace it for full integration tests.
-->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mongodb</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

> **Rule:** Use embedded MongoDB for `@DataMongoTest` slice tests. Use TestContainers for `@SpringBootTest` integration tests that need a real replica set or complex aggregations.

---

## 5. Maven Plugins

All plugins go inside `<build><plugins>`.

### 5.1 maven-compiler-plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>26</source>
        <target>26</target>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

### 5.2 maven-surefire-plugin (unit tests only)

Excludes integration test files so they don't run during `mvn test`.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
            <exclude>**/*IT.java</exclude>
        </excludes>
    </configuration>
</plugin>
```

### 5.3 maven-failsafe-plugin (integration tests)

Runs `*IntegrationTest` and `*IT` classes during `mvn verify`.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 5.4 jacoco-maven-plugin (coverage gate 80%)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>CLASS</element>
                        <excludes>
                            <exclude>*Config</exclude>
                            <exclude>*Application</exclude>
                            <exclude>*Filter</exclude>
                        </excludes>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 5.5 sonar-maven-plugin (optional — SonarQube)

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.11.0.3922</version>
</plugin>
```

Run with: `mvn sonar:sonar -Dsonar.token=${SONAR_TOKEN}`

### 5.6 spotbugs-maven-plugin

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <failOnError>true</failOnError>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 5.7 maven-checkstyle-plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <failsOnError>true</failsOnError>
        <consoleOutput>true</consoleOutput>
    </configuration>
    <executions>
        <execution>
            <id>validate</id>
            <phase>validate</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

---

## 6. Complete `pom.xml` — `user-service` Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.7</version>
        <relativePath/>
    </parent>

    <groupId>com.yourcompany</groupId>
    <artifactId>user-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>user-service</name>
    <description>User domain microservice</description>

    <!-- ===== PROPERTIES ===== -->
    <properties>
        <java.version>26</java.version>
        <spring-cloud.version>2025.1.0</spring-cloud.version>
        <maven.compiler.source>26</maven.compiler.source>
        <maven.compiler.target>26</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- ===== BOM MANAGEMENT ===== -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- ===== DEPENDENCIES ===== -->
    <dependencies>

        <!-- Core -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Observability -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bridge-otel</artifactId>
        </dependency>
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-exporter-otlp</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

        <!-- Lombok — @Slf4j ONLY, provided scope -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- OpenAPI static file serving -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.8.8</version>
        </dependency>

        <!-- Messaging (include if this service uses RabbitMQ) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mongodb</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <!-- ===== BUILD / PLUGINS ===== -->
    <build>
        <plugins>

            <!-- Java 26 compiler -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>26</source>
                    <target>26</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <!-- Unit tests (excludes integration tests) -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>**/*IntegrationTest.java</exclude>
                        <exclude>**/*IT.java</exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- Integration tests (runs on mvn verify) -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- JaCoCo coverage gate — 80% minimum -->
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.12</version>
                <executions>
                    <execution>
                        <id>prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>check</id>
                        <goals>
                            <goal>check</goal>
                        </goals>
                        <configuration>
                            <rules>
                                <rule>
                                    <element>CLASS</element>
                                    <excludes>
                                        <exclude>*Config</exclude>
                                        <exclude>*Application</exclude>
                                        <exclude>*Filter</exclude>
                                    </excludes>
                                    <limits>
                                        <limit>
                                            <counter>LINE</counter>
                                            <value>COVEREDRATIO</value>
                                            <minimum>0.80</minimum>
                                        </limit>
                                    </limits>
                                </rule>
                            </rules>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- SpotBugs static analysis -->
            <plugin>
                <groupId>com.github.spotbugs</groupId>
                <artifactId>spotbugs-maven-plugin</artifactId>
                <version>4.8.3.1</version>
                <configuration>
                    <effort>Max</effort>
                    <threshold>Low</threshold>
                    <failOnError>true</failOnError>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>check</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- Checkstyle -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-checkstyle-plugin</artifactId>
                <version>3.3.1</version>
                <configuration>
                    <configLocation>checkstyle.xml</configLocation>
                    <failsOnError>true</failsOnError>
                    <consoleOutput>true</consoleOutput>
                </configuration>
                <executions>
                    <execution>
                        <id>validate</id>
                        <phase>validate</phase>
                        <goals>
                            <goal>check</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- SonarQube (optional — run manually with -Dsonar.token=...) -->
            <plugin>
                <groupId>org.sonarsource.scanner.maven</groupId>
                <artifactId>sonar-maven-plugin</artifactId>
                <version>3.11.0.3922</version>
            </plugin>

        </plugins>
    </build>

</project>
```

---

## 7. Key Commands

| Command | Effect |
|---|---|
| `mvn test` | Unit tests only (excludes `*IT`, `*IntegrationTest`) |
| `mvn verify` | Unit + integration tests + JaCoCo gate + SpotBugs + Checkstyle |
| `mvn jacoco:report` | Generate HTML coverage report in `target/site/jacoco/` |
| `mvn sonar:sonar -Dsonar.token=<token>` | Push coverage + analysis to SonarQube |
| `mvn spotbugs:gui` | Open SpotBugs GUI to review findings |

---

## 8. Per-Service Checklist

- [ ] Parent is `spring-boot-starter-parent:4.0.7`
- [ ] `spring-cloud.version=2025.1.0` in `<properties>`
- [ ] Spring Cloud BOM in `<dependencyManagement>`
- [ ] No manually pinned Spring Boot starter versions (managed by parent)
- [ ] Lombok in `provided` scope only
- [ ] JaCoCo gate at 80% with `*Config`, `*Application`, `*Filter` excluded
- [ ] Surefire excludes `*IntegrationTest` and `*IT`
- [ ] Failsafe runs integration tests on `mvn verify`
- [ ] SpotBugs with `effort=Max` and `failOnError=true`
- [ ] Checkstyle runs at `validate` phase
- [ ] `mvn verify` is green before any commit
- [ ] `spring-boot-starter-amqp` added only if service publishes/consumes events
- [ ] TestContainers used for `@SpringBootTest` integration tests

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
