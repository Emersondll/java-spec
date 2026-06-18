# Code Quality Standards - Java 26 & Spring Boot

## JavaDoc Requirements - MANDATORY

**Every class, interface, method, and public field MUST have complete JavaDoc.**

### Class-Level JavaDoc

```java
/**
 * Service for managing user accounts and authentication.
 * 
 * Responsibilities:
 * - User registration and account creation
 * - User profile updates
 * - Account status management
 * 
 * Thread Safety: This service is thread-safe. All operations are stateless
 * and delegated to thread-safe repositories.
 * 
 * Transaction Management: Methods marked with @Transactional maintain ACID
 * guarantees. Rollback occurs on any RuntimeException.
 * 
 * Dependencies are injected via constructor to ensure immutability
 * and facilitate unit testing via mocking.
 * 
 * @author Engineering Team
 * @version 1.0
 * @since 1.0.0
 * @see UserRepository for database operations
 * @see EmailService for email notifications
 */
@Service
@Transactional
@Slf4j
public class UserService {
    // Implementation
}
```

### Method-Level JavaDoc

```java
/**
 * Creates a new user account with email verification.
 * 
 * The user record is persisted immediately, but account remains inactive
 * until email verification is completed. A verification email is sent
 * asynchronously to avoid blocking the HTTP response.
 * 
 * Transaction Behavior:
 * - User record is persisted within transaction
 * - Email sending is asynchronous and outside transaction
 * - Rollback occurs if user persistence fails
 * 
 * @param request the user creation request containing name, email, password
 *        Must be non-null and pass @Valid validation
 * 
 * @return UserResponse containing created user ID, email, and status
 *         Will have status=PENDING_VERIFICATION
 * 
 * @throws EmailAlreadyExistsException if email already registered
 *         Status: 409 Conflict
 * 
 * @throws PasswordInvalidException if password doesn't meet security requirements
 *         Status: 400 Bad Request. Minimum 8 chars, 1 uppercase, 1 digit, 1 special char
 * 
 * @throws DataAccessException if database operation fails
 *         Status: 500 Internal Server Error
 * 
 * @see EmailService#sendVerificationEmail for email verification details
 * @see PasswordValidator#validate for password requirements
 * 
 * Complexity: O(1) for user record save, O(1) for email queue submission
 * Performance SLA: < 200ms p95
 */
@Transactional
public UserResponse createUser(CreateUserRequest request) {
    Objects.requireNonNull(request, "CreateUserRequest cannot be null");
    
    // Implementation
}
```

### Record JavaDoc

```java
/**
 * Request record for creating a new user account.
 * 
 * All fields are mandatory and validated by @NotBlank/@Email constraints.
 * Compact constructor enforces null-safety.
 * 
 * This immutable record is automatically generated with equals/hashCode/toString
 * based on field values. Perfect for use in Maps/Sets.
 * 
 * Field Constraints:
 * - name: 1-100 characters, non-blank, no leading/trailing whitespace
 * - email: Must be valid RFC 5322 email format, unique across system
 * - password: Minimum 8 characters, must contain uppercase, digit, special char
 * 
 * @param name The user's full display name. Used in emails and UI.
 *             Example: "John Doe"
 * @param email The user's email address. Used for login and notifications.
 *              Example: "john.doe@example.com"
 * @param password The user's account password. Hashed immediately in service layer
 *                 using BCrypt algorithm. Never logged or exposed.
 *                 Example: "SecureP@ss123" (will be hashed before storage)
 * 
 * @since 1.0.0
 * @author Engineering Team
 * 
 * @see CreateUserRequest#CreateUserRequest compact constructor for validation
 * @see UserService#createUser for business logic
 */
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100, message = "Name must be 1-100 characters")
    String name,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid RFC 5322 format")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {
    /**
     * Compact constructor - validates all parameters before record instantiation.
     * 
     * This constructor is called implicitly when creating records via new.
     * Provides null-safety and precondition validation.
     * 
     * Validation Rules:
     * - name: Cannot be null, trimmed to remove leading/trailing whitespace
     * - email: Cannot be null, lowercased for case-insensitive storage
     * - password: Cannot be null, length >= 8
     * 
     * @throws NullPointerException if any field is null
     * @throws IllegalArgumentException if any field fails validation
     */
    public CreateUserRequest {
        Objects.requireNonNull(name, "Name cannot be null");
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(password, "Password cannot be null");
        
        // Trim whitespace
        name = name.strip();
        email = email.strip().toLowerCase();
    }
}
```

## Code Quality Checklist

### Method-Level Standards

```java
// ✓ CORRECT - Single Responsibility, <30 lines
/**
 * Validates user email format and uniqueness.
 * 
 * @param email the email to validate
 * @return true if email is valid and available
 * @throws InvalidEmailException if format is invalid
 * @throws EmailAlreadyExistsException if email already registered
 */
private boolean validateEmail(String email) {
    if (!EMAIL_PATTERN.matcher(email).matches()) {
        throw new InvalidEmailException("Email format invalid: " + email);
    }
    if (repository.existsByEmail(email)) {
        throw new EmailAlreadyExistsException("Email already registered: " + email);
    }
    return true;
}

// ✗ INCORRECT - Multiple responsibilities, >50 lines
public void processUserAndSendEmail(CreateUserRequest request) {
    // Validation
    if (request.name() == null) throw new Exception();
    // Persistence
    User user = new User();
    // Email
    // Cache
    // ... massive method
}
```

### Class-Level Standards

- **Maximum 300 lines per class**
- **Single Responsibility Principle (SRP)**
- **Cohesion: All methods use most fields**

```java
// ✓ Well-organized
/**
 * Service managing user creation and updates.
 */
@Service
public class UserService {
    // CRUD and user-specific operations only
}

// ✗ Too many responsibilities
@Service
public class UserService {
    // CRUD Users
    // Send emails
    // Generate reports
    // Process payments
    // Integrate with third-party APIs
    // ... violates SRP
}
```

### Method Parameters - Maximum 3

```java
// ✓ CORRECT - DTO encapsulates multiple values
/**
 * Creates user from request record.
 */
public UserResponse createUser(CreateUserRequest request) { }

// ✗ INCORRECT - Too many parameters
public UserResponse createUser(
    String name,
    String email,
    String phone,
    LocalDate birthDate,
    String address,
    String city,
    String postalCode,
    String country
) { }
```

## Null-Safety Standards

### Use Optional in Public APIs

```java
/**
 * Finds user by email address.
 * 
 * @param email the email to search for
 * @return Optional containing user if found, empty otherwise.
 *         Never returns null
 * 
 * @throws IllegalArgumentException if email is blank
 * @throws DataAccessException if database operation fails
 */
public Optional<User> findByEmail(String email) {
    if (email == null || email.isBlank()) {
        throw new IllegalArgumentException("Email cannot be blank");
    }
    return repository.findByEmail(email.toLowerCase());
}

// Usage in service layer
/**
 * Retrieves user response by email with error handling.
 */
public UserResponse findUserByEmail(String email) {
    return findByEmail(email)
        .map(UserResponse::fromDomain)
        .orElseThrow(() -> new UserNotFoundException("Email not found: " + email));
}
```

### Null-Safety with Objects.requireNonNull

```java
/**
 * Validates email format.
 * 
 * @param email the email to validate (non-null)
 * @return true if email format is valid
 * @throws NullPointerException if email is null
 */
public boolean isValidEmail(String email) {
    Objects.requireNonNull(email, "email must not be null");
    return EMAIL_PATTERN.matcher(email).matches();
}
```

## Immutability Standards

### Use Records for Value Objects

```java
// ✓ CORRECT - Immutable record
/**
 * Represents a user's profile information.
 * Immutable by design - records generate final fields and no setters.
 */
public record UserProfile(
    String displayName,
    String bio,
    LocalDate birthDate
) {
    /**
     * Compact constructor ensuring data validity.
     */
    public UserProfile {
        Objects.requireNonNull(displayName, "Display name cannot be null");
        if (displayName.isBlank()) {
            throw new IllegalArgumentException("Display name cannot be blank");
        }
    }
}

// ✗ INCORRECT - Mutable class with setters
public class UserProfile {
    private String displayName;  // Can be changed = potential bugs
    
    public void setDisplayName(String displayName) { }  // Mutation
}
```

### Collections Must Be Unmodifiable

```java
/**
 * Retrieves all user roles.
 * 
 * @param userId the user identifier (MongoDB String ID)
 * @return unmodifiable list of roles. Modifications throw UnsupportedOperationException
 * @throws UserNotFoundException if user not found
 */
public List<Role> getUserRoles(String userId) {
    User user = findUserById(userId);
    return Collections.unmodifiableList(user.getRoles());
}

// Or use Java 9+ List.copyOf()
return List.copyOf(user.getRoles());
```

## Timestamp Standard — Use `Instant`

All timestamps MUST use `java.time.Instant` (UTC, timezone-safe). Never use `LocalDateTime` for persistence or API fields.

```java
// ✓ CORRECT
private Instant createdAt;
private Instant updatedAt;

// ✗ INCORRECT — LocalDateTime has no timezone information
private LocalDateTime createdAt;
```

## Constants Standard — Extract Repeated Literals (MUST)

String and numeric literals that appear more than once MUST be extracted to `private static final` constants. This avoids repetition, reduces typo risk, and improves performance.

### String Constants

```java
// ✗ INCORRECT — repeated string literals
public UserResponse findByEmail(String email) {
    log.info("Finding user by email. email={}", email);
    return repository.findByEmail(email)
        .map(UserResponse::fromDomain)
        .orElseThrow(() -> new ResourceNotFoundException("User not found with email: " + email));
}

public void deleteByEmail(String email) {
    log.info("Deleting user by email. email={}", email);
    if (!repository.existsByEmail(email)) {
        throw new ResourceNotFoundException("User not found with email: " + email);
    }
    repository.deleteByEmail(email);
}

// ✓ CORRECT — constants for repeated strings
private static final String USER_NOT_FOUND_BY_EMAIL = "User not found with email: ";
private static final String LOG_EMAIL_PARAM = "email={}";

public UserResponse findByEmail(String email) {
    log.info("Finding user by email. " + LOG_EMAIL_PARAM, email);
    return repository.findByEmail(email)
        .map(UserResponse::fromDomain)
        .orElseThrow(() -> new ResourceNotFoundException(USER_NOT_FOUND_BY_EMAIL + email));
}

public void deleteByEmail(String email) {
    log.info("Deleting user by email. " + LOG_EMAIL_PARAM, email);
    if (!repository.existsByEmail(email)) {
        throw new ResourceNotFoundException(USER_NOT_FOUND_BY_EMAIL + email);
    }
    repository.deleteByEmail(email);
}
```

### Numeric Constants

```java
// ✗ INCORRECT — magic numbers
if (password.length() < 8) { throw new IllegalArgumentException("..."); }
if (page.getSize() > 100) { throw new IllegalArgumentException("..."); }

// ✓ CORRECT — named constants
private static final int MIN_PASSWORD_LENGTH = 8;
private static final int MAX_PAGE_SIZE = 100;

if (password.length() < MIN_PASSWORD_LENGTH) { throw new IllegalArgumentException("..."); }
if (page.getSize() > MAX_PAGE_SIZE) { throw new IllegalArgumentException("..."); }
```

### Constant Placement Rules

| Scope | Location | Example |
|---|---|---|
| Used in a single class | `private static final` in that class | Error messages, log params |
| Shared across same package | Package-level utility class (`XxxConstants`) | Query field names, cache keys |
| Shared across modules | `common/constants/` package | HTTP header names, event types |

### What MUST Be a Constant

- Exception messages repeated > 1×
- Log message fragments repeated > 1×
- Field names used in queries or JSON keys
- Cache key prefixes
- Event type strings (e.g., `"order.created"`)
- Magic numbers (thresholds, limits, timeouts)
- HTTP header names
- Configuration property names used in `@Value`

### What Does NOT Need a Constant

- Strings used exactly once (inline is acceptable)
- Standard framework values (`"application/json"`, `"/api/v1/"`) unless repeated
- Test-only literals

## Comment and Documentation Standards

### What TO Comment

```java
/**
 * Complex algorithm explanation.
 * Exponential backoff retry strategy to prevent cascade failures during high load.
 * Formula: sleep_duration = 2^attempt seconds, max 60 seconds
 * 
 * Example: attempt 1 = 2s, attempt 2 = 4s, attempt 3 = 8s, ... attempt 6+ = 60s
 */
private void retryWithExponentialBackoff() {
    long sleep = Math.min((long) Math.pow(2, attempt), 60) * 1000;
    Thread.sleep(sleep);
}

/**
 * Non-obvious business rule or domain decision.
 * We filter by created_at instead of updated_at because:
 * - Batch processing job runs at 2AM daily
 * - User modifications after 2AM should process next day
 * - Avoids double-processing on rapid updates
 */
List<User> findNewActiveUsers(Instant since);

/**
 * Workaround for known library bug.
 * Spring Data before version X generates N+1 queries for certain aggregations.
 * This custom query works around that until the upgrade.
 * 
 * TODO: Remove after Spring Data upgrade — track in issue #1234
 */
```

### What NOT TO Comment

```java
// ✗ Obvious - Don't comment obvious code
// Increment counter
counter++;

// ✗ Outdated - Remove instead of commenting
// User was loaded from cache
// Cache is now updated daily so this is unnecessary
User user = repository.findById(id);

// ✓ Remove bad code entirely
// Don't leave dead code commented out
```

## Static Analysis Configuration (Maven)

### SonarQube — `pom.xml`

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.11.0.3922</version>
</plugin>
```

Run with:
```bash
mvn sonar:sonar \
  -Dsonar.projectKey=user-service \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=${SONAR_TOKEN}
```

### SpotBugs — `pom.xml`

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <excludeFilterFile>spotbugs-exclude.xml</excludeFilterFile>
    </configuration>
    <executions>
        <execution>
            <id>spotbugs-check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

### Checkstyle — `pom.xml`

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
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

### Checkstyle Configuration — `checkstyle.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
    <!-- Size violations -->
    <module name="LineLength">
        <property name="max" value="120"/>
        <property name="ignorePattern" value="^package.*|^import.*|a href|href|http://|https://"/>
    </module>
    
    <!-- JavaDoc requirements -->
    <module name="TreeWalker">
        <module name="JavadocMethod">
            <property name="scope" value="public"/>
        </module>
        <module name="JavadocType">
            <property name="scope" value="public"/>
        </module>
    </module>
</module>
```

## Lombok Policy — Context-Aware (MUST)

Lombok usage on entity/object classes is **determined by the existing project context**.
`@Slf4j` for logging is always allowed and does NOT count as "the project uses Lombok."

### Detection Rule

Before refactoring, the AI MUST scan the project's entity/model/domain classes:

1. Search for `@Data`, `@Builder`, `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor` on **entity/object classes** (not on records, not `@Slf4j` alone)
2. If **≥1 entity class** uses any of these → **project uses Lombok** → follow the Lombok pattern
3. If **zero entity classes** use them → **project does NOT use Lombok** → manual getters/setters

### Project Uses Lombok → Follow Builder Pattern

```java
// ✓ CORRECT — project already uses Lombok on entities
/**
 * User domain entity for persistence.
 *
 * @author Engineering Team
 * @since 1.0.0
 */
@Document(collection = "users")   // MongoDB
// @Entity @Table(name = "users") // JPA (relational)
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private static final String EMAIL_REQUIRED = "Email cannot be null";

    @Id
    private String id;             // MongoDB: String
    // private Long id;            // JPA: Long

    private String name;

    @Indexed(unique = true)        // MongoDB
    // @Column(unique = true)      // JPA
    private String email;

    private String password;
    private UserStatus status;
    private Instant createdAt;
    private Instant updatedAt;
}
```

### Project Does NOT Use Lombok → Manual Getters/Setters

```java
// ✓ CORRECT — project does NOT use Lombok on entities
/**
 * User domain entity for persistence.
 *
 * @author Engineering Team
 * @since 1.0.0
 */
@Document(collection = "users")
public class User {

    @Id
    private String id;
    private String name;
    private String email;
    private Instant createdAt;

    /** Default no-arg constructor for framework use. */
    public User() { }

    /**
     * @return the user's unique identifier
     */
    public String getId() { return id; }

    /**
     * @param id the user's unique identifier
     */
    public void setId(String id) { this.id = id; }

    // ... remaining getters/setters with JavaDoc
}
```

### Summary Table

| Detection Result | Allowed on Entities | Preferred Pattern |
|---|---|---|
| Project uses Lombok | `@Getter`, `@Setter`, `@Builder`, `@Data`, `@NoArgsConstructor`, `@AllArgsConstructor` | `@Builder` + `@Getter` + `@NoArgsConstructor` + `@AllArgsConstructor` |
| Project does NOT use Lombok | Only `@Slf4j` (on services, not entities) | Manual getters/setters with JavaDoc |
| DTOs / Requests / Responses | **Always** `record` — Lombok is irrelevant | Java 26 Record with compact constructor |

## Database — MongoDB or Relational (JPA)

The spec supports **both** database paradigms. Detect from `pom.xml`:
- `spring-boot-starter-data-mongodb` → MongoDB project
- `spring-boot-starter-data-jpa` → Relational project (PostgreSQL, MySQL)

### MongoDB Entity

```java
/**
 * User document stored in MongoDB.
 *
 * @author Engineering Team
 * @since 1.0.0
 */
@Document(collection = "users")
public class User {

    @Id
    private String id;    // MongoDB uses String IDs (ObjectId)

    @Indexed(unique = true)
    private String email;

    private Instant createdAt;
}
```

### JPA / Relational Entity

```java
/**
 * User entity stored in a relational database (PostgreSQL/MySQL).
 *
 * @author Engineering Team
 * @since 1.0.0
 */
@Entity
@Table(name = "users")
public class User {

    private static final String SEQUENCE_NAME = "users_id_seq";

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;     // Relational uses Long IDs (auto-increment)

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = Instant.now();
    }
}
```

### Repository Pattern Comparison

```java
// MongoDB
public interface UserRepository extends MongoRepository<User, String> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}

// JPA (Relational)
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

### ID Type Rule

| Database | ID Type | Annotation |
|---|---|---|
| MongoDB | `String` | `@Id` (Spring Data MongoDB) |
| PostgreSQL/MySQL | `Long` | `@Id` + `@GeneratedValue` (JPA) |

## Refactoring Instructions for AI

When an AI refactors code, it MUST:

1. **Convert all DTOs to Java 26 Records**
   - Replace POJOs with record syntax
   - Add compact constructors for validation
   - Keep @Valid annotations on fields

2. **Add complete JavaDoc**
   - Class-level: Purpose, responsibilities, thread-safety
   - Method-level: What it does, parameters, return value, exceptions, performance
   - Record-level: What the record represents, field meanings, constraints

3. **Ensure Constructor Injection**
   - Replace @Autowired field injection with constructor injection
   - Use Objects.requireNonNull() for null-safety checks
   - Mark fields as private final

4. **Validate Null-Safety**
   - Remove null pointer exceptions by using Optional or validation
   - Use Objects.requireNonNull() at method entry
   - Never return null, use Optional or throw exception

5. **Enforce Method Size < 30 Lines**
   - Extract helper methods
   - Use method references for stream operations
   - Avoid nested loops and complex conditions

6. **Validate API Responses**
   - All @RestController methods return Record types
   - Use correct HTTP status codes
   - Include error handling with @ExceptionHandler

7. **Verify Code Quality Metrics**
   - No violations of Checkstyle rules
   - No high-severity SpotBugs issues
   - SonarQube code coverage > 80%

8. **Use correct ID type per database**
   - MongoDB: entity `@Id` fields must be `String`, repository extends `MongoRepository<T, String>`
   - JPA: entity `@Id` fields must be `Long`, repository extends `JpaRepository<T, Long>`

9. **Use Instant for all timestamps**
   - Never use `LocalDateTime` for persistence or API fields
   - Use `Instant` throughout for UTC timezone safety

10. **Apply Lombok policy based on project context**
    - Scan existing entity classes to detect Lombok usage
    - If project uses Lombok → apply `@Builder` + `@Getter` + `@NoArgsConstructor` + `@AllArgsConstructor`
    - If project does NOT use Lombok → manual getters/setters only
    - `@Slf4j` alone does NOT mean "project uses Lombok"

11. **Extract constants from repeated literals**
    - String literals used > 1× → `private static final String`
    - Numeric literals (magic numbers) → `private static final int/long`
    - Place constants at the top of the class, before fields

## Code Review Checklist for AI

Before completing refactoring:

- [ ] All classes have complete JavaDoc
- [ ] All methods have complete JavaDoc
- [ ] All public records have field JavaDoc
- [ ] No @Autowired field injection (use constructor)
- [ ] All requests/responses are Records
- [ ] Methods have max 30 lines
- [ ] Methods have max 3 parameters
- [ ] No null pointer exceptions (use Optional or Objects.requireNonNull)
- [ ] Lombok policy applied correctly (context-aware: project uses Lombok → @Builder; does not → manual)
- [ ] Entity ID type matches database (MongoDB → String, JPA → Long)
- [ ] All timestamps are Instant (never LocalDateTime)
- [ ] HTTP status codes correct (201 for POST, 204 for DELETE)
- [ ] No System.out.println() in code
- [ ] No TODO/FIXME without issue ticket reference
- [ ] Repeated string/numeric literals extracted to constants
- [ ] SonarQube violations fixed
- [ ] SpotBugs critical issues fixed
