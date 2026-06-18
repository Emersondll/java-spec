# Architecture Guidelines - Spring Boot Java 26

## Project Structure (Maven)

```
project/
├── src/main/java/
│   ├── com/company/domain/           # Entities, Domain Models, Value Objects
│   ├── com/company/application/      # Use Cases, Services, DTOs
│   ├── com/company/infrastructure/   # Repositories, API Clients, Config
│   ├── com/company/presentation/     # Controllers, REST Endpoints
│   ├── com/company/common/           # Utilities, Exceptions, Constants
│   └── com/company/Application.java
├── src/test/java/                    # Mirror main structure
├── src/main/resources/               # application.yml, logback.xml
├── docker/                           # Dockerfile, docker-compose.yml
├── docs/                             # Architecture Decision Records (ADR)
└── pom.xml
```

## Java 26 Features Required

### 1. Records for DTOs/Requests/Responses

**MANDATORY**: All request and response objects MUST use Java 26 Records.

```java
/**
 * Record for creating a new user via POST request.
 * Immutable data carrier with automatic equals/hashCode/toString.
 *
 * @param name     The user's full name (non-null, 1-100 chars)
 * @param email    The user's email address (unique, valid email format)
 * @param password The user's password (min 8 chars, hashed in service)
 */
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100)
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {
    /**
     * Compact constructor - validates all fields before instantiation.
     */
    public CreateUserRequest {
        Objects.requireNonNull(name, "Name cannot be null");
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(password, "Password cannot be null");
    }
}

/**
 * Record representing the complete user response after creation.
 * Contains sensitive-stripped data safe for API response.
 *
 * @param id        The unique user identifier (MongoDB String ID)
 * @param name      The user's full name
 * @param email     The user's email address
 * @param createdAt UTC instant of creation
 * @param status    Current user account status (ACTIVE, INACTIVE, SUSPENDED)
 */
public record UserResponse(
    String id,
    String name,
    String email,
    Instant createdAt,
    UserStatus status
) {
    /**
     * Factory method to convert Domain User entity to API response record.
     *
     * @param user the domain User entity
     * @return UserResponse record with safe data
     * @throws NullPointerException if user is null
     */
    public static UserResponse fromDomain(User user) {
        Objects.requireNonNull(user, "User cannot be null");
        return new UserResponse(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getCreatedAt(),
            user.getStatus()
        );
    }
}
```

### 2. Sealed Classes for Polymorphism

```java
/**
 * Sealed interface restricting implementations to specific user event types.
 * Enables exhaustive pattern matching and type safety.
 *
 * <p>Named {@code UserDomainEvent} to avoid conflict with the generic
 * {@code DomainEvent<T>} RabbitMQ envelope defined in spec/16-RABBITMQ-SAGA.md.</p>
 */
public sealed interface UserDomainEvent permits UserCreatedEvent, UserUpdatedEvent {
    /**
     * Returns the UTC instant when the event occurred.
     *
     * @return UTC instant of the event
     */
    Instant occurredAt();
}

/**
 * Concrete implementation of UserDomainEvent for user creation.
 * Uses record to maintain immutability and automatic value semantics.
 *
 * @param userId    The String ID of the created user (MongoDB)
 * @param email     The email of the created user
 * @param occurredAt The UTC instant of creation
 */
public record UserCreatedEvent(
    String userId,
    String email,
    Instant occurredAt
) implements UserDomainEvent {
    /**
     * Compact constructor ensuring data validity.
     */
    public UserCreatedEvent {
        Objects.requireNonNull(userId, "UserId cannot be null");
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(occurredAt, "OccurredAt cannot be null");
    }
}
```

### 3. Pattern Matching

```java
/**
 * Service demonstrating pattern matching with sealed classes and records.
 * Replaces verbose if-else or switch chains.
 */
@Service
public class EventPublisher {

    /**
     * Process user domain event using pattern matching.
     * Exhaustive matching ensures all event types are handled.
     *
     * @param event the user domain event to process
     */
    public void publish(UserDomainEvent event) {
        if (event instanceof UserCreatedEvent userCreated) {
            notifyNewUser(userCreated.userId(), userCreated.email());
        } else if (event instanceof UserUpdatedEvent userUpdated) {
            notifyUserUpdate(userUpdated.userId());
        }
    }
}
```

## Dependency Injection

All dependencies MUST be injected via constructor. NO field injection.

```java
/**
 * User service with explicit dependency injection.
 * Follows Spring best practices for testability and circular dependency prevention.
 */
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;
    private final MeterRegistry meterRegistry;

    /**
     * Constructor-based dependency injection.
     * Spring automatically wires beans in required order.
     *
     * @param userRepository  repository for user persistence
     * @param emailService    service for sending emails
     * @param passwordEncoder encoder for password hashing
     * @param meterRegistry   registry for custom metrics
     */
    public UserService(
        UserRepository userRepository,
        EmailService emailService,
        PasswordEncoder passwordEncoder,
        MeterRegistry meterRegistry
    ) {
        this.userRepository = Objects.requireNonNull(userRepository, "UserRepository cannot be null");
        this.emailService = Objects.requireNonNull(emailService, "EmailService cannot be null");
        this.passwordEncoder = Objects.requireNonNull(passwordEncoder, "PasswordEncoder cannot be null");
        this.meterRegistry = Objects.requireNonNull(meterRegistry, "MeterRegistry cannot be null");
    }

    /**
     * Creates a new user with email verification.
     *
     * @param request contains name, email, password
     * @return UserResponse with created user details
     * @throws ConflictException        if email is already registered
     * @throws IllegalArgumentException if request validation fails
     */
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        // Implementation
    }
}
```

## Repository Pattern

```java
/**
 * Repository interface for User persistence operations.
 * Extends Spring Data MongoDB for automatic CRUD implementation.
 * Define custom queries for domain-specific operations.
 */
public interface UserRepository extends MongoRepository<User, String> {

    /**
     * Find user by email address.
     * Ensures email uniqueness constraint at query level.
     *
     * @param email the email to search for
     * @return Optional containing user if found, empty otherwise
     */
    Optional<User> findByEmail(String email);

    /**
     * Find all users by status created after the specified instant.
     * Used for user retention reports and engagement metrics.
     *
     * @param status       the user status filter (e.g., ACTIVE)
     * @param createdAfter UTC instant threshold
     * @return List of matching users, empty list if none found
     */
    List<User> findByStatusAndCreatedAfter(UserStatus status, Instant createdAfter);

    /**
     * Check if email exists without loading full entity.
     * Performance optimization for validation checks.
     *
     * @param email the email to check
     * @return true if email exists, false otherwise
     */
    boolean existsByEmail(String email);
}
```

## Service Layer Design

```java
/**
 * Core business logic service for user operations.
 * Enforces transactional consistency and domain rules.
 * Always returns DTOs/Records, never raw entities.
 */
@Service
@Transactional
@Slf4j
public class UserService {

    private final UserRepository repository;
    private final MeterRegistry meterRegistry;

    /**
     * Retrieves user by String ID with error handling.
     *
     * @param id the user ID to retrieve (MongoDB String)
     * @return UserResponse if found
     * @throws ResourceNotFoundException if user does not exist
     */
    public UserResponse findUserById(String id) {
        Objects.requireNonNull(id, "User ID cannot be null");

        User user = repository.findById(id)
            .orElseThrow(() -> {
                log.warn("User not found. userId={}", id);
                meterRegistry.counter("user.not_found").increment();
                return new ResourceNotFoundException("User not found: " + id);
            });

        return UserResponse.fromDomain(user);
    }

    /**
     * Fetches paginated list of active users.
     *
     * @param pageable pagination and sorting parameters
     * @return PageResponse of UserResponse records
     */
    public PageResponse<UserResponse> listActiveUsers(Pageable pageable) {
        Page<User> page = repository.findByStatus(UserStatus.ACTIVE, pageable);
        return PageResponse.from(page.map(UserResponse::fromDomain));
    }
}
```

## Controller Layer (REST Endpoints)

```java
/**
 * REST controller for user-related HTTP endpoints.
 * Handles request validation, HTTP semantics, and response mapping.
 * Business logic delegated to service layer.
 */
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserController {

    private final UserService userService;

    /**
     * Constructor injection of user service.
     *
     * @param userService the user business logic service
     */
    public UserController(UserService userService) {
        this.userService = Objects.requireNonNull(userService, "UserService cannot be null");
    }

    /**
     * POST endpoint to create a new user.
     * Validates request constraints via @Valid annotation.
     * Returns 201 CREATED with Location header.
     *
     * @param request the user creation request record
     * @return ResponseEntity with created user response and Location header
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest request
    ) {
        log.info("Creating user. email={}", request.email());
        UserResponse created = userService.createUser(request);

        return ResponseEntity
            .created(URI.create("/api/v1/users/" + created.id()))
            .body(created);
    }

    /**
     * GET endpoint to retrieve user by ID.
     * Returns 200 OK on success, 404 NOT FOUND if user doesn't exist.
     *
     * @param id the user identifier (path variable — MongoDB String)
     * @return ResponseEntity with user response
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable String id) {
        log.debug("Retrieving user. id={}", id);
        UserResponse user = userService.findUserById(id);
        return ResponseEntity.ok(user);
    }

    /**
     * GET endpoint to list users with pagination.
     * Query parameters: page (default 0), size (default 20), sort (default createdAt,desc).
     *
     * @param pageable pagination and sorting injected by Spring
     * @return ResponseEntity with paginated user responses
     */
    @GetMapping
    public ResponseEntity<PageResponse<UserResponse>> listUsers(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable
    ) {
        log.debug("Listing users. page={}, size={}", pageable.getPageNumber(), pageable.getPageSize());
        PageResponse<UserResponse> users = userService.listActiveUsers(pageable);
        return ResponseEntity.ok(users);
    }
}
```

## Exception Handling

```java
// ErrorResponse is defined in common/dto/ErrorResponse.java — see spec/15-ERROR-RESPONSE.md

/**
 * Global exception handler providing consistent error responses.
 * Centralizes error handling across all controllers.
 * Prevents sensitive information leakage in error messages.
 */
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * Handle entity not found exceptions.
     * Returns 404 NOT FOUND with error details.
     *
     * @param ex      the thrown ResourceNotFoundException
     * @param request current HTTP request
     * @return 404 error envelope
     */
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFound(ResourceNotFoundException ex,
                                                HttpServletRequest request) {
        log.debug("Resource not found. message={}", ex.getMessage());
        return ErrorResponse.of(HttpStatus.NOT_FOUND, "RESOURCE_NOT_FOUND", ex.getMessage(), request);
    }

    /**
     * Handle validation exceptions from @Valid constraints.
     * Returns 400 BAD REQUEST with all field errors concatenated.
     *
     * @param ex      the thrown MethodArgumentNotValidException
     * @param request current HTTP request
     * @return 400 error envelope with field error details
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationError(MethodArgumentNotValidException ex,
                                               HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ErrorResponse.of(HttpStatus.BAD_REQUEST, "VALIDATION_ERROR", message, request);
    }

    /**
     * Handle uniqueness constraint violations.
     * Returns 409 CONFLICT.
     *
     * @param ex      the thrown ConflictException
     * @param request current HTTP request
     * @return 409 error envelope
     */
    @ExceptionHandler(ConflictException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleConflict(ConflictException ex, HttpServletRequest request) {
        return ErrorResponse.of(HttpStatus.CONFLICT, "CONFLICT", ex.getMessage(), request);
    }

    /**
     * Handle generic unexpected exceptions.
     * Returns 500 INTERNAL SERVER ERROR without exposing stack trace.
     *
     * @param ex      the unexpected exception
     * @param request current HTTP request
     * @return 500 error envelope with generic message
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error on {}", request.getRequestURI(), ex);
        return ErrorResponse.of(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "INTERNAL_ERROR",
            "An unexpected error occurred",
            request
        );
    }
}
```

## API Versioning Strategy

```java
// v1 endpoints - stable interface
@RequestMapping("/api/v1/users")
@RestController
public class UserV1Controller {
    // Current stable implementation
}

// v2 endpoints - breaking changes
@RequestMapping("/api/v2/users")
@RestController
public class UserV2Controller {
    // New implementation with breaking changes
    // Maintain v1 for backwards compatibility during deprecation period
}
```

## Environment Configuration

Configuration is split across profile files — see `spec/19-SPRING-PROFILES.md`.

```
application.yml      — shared base (always active)
application-dev.yml  — local development
application-test.yml — automated tests
application-prod.yml — production (all values via env var, no defaults)
```

Base `application.yml` (shared settings only, no credentials):

```yaml
spring:
  application:
    name: ${SERVICE_NAME}
  jackson:
    deserialization:
      fail-on-null-for-primitives: false
  data:
    mongodb:
      auto-index-creation: false   # always false in base; dev overrides to true

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
    token: ${GATEWAY_TOKEN}
```

## Naming Conventions

| Element | Pattern | Example |
|---------|---------|---------|
| Java Classes | PascalCase | `UserService`, `CreateUserRequest` |
| Methods | camelCase | `createUser()`, `findByEmail()` |
| Constants | UPPER_SNAKE_CASE | `DEFAULT_PAGE_SIZE = 20` |
| Package | lowercase.dot.notation | `com.company.domain.user` |
| Records | PascalCase + Suffix | `CreateUserRequest`, `UserResponse` |
| Request Records | Suffix "Request" | `CreateUserRequest`, `UpdateUserRequest` |
| Response Records | Suffix "Response" | `UserResponse`, `UserListResponse` |

## Implementation Checklist

- [ ] All requests/responses are Java 26 Records
- [ ] All classes have complete JavaDoc comments
- [ ] Constructor injection (no @Autowired fields)
- [ ] Services return Records/DTOs, never raw entities
- [ ] Exception handling centralized in @RestControllerAdvice
- [ ] Transactions managed at service layer (@Transactional)
- [ ] API endpoints versioned (/api/v1/, /api/v2/)
- [ ] HTTP status codes correct (201 for POST, 204 for DELETE, etc.)
- [ ] Input validation via @Valid and custom validators
- [ ] Logging at appropriate levels (info, warn, error)
- [ ] IDs are `String` (MongoDB) — never `Long`
- [ ] Timestamps are `Instant` — never `LocalDateTime`
- [ ] List endpoints return `PageResponse<T>` — never Spring's `Page<T>` directly
- [ ] Error codes match spec/15-ERROR-RESPONSE.md (`VALIDATION_ERROR`, `INTERNAL_ERROR`, etc.)
