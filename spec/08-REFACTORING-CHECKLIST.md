# AI Refactoring Checklist - Practical Implementation Guide

## Pre-Refactoring Analysis

Before AI refactors code, perform analysis:

1. **Identify All DTOs/POJOs**
   - Find all classes with @Data, @Getter, @Setter, @NoArgsConstructor
   - These are candidates for conversion to Records
   - Check if they are used as request/response objects

2. **Identify Missing JavaDoc**
   - Scan all public classes without class-level JavaDoc
   - Scan all public methods without method-level JavaDoc
   - Find all fields without documentation

3. **Identify Field Injection**
   - Find @Autowired on fields
   - These MUST be converted to constructor injection
   - Mark as priority refactoring

4. **Identify N+1 Queries**
   - Find methods loading collections in loops
   - Look for eager fetching that could be optimized
   - Check for missing pagination

5. **Identify Code Quality Issues**
   - Methods > 30 lines
   - Classes > 300 lines
   - Methods with > 3 parameters
   - Null pointer risks (no Optional, no null checks)

## Step-by-Step Refactoring Process

### Step 1: Convert POJO to Record

**Pattern to identify (BEFORE):**
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateUserRequest {
    @NotBlank
    private String name;
    
    @Email
    private String email;
    
    @Size(min = 8)
    private String password;
}
```

**Action: Convert to Record (AFTER):**
```java
/**
 * Record for creating a new user via API.
 * Immutable data container with automatic equals/hashCode/toString.
 * 
 * @param name User's full display name (1-100 chars, non-blank)
 * @param email User's email address (unique, valid RFC 5322 format)
 * @param password User's password (minimum 8 characters, hashed in service)
 */
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100)
    String name,
    
    @NotBlank(message = "Email is required")
    @Email
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8)
    String password
) {
    /**
     * Compact constructor ensuring field validity.
     * 
     * @throws NullPointerException if any field is null
     */
    public CreateUserRequest {
        Objects.requireNonNull(name, "Name cannot be null");
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(password, "Password cannot be null");
        
        name = name.strip();
        email = email.strip().toLowerCase();
    }
}
```

**Validation Checklist:**
- [ ] Annotations moved from Lombok to fields
- [ ] Compact constructor added with null checks
- [ ] Complete JavaDoc added with field descriptions
- [ ] Validation logic in compact constructor
- [ ] Factory methods added if needed (from/of)

### Step 2: Add Complete JavaDoc to Class

**Action: Add class-level JavaDoc**

```java
// BEFORE - No JavaDoc
@Service
public class UserService {
}

// AFTER - Complete JavaDoc
/**
 * Service for user account management and authentication.
 * 
 * Responsibilities:
 * - User registration and creation
 * - Profile updates
 * - Account status management
 * - Email verification workflows
 * 
 * Thread Safety: This service is stateless and thread-safe.
 * All state is managed by underlying repositories and services.
 * 
 * Dependencies are injected via constructor ensuring:
 * - Immutability
 * - Testability via mocking
 * - Failure transparency
 * 
 * @author Engineering Team
 * @version 1.0
 * @since 1.0.0
 * @see UserRepository for persistence operations
 * @see EmailService for email notifications
 */
@Service
@Transactional
@Slf4j
public class UserService {
}
```

**Validation Checklist:**
- [ ] One-line summary at top
- [ ] Detailed description of responsibilities
- [ ] Thread-safety documented
- [ ] Dependencies listed with @see
- [ ] @author, @version, @since included
- [ ] No spelling errors

### Step 3: Convert Field Injection to Constructor Injection

**Action: Replace @Autowired fields with constructor**

```java
// BEFORE - Field injection (anti-pattern)
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
    
    @Autowired
    private EmailService emailService;
}

// AFTER - Constructor injection (best practice)
/**
 * Service for user operations with explicit dependency injection.
 */
@Service
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    /**
     * Constructor-based dependency injection.
     * Spring automatically wires beans in required order.
     * All dependencies are final ensuring immutability.
     * 
     * @param userRepository repository for user persistence operations
     * @param emailService service for email notifications
     */
    public UserService(
        UserRepository userRepository,
        EmailService emailService
    ) {
        this.userRepository = Objects.requireNonNull(userRepository, "UserRepository cannot be null");
        this.emailService = Objects.requireNonNull(emailService, "EmailService cannot be null");
    }
}
```

**Validation Checklist:**
- [ ] All fields marked as private final
- [ ] Constructor parameter validation with Objects.requireNonNull()
- [ ] Constructor JavaDoc added
- [ ] All @Autowired annotations removed
- [ ] No setter methods
- [ ] Fields assigned in constructor, not initialized

### Step 4: Add Method-Level JavaDoc

**Action: Document every public method**

```java
// BEFORE - No JavaDoc
public UserResponse createUser(CreateUserRequest request) {
    User user = new User();
    user.setName(request.getName());
    user.setEmail(request.getEmail());
    user.setPassword(passwordEncoder.encode(request.getPassword()));
    User saved = userRepository.save(user);
    emailService.sendVerificationEmail(user.getEmail());
    return UserResponse.from(saved);
}

// AFTER - Complete JavaDoc
/**
 * Creates a new user account with email verification.
 * 
 * The user record is persisted immediately but remains in PENDING_VERIFICATION
 * status until the user completes email verification via sent link.
 * 
 * Verification email is sent asynchronously to avoid blocking HTTP response.
 * If email sending fails, the user account still exists but cannot log in
 * until email is verified.
 * 
 * Transaction Behavior:
 * - User record persisted within transaction
 * - Email sending is asynchronous (outside transaction)
 * - If persistence fails, exception thrown and transaction rolled back
 * - If email sending fails, user already persisted (idempotent retry safe)
 * 
 * Performance:
 * - Time Complexity: O(1) for persistence, email queue submission O(1)
 * - Typical Duration: < 200ms p95 (excludes email server latency)
 * - Database: INSERT into users table
 * 
 * @param request the user creation request record
 *        Must be non-null and pass @Valid validation
 *        Contains name (1-100 chars), email (unique), password (8+ chars)
 * 
 * @return UserResponse containing created user details
 *         Status will be PENDING_VERIFICATION
 *         ID is system-generated (non-null)
 * 
 * @throws EmailAlreadyExistsException if email already registered in system
 *         HTTP Status: 409 Conflict
 *         Message: "Email [email] already registered"
 * 
 * @throws PasswordInvalidException if password doesn't meet security requirements
 *         HTTP Status: 400 Bad Request
 *         Requirements: 8+ chars, 1 uppercase, 1 digit, 1 special char
 * 
 * @throws DataAccessException if database operation fails
 *         HTTP Status: 500 Internal Server Error
 *         Common causes: database unavailable, constraint violation
 * 
 * @see PasswordValidator#validate for password strength rules
 * @see EmailService#sendVerificationEmail for email verification details
 * 
 * @implNote Email sending is asynchronous via @Async annotation.
 *           Check logs for email delivery issues separately from user creation.
 */
@Transactional
public UserResponse createUser(CreateUserRequest request) {
    Objects.requireNonNull(request, "CreateUserRequest cannot be null");
    
    log.info("Creating user. email={}", request.email());
    
    // Implementation...
}
```

**Validation Checklist:**
- [ ] One-line summary at top
- [ ] Detailed description of what method does
- [ ] All parameters documented with @param
- [ ] Return value documented with @return
- [ ] All checked exceptions documented with @throws
- [ ] Performance characteristics documented
- [ ] Example usage (if complex)
- [ ] Related methods linked with @see
- [ ] No spelling errors

### Step 5: Ensure All Requests/Responses are Records

**Pattern to identify:**

```java
// BEFORE - Any response class not a Record
public class UserDTO {
    private Long id;
    private String name;
    private String email;
    
    // getters, setters, equals, hashCode, toString...
}

// AFTER - Convert to Record
/**
 * Record for user API response containing public user data.
 * 
 * Contains safe data for external API consumers:
 * - id: user identifier
 * - name: user's full name
 * - email: user's registered email
 * - createdAt: ISO 8601 creation timestamp
 * - status: current account status
 * 
 * Excludes sensitive fields:
 * - password (never exposed)
 * - last_login (privacy concern)
 * - internal_references (API contract)
 * 
 * @param id unique user identifier (system-generated, immutable, String for MongoDB)
 * @param name user's full display name
 * @param email user's email address
 * @param createdAt UTC instant of user creation
 * @param status current account status (ACTIVE, INACTIVE, PENDING_VERIFICATION)
 */
public record UserResponse(
    String id,
    String name,
    String email,
    Instant createdAt,
    UserStatus status
) {
    /**
     * Factory method converting domain User entity to API response.
     * 
     * @param user domain User entity from database
     * @return UserResponse record with safe API data
     * @throws NullPointerException if user is null
     */
    public static UserResponse fromDomain(User user) {
        Objects.requireNonNull(user, "User entity cannot be null");
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

**Validation Checklist:**
- [ ] Request record has @Valid annotations on fields
- [ ] Compact constructor validates constraints
- [ ] Response record immutable (all fields non-mutable)
- [ ] Response record has fromDomain(Entity) factory method
- [ ] Complete JavaDoc for record and fields
- [ ] No getter/setter methods (records generate field accessors)
- [ ] List endpoints return `PageResponse<T>` — never Spring's `Page<T>` directly (see **17-PAGINATION.md**)

### Step 6: Fix Method Size (>30 lines)

**Action: Extract helper methods from large methods**

```java
// BEFORE - Method > 30 lines
public UserResponse createUser(CreateUserRequest request) {
    // Validate email
    if (request.email() == null || request.email().isBlank()) {
        throw new IllegalArgumentException("Email required");
    }
    if (userRepository.existsByEmail(request.email())) {
        throw new EmailAlreadyExistsException(request.email());
    }
    
    // Validate password
    if (request.password() == null || request.password().length() < 8) {
        throw new IllegalArgumentException("Password min 8 chars");
    }
    if (!containsUppercase(request.password())) {
        throw new IllegalArgumentException("Password needs uppercase");
    }
    if (!containsDigit(request.password())) {
        throw new IllegalArgumentException("Password needs digit");
    }
    
    // Create and persist
    User user = new User(request.name(), request.email());
    user.setPassword(passwordEncoder.encode(request.password()));
    User saved = userRepository.save(user);
    
    // Send email
    emailService.sendVerificationEmail(saved.getEmail());
    
    // Return response
    return UserResponse.fromDomain(saved);
}

// AFTER - Extract helper methods, method < 30 lines
/**
 * Creates a new user account with email verification.
 * 
 * @param request user creation request
 * @return created user response
 */
@Transactional
public UserResponse createUser(CreateUserRequest request) {
    Objects.requireNonNull(request, "Request cannot be null");
    
    validateEmailNotExists(request.email());
    validatePasswordStrength(request.password());
    
    User user = persistUser(request);
    emailService.sendVerificationEmail(user.getEmail());
    
    return UserResponse.fromDomain(user);
}

/**
 * Validates email is not already registered.
 * 
 * @param email email to validate
 * @throws EmailAlreadyExistsException if email exists
 */
private void validateEmailNotExists(String email) {
    if (userRepository.existsByEmail(email)) {
        throw new EmailAlreadyExistsException(email);
    }
}

/**
 * Validates password meets security requirements.
 * 
 * @param password password to validate
 * @throws PasswordInvalidException if requirements not met
 */
private void validatePasswordStrength(String password) {
    if (password == null || password.length() < 8) {
        throw new PasswordInvalidException("Password must be 8+ characters");
    }
    if (!password.matches(".*[A-Z].*")) {
        throw new PasswordInvalidException("Password needs uppercase letter");
    }
    if (!password.matches(".*[0-9].*")) {
        throw new PasswordInvalidException("Password needs digit");
    }
}

/**
 * Persists new user to database.
 * 
 * @param request user creation request
 * @return persisted user entity
 */
private User persistUser(CreateUserRequest request) {
    // Use manual constructor (no Lombok @Builder unless the project already uses it on entities)
    User user = new User();
    user.setName(request.name());
    user.setEmail(request.email());
    user.setPassword(passwordEncoder.encode(request.password()));
    user.setStatus(UserStatus.PENDING_VERIFICATION);
    user.setCreatedAt(Instant.now());
    
    return userRepository.save(user);
}
```

**Validation Checklist:**
- [ ] Each method < 30 lines
- [ ] Each helper method has single responsibility
- [ ] Helper method JavaDoc added
- [ ] Method names describe what they do
- [ ] Private scope for helpers
- [ ] No code duplication across methods

### Step 7: Validate HTTP Status Codes

> **Error envelope standard:** See **15-ERROR-RESPONSE.md** for the `ErrorResponse` record structure and internal codes:
> `VALIDATION_ERROR` · `RESOURCE_NOT_FOUND` · `CONFLICT` · `INTERNAL_ERROR`
> Never use `VALIDATION_FAILED` or `INTERNAL_SERVER_ERROR`.

**Check all REST endpoints return correct status:**

```java
// BEFORE - Missing status codes
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @PostMapping
    public UserResponse create(@RequestBody CreateUserRequest request) {
        return service.createUser(request);
    }
    
    @GetMapping("/{id}")
    public UserResponse get(@PathVariable String id) {
        return service.findUserById(id);
    }
    
    @DeleteMapping("/{id}")
    public void delete(@PathVariable String id) {
        service.deleteUser(id);
    }
}

// AFTER - Correct HTTP status codes
/**
 * REST controller for user-related HTTP endpoints.
 * Handles request/response mapping and HTTP protocol semantics.
 */
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserController {
    
    /**
     * POST endpoint to create user.
     * Returns 201 CREATED with Location header for created resource.
     * 
     * @param request user creation request
     * @return ResponseEntity with 201 status and user response
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserResponse> create(
        @Valid @RequestBody CreateUserRequest request
    ) {
        log.info("Creating user. email={}", request.email());
        UserResponse created = service.createUser(request);
        
        return ResponseEntity
            .created(URI.create("/api/v1/users/" + created.id()))
            .body(created);
    }
    
    /**
     * GET endpoint to retrieve user.
     * Returns 200 OK if found, 404 NOT FOUND if missing.
     * 
     * @param id user identifier
     * @return ResponseEntity with 200 status and user response
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> get(@PathVariable String id) {
        log.debug("Getting user. id={}", id);
        return ResponseEntity.ok(service.findUserById(id));
    }
    
    /**
     * DELETE endpoint to remove user.
     * Returns 204 NO CONTENT on successful deletion.
     * 
     * @param id user identifier
     * @return ResponseEntity with 204 status
     */
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public ResponseEntity<Void> delete(@PathVariable String id) {
        log.info("Deleting user. id={}", id);
        service.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

**HTTP Status Codes Reference:**
- **200 OK**: GET successful, default for retrieval
- **201 CREATED**: POST successful, include Location header
- **204 NO CONTENT**: DELETE/PUT successful, no body needed
- **400 BAD REQUEST**: Validation failed, invalid parameters
- **401 UNAUTHORIZED**: Not authenticated
- **403 FORBIDDEN**: Authenticated but no permission
- **404 NOT FOUND**: Resource doesn't exist
- **409 CONFLICT**: Resource already exists (email, unique constraint)
- **500 INTERNAL SERVER ERROR**: Unexpected error
- **503 SERVICE UNAVAILABLE**: Database down, external service unavailable

**Validation Checklist:**
- [ ] POST returns 201 CREATED with Location header
- [ ] GET returns 200 OK
- [ ] DELETE returns 204 NO CONTENT
- [ ] Not found returns 404 NOT FOUND
- [ ] Validation error returns 400 BAD REQUEST
- [ ] Duplicate email returns 409 CONFLICT
- [ ] @ResponseStatus used for non-200 responses

## Quality Metrics to Verify

After refactoring, verify:

```bash
# Run SonarQube analysis
mvn clean verify sonar:sonar

# Check test coverage
mvn jacoco:report

# Verify no Checkstyle violations
mvn checkstyle:check

# Verify no SpotBugs issues
mvn spotbugs:check

# Generate JavaDoc
mvn javadoc:javadoc
```

**Acceptance Criteria:**
- [ ] Zero SonarQube critical issues
- [ ] Test coverage > 80%
- [ ] Zero Checkstyle violations
- [ ] Zero critical SpotBugs issues
- [ ] JavaDoc generation succeeds (no warnings)
- [ ] All tests pass (mvn test)
- [ ] Application starts successfully
- [ ] API endpoints respond correctly

## Refactoring Priority Order

1. **High Priority (Breaking Changes, Security):**
   - Convert all DTO POJOs to Records
   - Fix field injection → constructor injection
   - Add missing null checks (prevent NPE)
   - Fix N+1 query problems

2. **Medium Priority (Quality):**
   - Add complete JavaDoc to all classes/methods
   - Break methods > 30 lines
   - Reduce class > 300 lines
   - Fix HTTP status codes

3. **Low Priority (Optimization):**
   - Add custom metrics
   - Optimize queries
   - Cache optimization
   - Performance profiling

## Automated Refactoring Checklist

Before deploying refactored code:

- [ ] All DTOs converted to Records
- [ ] All classes have complete JavaDoc
- [ ] All methods have complete JavaDoc  
- [ ] All public fields documented
- [ ] No field injection (@Autowired on fields)
- [ ] All methods < 30 lines
- [ ] All classes < 300 lines
- [ ] HTTP status codes correct
- [ ] Lombok policy applied correctly (context-aware: project uses Lombok → @Builder; does not → manual)
- [ ] Entity ID type matches database (MongoDB → String, JPA → Long)
- [ ] All timestamps are Instant (never LocalDateTime)
- [ ] HTTP status codes correct (201 for POST, 204 for DELETE)
- [ ] No System.out.println() in code
- [ ] No TODO/FIXME without issue ticket reference
- [ ] Repeated string/numeric literals extracted to private static final constants
- [ ] SonarQube violations fixed
- [ ] SpotBugs critical issues fixed
- [ ] Application starts without errors
- [ ] All API tests pass

### Branch & PR Compliance (MANDATORY)

- [ ] Branch follows naming convention `feat/*`, `fix/*`, `refactor/*` etc. — see **23-GIT-BRANCHING.md**
- [ ] PR opened to `main` with description filled (what / why / how)
- [ ] CI green (`mvn verify`) before requesting review
- [ ] No direct commits to `main`

### OpenAPI Documentation (MANDATORY)

- [ ] `openapi.yaml` at `src/main/resources/static/openapi.yaml`
- [ ] springdoc configured: `swagger-ui.url: /openapi.yaml` + `api-docs.enabled: false`
- [ ] **Zero Swagger annotations in code** — `@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition` are FORBIDDEN
- [ ] All endpoints documented in the YAML file
- [ ] All schemas include description and examples
- [ ] Swagger UI accessible at `/swagger-ui.html`

> **Rule:** Documentation lives in the file — never in the code. See `13-OPENAPI-REST-REFERENCE.md`.
