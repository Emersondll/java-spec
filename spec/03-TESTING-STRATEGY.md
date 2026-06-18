# Testing Strategy - Spring Boot Java 26

## Coverage Requirements

| Layer | Target Coverage | Includes | Excludes |
|-------|-----------------|----------|----------|
| **Services** | 90% | Business logic, edge cases | Property getters/setters |
| **Controllers** | 70% | Happy path, error handlers | Spring binding logic |
| **Repositories** | 60% | Custom queries | Spring Data auto-generated CRUD |
| **Records/DTOs** | Not required | N/A | Auto-generated equals/hashCode/toString |
| **Configuration** | 0% | Not tested | Spring configuration beans |

**Overall Target: 80% minimum coverage**

## Test Structure

```
src/test/java/
├── com/company/domain/
│   └── UserServiceTest.java
├── com/company/presentation/
│   └── UserControllerTest.java
├── com/company/infrastructure/
│   └── UserRepositoryTest.java
└── com/company/integration/
    └── UserApiIntegrationTest.java
```

## Unit Testing with Mockito

### Service Layer Testing

```java
/**
 * Unit tests for UserService using Mockito mocks.
 *
 * Tests focus on:
 * - Happy path scenarios
 * - Exception handling
 * - Correct method invocations on dependencies
 * - Edge cases and boundary conditions
 *
 * Does NOT test:
 * - Database persistence (mocked)
 * - HTTP layer (integration tests)
 * - Spring configuration
 */
@ExtendWith(MockitoExtension.class)
@DisplayName("UserService")
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @Mock
    private PasswordEncoder passwordEncoder;

    @Mock
    private MeterRegistry meterRegistry;

    @InjectMocks
    private UserService userService;

    /**
     * Test: Successfully create user with valid request.
     *
     * Scenario:
     * - Repository.save returns new user with ID
     * - Email service is called asynchronously
     * - Response contains all user details
     *
     * Verifies:
     * - User persisted with correct data
     * - Email service invoked
     * - Response maps correctly
     */
    @DisplayName("should create user successfully with valid request")
    @Test
    void shouldCreateUserSuccessfully() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "John Doe",
            "john@example.com",
            "SecureP@ss123"
        );

        User savedUser = new User();
        savedUser.setId("user-id-001");
        savedUser.setName("John Doe");
        savedUser.setEmail("john@example.com");
        savedUser.setStatus(UserStatus.PENDING_VERIFICATION);
        savedUser.setCreatedAt(Instant.now());

        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        when(passwordEncoder.encode("SecureP@ss123")).thenReturn("$2a$10$encrypted");

        // Act
        UserResponse result = userService.createUser(request);

        // Assert
        assertThat(result)
            .isNotNull()
            .satisfies(response -> {
                assertThat(response.id()).isEqualTo("user-id-001");
                assertThat(response.name()).isEqualTo("John Doe");
                assertThat(response.email()).isEqualTo("john@example.com");
                assertThat(response.status()).isEqualTo(UserStatus.PENDING_VERIFICATION);
            });

        // Verify interactions
        verify(userRepository, times(1)).save(any(User.class));
        verify(emailService, times(1)).sendVerificationEmail("john@example.com");
    }

    /**
     * Test: Throw exception when email already exists.
     *
     * Scenario:
     * - User attempts to register with existing email
     * - Repository returns true for existsByEmail
     *
     * Verifies:
     * - Exception is thrown
     * - Exception message contains email
     * - No user is persisted
     * - No email is sent
     */
    @DisplayName("should throw exception when email already exists")
    @Test
    void shouldThrowExceptionWhenEmailExists() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "Jane Doe",
            "existing@example.com",
            "SecureP@ss123"
        );

        when(userRepository.existsByEmail("existing@example.com")).thenReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(ConflictException.class)
            .hasMessageContaining("existing@example.com");

        verify(userRepository, never()).save(any());
        verify(emailService, never()).sendVerificationEmail(anyString());
    }

    /**
     * Test: Throw ResourceNotFoundException when user not found by ID.
     *
     * Scenario:
     * - Repository returns empty Optional for non-existent user ID
     *
     * Verifies:
     * - ResourceNotFoundException is thrown
     * - Repository was queried correctly
     */
    @DisplayName("should throw ResourceNotFoundException when user not found")
    @Test
    void shouldThrowWhenUserNotFound() {
        // Arrange
        when(userRepository.findById("non-existent-id")).thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> userService.findUserById("non-existent-id"))
            .isInstanceOf(ResourceNotFoundException.class);

        verify(userRepository, times(1)).findById("non-existent-id");
    }

    /**
     * Test: Parametrized test for multiple email formats.
     *
     * Test Cases:
     * - valid@email.com → valid
     * - user.name@domain.co.uk → valid
     * - invalid.email → invalid
     * - user@.com → invalid
     */
    @DisplayName("should validate various email formats")
    @ParameterizedTest
    @CsvSource({
        "john@example.com, true",
        "user.name@domain.co.uk, true",
        "invalid.email, false",
        "user@.com, false",
        "plaintext, false"
    })
    void shouldValidateEmailFormats(String email, boolean expected) {
        if (expected) {
            assertThatCode(() -> userService.validateEmail(email))
                .doesNotThrowAnyException();
        } else {
            assertThatThrownBy(() -> userService.validateEmail(email))
                .isInstanceOf(IllegalArgumentException.class);
        }
    }
}
```

## Integration Testing with Spring Context

> **Detect from `pom.xml`:** If `spring-boot-starter-data-mongodb` is present → use the MongoDB pattern below. If `spring-boot-starter-data-jpa` is present → use the JPA/Relational pattern. Never mix them.

### MongoDB Projects — Repository Tests

```java
/**
 * Integration tests for UserRepository with real MongoDB via TestContainers.
 *
 * Uses @DataMongoTest which:
 * - Loads only Spring Data MongoDB configuration
 * - Does NOT load the full application context (faster)
 * - Requires a running MongoDB instance (provided by TestContainers)
 *
 * Tests focus on:
 * - Custom query correctness
 * - Database constraints (unique indexes)
 * - Query behaviour with real MongoDB
 */
@DataMongoTest
@Testcontainers
@DisplayName("UserRepository")
class UserRepositoryTest {

    @Container
    static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void setMongoProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
    }

    @Autowired
    private UserRepository repository;

    @BeforeEach
    void cleanUp() {
        repository.deleteAll();
    }

    /**
     * Test: Save and retrieve user by email.
     *
     * Verifies:
     * - User is persisted with all fields
     * - Query returns correct user
     */
    @DisplayName("should save and find user by email")
    @Test
    void shouldSaveAndFindByEmail() {
        // Arrange
        User user = new User();
        user.setName("John Doe");
        user.setEmail("john@example.com");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());

        // Act
        User saved = repository.save(user);
        Optional<User> found = repository.findByEmail("john@example.com");

        // Assert
        assertThat(found)
            .isPresent()
            .get()
            .satisfies(u -> {
                assertThat(u.getId()).isEqualTo(saved.getId());
                assertThat(u.getName()).isEqualTo("John Doe");
                assertThat(u.getEmail()).isEqualTo("john@example.com");
            });
    }

    /**
     * Test: Find users by status and creation date.
     *
     * Verifies:
     * - Query returns only matching users
     * - Date range filtering works correctly
     * - Result set has correct count
     */
    @DisplayName("should find users by status and creation date")
    @Test
    void shouldFindByStatusAndCreatedAfter() {
        // Arrange
        Instant cutoffDate = Instant.now().minusSeconds(86400);

        User active1 = new User();
        active1.setName("Active User 1");
        active1.setEmail("active1@example.com");
        active1.setStatus(UserStatus.ACTIVE);
        active1.setCreatedAt(Instant.now());

        User active2 = new User();
        active2.setName("Active User 2");
        active2.setEmail("active2@example.com");
        active2.setStatus(UserStatus.ACTIVE);
        active2.setCreatedAt(Instant.now());

        User inactive = new User();
        inactive.setName("Inactive User");
        inactive.setEmail("inactive@example.com");
        inactive.setStatus(UserStatus.INACTIVE);
        inactive.setCreatedAt(Instant.now());

        repository.saveAll(List.of(active1, active2, inactive));

        // Act
        List<User> activeUsers = repository.findByStatusAndCreatedAtAfter(
            UserStatus.ACTIVE, cutoffDate
        );

        // Assert
        assertThat(activeUsers)
            .hasSize(2)
            .extracting("email")
            .containsExactlyInAnyOrder("active1@example.com", "active2@example.com");
    }
}
```

### Relational (JPA) Repository Integration Tests

```java
/**
 * Integration tests for UserRepository with real PostgreSQL via TestContainers.
 *
 * Uses @DataJpaTest which:
 * - Loads only Spring Data JPA repositories, DataSource, and transaction manager
 * - Does NOT load the full application context (faster)
 * - Requires a running database instance (provided by TestContainers)
 */
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
@DisplayName("UserRepository JPA")
class UserRepositoryJpaTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void setDatasourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository repository;

    @BeforeEach
    void cleanUp() {
        repository.deleteAll();
    }

    /**
     * Test: Save and retrieve user by email.
     */
    @DisplayName("should save and find user by email")
    @Test
    void shouldSaveAndFindByEmail() {
        // Arrange
        User user = new User();
        user.setName("John Doe");
        user.setEmail("john@example.com");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());

        // Act
        User saved = repository.save(user);
        Optional<User> found = repository.findByEmail("john@example.com");

        // Assert
        assertThat(found)
            .isPresent()
            .get()
            .satisfies(u -> {
                assertThat(u.getId()).isEqualTo(saved.getId()); // Long ID
                assertThat(u.getName()).isEqualTo("John Doe");
                assertThat(u.getEmail()).isEqualTo("john@example.com");
            });
    }
}
```

### Service Integration Tests

```java
/**
 * Integration tests for UserService with full Spring context and real MongoDB.
 *
 * Uses @SpringBootTest which:
 * - Loads full application context
 * - Uses a real MongoDB instance (run via Docker before tests)
 * - Tests service layer with real Spring beans
 *
 * Tests focus on:
 * - End-to-end service behaviour
 * - Spring dependency integration
 */
@SpringBootTest
@ActiveProfiles("test")
@DisplayName("UserService Integration Tests")
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository repository;

    @BeforeEach
    void cleanUp() {
        repository.deleteAll();
    }

    /**
     * Test: Create user and verify persistence.
     *
     * Scenario:
     * - Create user via service
     * - Verify record exists in database
     * - Verify all fields are persisted correctly
     */
    @DisplayName("should create and persist user")
    @Test
    void shouldCreateAndPersistUser() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "Integration Test User",
            "integration@example.com",
            "SecureP@ss123"
        );

        // Act
        UserResponse created = userService.createUser(request);

        // Assert
        User persisted = repository.findById(created.id())
            .orElseThrow(() -> new AssertionError("User not persisted"));

        assertThat(persisted)
            .satisfies(user -> {
                assertThat(user.getName()).isEqualTo("Integration Test User");
                assertThat(user.getEmail()).isEqualTo("integration@example.com");
                assertThat(user.getStatus()).isEqualTo(UserStatus.PENDING_VERIFICATION);
            });
    }
}
```

> **Note:** MongoDB does not support multi-document transactions on a standalone node.
> `@Transactional` has no effect in single-node MongoDB tests. Use it only when
> connecting to a replica set (e.g., via TestContainers `MongoDBContainer` with replica
> set mode). For standalone test databases, rely on `@BeforeEach` / `@AfterEach` to
> clean state between tests.

---

## API Integration Tests (E2E)

### MockMvc Testing

```java
/**
 * REST API integration tests using MockMvc.
 *
 * MockMvc provides:
 * - HTTP client without starting server (fast)
 * - Full Spring context
 * - Request/response validation
 *
 * Ideal for:
 * - Testing HTTP contract (status codes, headers)
 * - Endpoint validation and error handling
 * - Request/response serialization
 */
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@DisplayName("UserController API Tests")
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private UserRepository repository;

    @BeforeEach
    void cleanUp() {
        repository.deleteAll();
    }

    /**
     * Test: POST endpoint creates user and returns 201.
     *
     * Scenario:
     * - Send valid CreateUserRequest
     * - Verify 201 Created status
     * - Verify response contains user details
     * - Verify Location header is set
     */
    @DisplayName("should POST /api/v1/users and return 201 Created")
    @Test
    void shouldCreateUserViaPost() throws Exception {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "John Doe",
            "john@example.com",
            "SecureP@ss123"
        );

        String requestJson = objectMapper.writeValueAsString(request);

        // Act & Assert
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestJson))
            .andExpect(status().isCreated())
            .andExpect(header().exists(HttpHeaders.LOCATION))
            .andExpect(jsonPath("$.id").isString())
            .andExpect(jsonPath("$.name").value("John Doe"))
            .andExpect(jsonPath("$.email").value("john@example.com"))
            .andExpect(jsonPath("$.status").value("PENDING_VERIFICATION"));
    }

    /**
     * Test: GET endpoint returns 404 for non-existent user.
     *
     * Scenario:
     * - Request non-existent user ID
     * - Verify 404 Not Found status
     * - Verify error response format matches 15-ERROR-RESPONSE.md standard
     */
    @DisplayName("should GET /api/v1/users/{id} return 404 when not found")
    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        // Act & Assert
        mockMvc.perform(get("/api/v1/users/non-existent-id"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("RESOURCE_NOT_FOUND"))
            .andExpect(jsonPath("$.message").exists());
    }

    /**
     * Test: POST with invalid request returns 400 Bad Request.
     *
     * Scenario:
     * - Send request with invalid email
     * - Verify 400 Bad Request status
     * - Verify validation error code matches standard (VALIDATION_ERROR)
     */
    @DisplayName("should POST /api/v1/users return 400 with invalid email")
    @Test
    void shouldReturn400WithInvalidEmail() throws Exception {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "John Doe",
            "invalid-email",
            "SecureP@ss123"
        );

        String requestJson = objectMapper.writeValueAsString(request);

        // Act & Assert
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestJson))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.code").value("VALIDATION_ERROR"))
            .andExpect(jsonPath("$.message").exists());
    }
}
```

## TestContainers for Real Database

```java
/**
 * Integration tests with real MongoDB via TestContainers.
 *
 * TestContainers provides:
 * - Real MongoDB instance in Docker
 * - Automatic cleanup after tests
 * - Realistic database behaviour (indexes, constraints)
 *
 * Use for:
 * - Testing complex queries
 * - Testing index behaviour
 * - Testing aggregation pipelines
 */
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
@DisplayName("User Repository with Real MongoDB")
class UserRepositoryWithContainerTest {

    @Container
    static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void setMongoProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
    }

    @Autowired
    private UserRepository repository;

    @BeforeEach
    void cleanUp() {
        repository.deleteAll();
    }

    /**
     * Test: Persist and retrieve a user on a real MongoDB instance.
     *
     * Verifies:
     * - Document is persisted with auto-generated String ID
     * - Retrieved document matches saved document
     */
    @DisplayName("should persist and retrieve user with real MongoDB")
    @Test
    void shouldPersistAndRetrieveUser() {
        // Arrange
        User user = new User();
        user.setName("Test User");
        user.setEmail("test@example.com");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());

        // Act
        User saved = repository.save(user);

        // Assert
        assertThat(saved.getId()).isNotNull().isNotEmpty();
        assertThat(repository.findById(saved.getId()))
            .isPresent()
            .get()
            .satisfies(found -> {
                assertThat(found.getName()).isEqualTo("Test User");
                assertThat(found.getEmail()).isEqualTo("test@example.com");
            });
    }
}
```

## Test Naming Convention

```
should[ExpectedBehavior]When[Condition]
should[ExpectedBehavior]WithInvalidInput
should[ExpectedBehavior]AndReturn[HttpStatus]

Examples:
- shouldCreateUserSuccessfully
- shouldThrowExceptionWhenEmailExists
- shouldReturn404WhenUserNotFound
- shouldReturn400WithInvalidEmail
- shouldUpdateUserWhenIdExists
- shouldDeleteUserAndCascadeOrders
- shouldListUsersWithPagination
```

## JaCoCo Coverage Configuration (Maven)

```xml
<!-- pom.xml — jacoco-maven-plugin -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
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

Run coverage verification:

```bash
./mvnw verify
```

Generate HTML report only:

```bash
./mvnw jacoco:report
```

## CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests and Coverage
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '26'
          distribution: 'temurin'

      - name: Run tests and coverage check
        run: ./mvnw verify

      - name: Generate coverage report
        run: ./mvnw jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./target/site/jacoco/jacoco.xml
          fail_ci_if_error: true
```

> **MongoDB in CI:** The TestContainers setup starts a real `mongo:7.0` Docker container
> automatically during the test run. No separate `services:` block is needed in the CI
> workflow when using TestContainers — Docker must be available on the runner (it is on
> `ubuntu-latest`).

## Test Checklist

- [ ] All service methods have unit tests
- [ ] Happy path and error paths tested
- [ ] Edge cases covered (nulls, empty lists, boundaries)
- [ ] Parametrized tests for multiple scenarios
- [ ] Mocks verify correct method calls
- [ ] Repository integration tests use `@DataMongoTest` or `@DataJpaTest` + TestContainers
- [ ] Full integration tests use `@SpringBootTest` + `@ActiveProfiles("test")`
- [ ] API tests verify HTTP contract with MockMvc
- [ ] Error codes match `15-ERROR-RESPONSE.md` (`VALIDATION_ERROR`, `RESOURCE_NOT_FOUND`, etc.)
- [ ] Coverage > 80% (`./mvnw verify` passes JaCoCo gate)
- [ ] No `@Disabled` tests (remove or implement)
- [ ] Test names follow `should[Behavior]When[Condition]` pattern
- [ ] `@BeforeEach` cleans database state (MongoDB or Relational) instead of relying on `@Transactional`
