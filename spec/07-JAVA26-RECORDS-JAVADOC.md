# Java 26 Features, Records & Complete JavaDoc Standards

## Java 26 & Spring Boot Version Requirements

**Mandatory**: All code MUST target **Java 26** and run on **Spring Boot 4.0.7**.

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.7</version>
    <relativePath/>
</parent>

<properties>
    <java.version>26</java.version>
    <maven.compiler.source>26</maven.compiler.source>
    <maven.compiler.target>26</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

> Application configuration MUST be YAML (`application.yml`) — `application.properties` is forbidden (see 00-INDEX §9).

> **Build tool:** This project uses Maven 3.9.9 exclusively. See **20-MAVEN-POM.md** for the complete pom.xml configuration.

## Records - MANDATORY for DTOs/Requests/Responses

**RULE: ALL request/response/DTO objects MUST be Records, NEVER POJOs.**

### Basic Record Structure

```java
/**
 * Record for user creation API request.
 * 
 * Immutable data structure with automatic generation of:
 * - Constructor with all fields
 * - equals(Object)
 * - hashCode()
 * - toString()
 * - Field getters (no "get" prefix, just field name)
 * 
 * Records provide:
 * - Type-safe parameter passing
 * - Automatic null-safety in compact constructor
 * - Serialization/deserialization support
 * 
 * @param name User's full display name (1-100 chars, non-blank)
 * @param email User's email address (unique, valid RFC 5322 format)
 * @param password User's account password (min 8 chars, hashed in service layer)
 * 
 * @since 1.0.0
 * @author Engineering Team
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
     * Compact constructor - validates all parameters before record creation.
     * 
     * Compact constructor benefits:
     * - Executes before implicit field assignment
     * - Can validate parameters and throw exceptions
     * - Prevents null values or invalid states
     * - More concise than regular constructor
     * 
     * Executed implicitly on: new CreateUserRequest(name, email, password)
     * 
     * Validation rules applied:
     * - name: null check, trim whitespace, length 1-100
     * - email: null check, lowercase normalization
     * - password: null check, minimum 8 characters
     * 
     * @throws NullPointerException if any parameter is null
     * @throws IllegalArgumentException if parameter validation fails
     */
    public CreateUserRequest {
        Objects.requireNonNull(name, "Name cannot be null");
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(password, "Password cannot be null");
        
        // Normalize inputs
        name = name.strip();
        email = email.strip().toLowerCase();
        
        // Additional validation
        if (name.isEmpty()) {
            throw new IllegalArgumentException("Name cannot be empty after trimming");
        }
        if (!isValidEmail(email)) {
            throw new IllegalArgumentException("Email format is invalid: " + email);
        }
        if (password.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }
    }
    
    /**
     * Validates email format using regex.
     * 
     * @param email the email to validate
     * @return true if email format is valid
     */
    private static boolean isValidEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$");
    }
}
```

### Response Records

```java
/**
 * Record for user API response.
 * 
 * Contains safe data for external API consumers:
 * - Public fields (id, name, email)
 * - Status information
 * - Timestamps
 * - Excludes: internal references, password data, sensitive fields
 * 
 * Response records are typically created from domain entities
 * via static factory method (see fromDomain).
 * 
 * @param id Unique user identifier — {@code String} for MongoDB (ObjectId); {@code Long} for JPA (auto-generated)
 * @param name User's full display name
 * @param email User's registered email address
 * @param createdAt UTC instant of user creation
 * @param status Current account status (ACTIVE, INACTIVE, PENDING_VERIFICATION)
 * 
 * @since 1.0.0
 * @author Engineering Team
 */
public record UserResponse(
    String id,
    String name,
    String email,
    Instant createdAt,
    UserStatus status
) {
    /**
     * Factory method converting domain User entity to API response record.
     * 
     * Converts:
     * - Domain User entity → Immutable UserResponse record
     * - Filters sensitive fields (password, last_login details)
     * - Ensures consistency in API responses
     * 
     * Use case: Service layer returns records to controller
     * 
     * @param user domain User entity from database
     * @return UserResponse record with safe API data
     * 
     * @throws NullPointerException if user parameter is null
     * @throws AssertionError if user state is invalid
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

> See also: **17-PAGINATION.md** — `PageResponse<T>` and `PageMetadata` are mandatory nested records for all list endpoints.

### Nested Records (for complex structures)

```java
/**
 * Response record for user with related orders.
 * 
 * Demonstrates composition of records:
 * - Parent record includes nested record
 * - Type-safe structure for complex responses
 * - Automatic equality based on all fields (including nested)
 * 
 * @param user user information
 * @param orders list of user's orders (may be empty)
 * @param totalSpent sum of all order amounts
 * 
 * @since 1.0.0
 */
public record UserWithOrdersResponse(
    UserResponse user,
    List<OrderSummary> orders,
    BigDecimal totalSpent
) {
    /**
     * Compact constructor ensuring orders is never null.
     */
    public UserWithOrdersResponse {
        Objects.requireNonNull(user, "User cannot be null");
        Objects.requireNonNull(totalSpent, "TotalSpent cannot be null");
        
        // Convert mutable list to immutable
        orders = List.copyOf(orders != null ? orders : List.of());
    }
}

/**
 * Nested record for order summary data.
 * Part of UserWithOrdersResponse structure.
 * 
 * @param id order identifier — {@code String} for MongoDB; {@code Long} for JPA
 * @param amount order total amount
 * @param status order status (PENDING, COMPLETED, CANCELLED)
 * @param createdAt UTC instant of order creation
 */
public record OrderSummary(
    String id,
    BigDecimal amount,
    OrderStatus status,
    Instant createdAt
) {
    /**
     * Compact constructor for validation.
     */
    public OrderSummary {
        Objects.requireNonNull(id, "Order ID cannot be null");
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(status, "Status cannot be null");
        Objects.requireNonNull(createdAt, "CreatedAt cannot be null");
        
        if (amount.signum() < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }
}
```

## Pattern Matching (Java 26 Feature)

### Pattern Matching with Sealed Classes

```java
/**
 * Sealed interface restricting implementations to specific types.
 * Enables exhaustive pattern matching and prevents unknown implementations.
 * 
 * Sealed interfaces provide:
 * - Restricted inheritance hierarchy
 * - Compiler-checked exhaustive pattern matching
 * - Better documentation of type relationships
 * 
 * @since 1.0.0
 */
// Note: ApiErrorResponse is an illustrative sealed-interface pattern.
// The actual HTTP error envelope is ErrorResponse in common/dto/ — see spec/15-ERROR-RESPONSE.md
public sealed interface ApiResponse permits SuccessResponse, ApiErrorResponse {
    /**
     * Get HTTP status code for response.
     * 
     * @return HTTP status code
     */
    int statusCode();
}

/**
 * Record implementing sealed interface for success responses.
 * 
 * @param data the response payload
 * @param message optional success message
 */
public record SuccessResponse(
    Object data,
    String message
) implements ApiResponse {
    
    @Override
    public int statusCode() {
        return 200;
    }
}

/**
 * Record implementing sealed interface for error responses (illustrative pattern only).
 * 
 * @param code error code identifier
 * @param message error message
 * @param details additional error context
 */
public record ApiErrorResponse(
    String code,
    String message,
    Map<String, String> details
) implements ApiResponse {
    
    @Override
    public int statusCode() {
        return 400;
    }
}

/**
 * Service using pattern matching on sealed types.
 * Compiler ensures all sealed types are handled.
 */
@Service
public class ResponseHandler {
    
    /**
     * Handle API responses using pattern matching.
     * 
     * Pattern matching allows:
     * - Type checking and casting in single expression
     * - Extracting fields directly from matched pattern
     * - Compiler ensures all sealed types handled
     * 
     * @param response the response to handle
     * @return HTTP status code
     */
    public int handleResponse(ApiResponse response) {
        // Pattern matching with type guards and variable binding
        return switch (response) {
            // Match SuccessResponse and bind fields
            case SuccessResponse success when success.data() != null -> {
                log.info("Success response: {}", success.message());
                yield 200;
            }
            
            // Match ErrorResponse and bind fields
            case ApiErrorResponse error -> {
                log.error("Error response: {} - {}", error.code(), error.message());
                yield 400;
            }
            
            // Compiler ensures all sealed types handled
            default -> throw new IllegalArgumentException("Unknown response type");
        };
    }
}
```

> See also: **15-ERROR-RESPONSE.md** — standard HTTP error envelope (`ErrorResponse` record + `GlobalExceptionHandler`).

## Complete JavaDoc Standards

### Class-Level JavaDoc Template

```java
/**
 * [One-line summary of class purpose].
 * 
 * [Detailed description of:
 *  - What the class does
 *  - Key responsibilities
 *  - When to use this class
 *  - Important constraints]
 * 
 * Thread Safety: [Describe thread safety guarantees]
 * - If mutable and unsynchronized: explicitly state not thread-safe
 * - If synchronized: describe synchronization mechanism
 * 
 * Lifecycle: [Describe initialization, usage, cleanup if relevant]
 * 
 * Example Usage:
 * <pre>{@code
 * // Example code block showing typical usage
 * UserService service = new UserService(repository);
 * UserResponse user = service.createUser(request);
 * }</pre>
 * 
 * Dependencies: [List significant dependencies and their roles]
 * - UserRepository: persistence operations
 * - EmailService: email notifications
 * 
 * Performance Characteristics:
 * - Insertion: O(1) average, database dependent
 * - Lookup: O(log n) with index
 * 
 * @author [Author name or team]
 * @version [Version number]
 * @since [Release version when introduced]
 * 
 * @see [Related class 1] for related functionality
 * @see [Related class 2] for related functionality
 */
@Service
@Slf4j
public class UserService {
    // Implementation
}
```

### Method-Level JavaDoc Template

```java
/**
 * [One-line summary of what method does].
 * 
 * [Detailed description including:
 *  - Algorithm/approach used
 *  - Side effects
 *  - Performance implications
 *  - When to use vs alternatives]
 * 
 * Preconditions:
 * - [Condition 1]: [What happens if violated]
 * - [Condition 2]: [What happens if violated]
 * 
 * Postconditions:
 * - [Guaranteed state after execution]
 * - [Data consistency guarantees]
 * 
 * Transaction Behavior:
 * - Transactional: Changes applied atomically
 * - Rollback: On any RuntimeException
 * - Duration: Typically < [expected time]
 * 
 * Performance:
 * - Time Complexity: O(n) where n = [description]
 * - Space Complexity: O(1)
 * - Typical duration: < 200ms p95
 * 
 * Example:
 * <pre>{@code
 * CreateUserRequest request = new CreateUserRequest("John", "john@example.com", "pass");
 * UserResponse response = service.createUser(request);
 * }</pre>
 * 
 * @param paramName [Description of parameter, constraints, valid values]
 * @param userId the user identifier (must be positive, non-null)
 * 
 * @return [Description of return value, null behavior, type]
 * 
 * @throws ExceptionType1 [When/why this exception is thrown]
 * @throws ExceptionType2 [When/why this exception is thrown]
 * @throws DataAccessException if database is unreachable
 * 
 * @implNote [Implementation-specific details for subclasses]
 * 
 * @see [Related method for similar functionality]
 * @since 1.0.0
 */
@Transactional
public UserResponse createUser(CreateUserRequest request) {
    // Implementation
}
```

### Field-Level JavaDoc

```java
/**
 * The user's unique identifier in the system.
 * 
 * Characteristics:
 * - Auto-generated by MongoDB (ObjectId as String)
 * - Immutable after document creation
 * - Primary key for all queries
 * - Used in URLs: /api/v1/users/{id}
 * 
 * Never null after persistence.
 */
@Id
private String id;

/**
 * The user's email address.
 * 
 * Constraints:
 * - Unique index in MongoDB (@Indexed(unique = true))
 * - Valid RFC 5322 email format
 * - Case-insensitive matching
 * - Normalized to lowercase on save
 * 
 * Example: john.doe@example.com
 */
@Indexed(unique = true)
private String email;
```

## JavaDoc for Records

```java
/**
 * [Summary of what this record represents].
 * 
 * [Detailed description of:
 *  - When this record is used
 *  - Data it carries
 *  - Immutability guarantees
 *  - Typical lifecycle]
 * 
 * Immutability: This record is fully immutable. All fields are final
 * and cannot be modified after construction. Suitable for use in
 * Maps, Sets, and as cache keys.
 * 
 * Serialization: This record can be serialized/deserialized via:
 * - Jackson JSON mapper (Spring auto-configuration)
 * - Records serialize field-by-field
 * 
 * Equality: Two records are equal if all fields are equal.
 * Automatically generated equals() and hashCode() based on fields.
 * 
 * @param field1Name [Type and description of first field]
 * @param field2Name [Type and description of second field — use String for MongoDB IDs, Long for JPA IDs]
 * 
 * @author Engineering Team
 * @since 1.0.0
 * 
 * @see [Related class]
 */
public record ExampleRecord(
    String field1Name,
    String field2Name  // String for MongoDB IDs; Long for JPA IDs — detect from pom.xml
) {
    /**
     * Compact constructor ensuring data validity.
     * [Description of validations performed]
     * 
     * @throws NullPointerException if any required field is null
     * @throws IllegalArgumentException if data fails validation
     */
    public ExampleRecord {
        Objects.requireNonNull(field1Name);
        Objects.requireNonNull(field2Name);
    }
}
```

## Refactoring AI Instructions

When refactoring code, AI MUST:

### 1. Convert All DTOs to Records

```java
// BEFORE: POJO class
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private String id;  // String for MongoDB; Long for JPA — detect from pom.xml
    private String name;
    private String email;
    @JsonIgnore
    private String password;
}

// AFTER: Java 26 Record
/**
 * Record for user API response.
 * Immutable, auto-generated equals/hashCode/toString.
 * 
 * @param id user identifier — String for MongoDB; Long for JPA
 * @param name user's full name
 * @param email user's email address
 */
public record UserDTO(
    String id,
    String name,
    String email
) {
    /**
     * Compact constructor for validation.
     */
    public UserDTO {
        Objects.requireNonNull(id);
        Objects.requireNonNull(name);
        Objects.requireNonNull(email);
    }
}
```

### 2. Add Complete JavaDoc to Every Class

```java
// BEFORE: No JavaDoc
public class UserService {
    public UserDTO createUser(CreateUserRequest request) { }
}

// AFTER: Complete JavaDoc
/**
 * Service for managing user accounts and authentication operations.
 * 
 * Responsibilities:
 * - User registration and account creation
 * - User profile management
 * - Account status operations
 * 
 * Thread Safety: This service is stateless and thread-safe.
 * All operations are delegated to thread-safe repositories and services.
 * 
 * @author Engineering Team
 * @version 1.0
 * @since 1.0.0
 * @see UserRepository for database operations
 */
@Service
@Transactional
@Slf4j
public class UserService {
    
    /**
     * Creates a new user account with email verification.
     * 
     * The user record is persisted immediately but remains inactive
     * until email verification completes.
     * 
     * @param request user creation request with name, email, password
     * @return UserDTO with created user details
     * @throws EmailAlreadyExistsException if email already registered
     */
    public UserDTO createUser(CreateUserRequest request) { }
}
```

### 3. Add JavaDoc to All Methods

**Every public and protected method MUST have JavaDoc.**

### 4. Document Records Completely

Each record field MUST be documented explaining:
- Data type and format
- Valid values and constraints
- Null-safety
- Usage context

## JavaDoc Validation Checklist

- [ ] Every class has class-level JavaDoc
- [ ] Every public method has method-level JavaDoc
- [ ] Every record has record-level JavaDoc
- [ ] Every record field has field documentation
- [ ] Every parameter documented with @param
- [ ] Every return value documented with @return
- [ ] Every exception documented with @throws
- [ ] Complex algorithms documented with approach
- [ ] Performance characteristics documented
- [ ] Thread-safety documented (if relevant)
- [ ] No spelling errors in JavaDoc
- [ ] Code examples in {@code} blocks where helpful
- [ ] @since, @author, @version included for classes
- [ ] @see references to related classes

> See also: **20-MAVEN-POM.md** — complete plugin configuration including `maven-javadoc-plugin`, JaCoCo, Surefire and all mandatory plugins.

## Maven Configuration for JavaDoc

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>3.5.0</version>
    <configuration>
        <source>26</source>
        <target>26</target>
        <javadocExecutable>${java.home}/bin/javadoc</javadocExecutable>
        <failOnWarnings>true</failOnWarnings>
        <doclint>all</doclint>
        <reportOutputDirectory>${project.basedir}/javadoc</reportOutputDirectory>
    </configuration>
    <executions>
        <execution>
            <id>attach-javadocs</id>
            <goals>
                <goal>jar</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## Records Best Practices

1. **Use Records for Immutable Data Carriers**
   - Request/Response DTOs
   - Value Objects
   - Data containers

2. **Use Compact Constructor for Validation**
   - Validate parameters before field assignment
   - Throw exceptions for invalid data
   - Normalize input (trim, lowercase)

3. **Provide Static Factory Methods**
   - Convert domain objects to records
   - fromDomain(Entity entity)
   - Create specialized records from generic ones

4. **Never Store Mutable State**
   - Wrap collections in Collections.unmodifiableList()
   - Or use List.copyOf() for immutability
   - Never expose mutable internal state

5. **Document Complete Contracts**
   - Every field documented
   - Validation rules explained
   - Immutability guarantees stated
   - Null-safety behavior clear
