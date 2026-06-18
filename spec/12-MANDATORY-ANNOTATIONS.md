# Mandatory Annotations - Spring Boot Stereotypes

## Golden Rule

**Every class with a specific responsibility MUST have the appropriate Spring stereotype annotation.**

---

## Stereotype Annotations

### @Repository - Data Access Layer

**Purpose**: Mark class as data access object (DAO) layer component.

**Characteristics**:
- Manages database operations
- Extends MongoRepository (MongoDB) or JpaRepository (JPA/Relational)
- Translates exceptions from database layer
- Proxy created by Spring

**When to Use**:
- Data persistence interfaces
- Custom query implementations
- Database operation classes

```java
// MongoDB Example
/**
 * Repository interface for User MongoDB documents.
 * 
 * MANDATORY: @Repository annotation required
 */
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    Optional<User> findByEmail(String email);
}

// JPA (Relational) Example
/**
 * Repository interface for User JPA entities.
 * 
 * MANDATORY: @Repository annotation required
 */
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

/**
 * Custom repository implementation.
 * MANDATORY: @Repository annotation required
 */
@Repository
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    
    private final EntityManager entityManager;
    
    /**
     * Constructor injection.
     * 
     * @param entityManager JPA entity manager
     */
    public UserRepositoryCustomImpl(EntityManager entityManager) {
        this.entityManager = Objects.requireNonNull(entityManager);
    }
    
    @Override
    public List<User> findActiveUsersOlderThan(int ageInDays) {
        // Custom query implementation
        String jpql = "SELECT u FROM User u WHERE u.status = 'ACTIVE' " +
                      "AND u.createdAt < CURRENT_TIMESTAMP - ?1 DAY";
        
        return entityManager.createQuery(jpql, User.class)
            .setParameter(1, ageInDays)
            .getResultList();
    }
}
```

> See also: **18-MONGODB-INDEXES.md** (MongoDB) or **18b-RELATIONAL-INDEXES-MIGRATIONS.md** (JPA) — index and migration conventions for repositories.

### @Service - Business Logic Layer

**Purpose**: Mark class as service/business logic component.

**Characteristics**:
- Encapsulates business logic
- Coordinates repositories
- Manages transactions (@Transactional)
- Implements domain logic

**When to Use**:
- Business logic implementation
- Orchestration of repositories
- Complex computations
- Transaction boundaries

```java
/**
 * Service for user business operations.
 * 
 * MANDATORY: @Service annotation required
 * 
 * Responsibilities:
 * - User creation with validation
 * - User profile management
 * - Complex business rules
 * - Transaction coordination
 * 
 * Thread Safety: Stateless, thread-safe
 * 
 * DO NOT use @Service on:
 * - Controllers (use @RestController)
 * - Repositories (use @Repository)
 * - Configurations (use @Configuration)
 */
@Service
@Transactional
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    
    /**
     * Constructor injection of dependencies.
     * 
     * @param userRepository user persistence layer
     * @param passwordEncoder password encoding utility
     * @param emailService email notification service
     */
    public UserService(
        UserRepository userRepository,
        PasswordEncoder passwordEncoder,
        EmailService emailService
    ) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.passwordEncoder = Objects.requireNonNull(passwordEncoder);
        this.emailService = Objects.requireNonNull(emailService);
    }
    
    /**
     * Create new user with business logic validation.
     * 
     * Business Rules:
     * - Email must be unique
     * - Password must meet security requirements
     * - User status starts as PENDING_VERIFICATION
     * 
     * Transaction:
     * - @Transactional manages persistence
     * - All operations atomic
     * - Rollback on any exception
     * 
     * @param request user creation request
     * @return created user response
     * @throws EmailAlreadyExistsException if email exists
     */
    public UserResponse createUser(CreateUserRequest request) {
        Objects.requireNonNull(request, "Request cannot be null");
        
        log.info("Creating user. email={}", request.email());
        
        // Validation
        validateEmailNotExists(request.email());
        validatePassword(request.password());
        
        // Create entity (Lombok Builder pattern example - used if project context has Lombok)
        User user = User.builder()
            .email(request.email())
            .name(request.name())
            .password(passwordEncoder.encode(request.password()))
            .status(UserStatus.PENDING_VERIFICATION)
            .createdAt(Instant.now())
            .build();

        // Note: If the project context does NOT use Lombok, instantiate manually:
        // User user = new User();
        // user.setEmail(request.email());
        // ...
        
        // Persist (within transaction)
        User saved = userRepository.save(user);
        
        // Send notification (async)
        emailService.sendVerificationEmailAsync(saved.getEmail());
        
        log.info("User created. userId={}", saved.getId());
        
        return UserResponse.fromDomain(saved);
    }
    
    /**
     * Validate email doesn't already exist.
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
     * Validate password strength.
     * 
     * @param password password to validate
     * @throws PasswordInvalidException if doesn't meet requirements
     */
    private void validatePassword(String password) {
        if (password.length() < 8) {
            throw new PasswordInvalidException("Password must be 8+ characters");
        }
    }
}
```

### @RestController - REST Endpoint Layer

**Purpose**: Mark class as REST controller handling HTTP requests.

**Characteristics**:
- Handles HTTP requests/responses
- Maps endpoints to URLs
- Deserializes/serializes DTOs
- Returns JSON (not views)

**When to Use**:
- REST API endpoints
- HTTP request handlers
- JSON response mapping

```java
/**
 * REST controller for user API endpoints.
 * 
 * MANDATORY: @RestController annotation required
 * 
 * Annotation Breakdown:
 * - @RestController = @Controller + @ResponseBody
 * - Returns JSON directly (not view names)
 * - Manages HTTP semantics (status codes, headers)
 * 
 * Responsibilities:
 * - HTTP request handling
 * - Input validation via @Valid
 * - Response status codes
 * - Error mapping to HTTP status
 * 
 * DO NOT put business logic here!
 * Delegate to @Service
 * 
 * DO NOT use @Controller for REST APIs
 * (Use @RestController instead)
 */
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserController {
    
    private final UserService userService;
    
    /**
     * Constructor injection of service.
     * 
     * @param userService business logic service
     */
    public UserController(UserService userService) {
        this.userService = Objects.requireNonNull(userService);
    }
    
    /**
     * POST endpoint to create user.
     * 
     * MANDATORY Annotations:
     * - @PostMapping: HTTP POST method
     * - @RequestBody: Deserialize JSON to DTO
     * - @Valid: Validate input constraints
     * - @ResponseStatus(201): HTTP status code
     * 
     * HTTP Semantics:
     * - Method: POST
     * - URL: /api/v1/users
     * - Request Body: JSON
     * - Response: 201 CREATED with Location header
     * - Response Body: JSON with created user
     * 
     * @param request user creation request record
     * @return ResponseEntity with 201 status and user data
     * @throws EmailAlreadyExistsException mapped to 409 CONFLICT
     * @throws ValidationException mapped to 400 BAD REQUEST
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest request
    ) {
        log.info("POST /api/v1/users - Creating user. email={}", request.email());
        
        UserResponse created = userService.createUser(request);
        
        return ResponseEntity
            .created(URI.create("/api/v1/users/" + created.id()))
            .body(created);
    }
    
    /**
     * GET endpoint to retrieve user by ID.
     * 
     * MANDATORY Annotations:
     * - @GetMapping("/{id}"): HTTP GET with path variable
     * - @PathVariable: Extract ID from URL path
     * 
     * HTTP Semantics:
     * - Method: GET
     * - URL: /api/v1/users/{id}
     * - Response: 200 OK with user JSON or 404 NOT FOUND
     * 
     * @param id user identifier from path
     * @return ResponseEntity with 200 status and user data
     * @throws UserNotFoundException mapped to 404 NOT FOUND
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable String id) {
        log.info("GET /api/v1/users/{} - Retrieving user. id={}", id, id);
        
        UserResponse user = userService.findUserById(id);
        
        return ResponseEntity.ok(user);
    }
}
```

> **Note for REST microservices:** Use only `@RestController`. The `@Controller` annotation is shown below for reference only — it is not used in stateless REST APIs.

### @Controller - View-Based Controller (Spring MVC)

**Purpose**: Mark class as controller returning HTML views.

**Characteristics**:
- Returns view names (not JSON)
- Renders HTML templates (Thymeleaf, JSP)
- Model passed to views
- Browser request handling

**When to Use**:
- Traditional Spring MVC applications
- Server-side rendered pages
- Form submissions

```java
/**
 * Controller for user web interface (HTML views).
 * 
 * MANDATORY: @Controller annotation required (NOT @RestController)
 * 
 * Difference from @RestController:
 * - @Controller: Returns view names, renders HTML
 * - @RestController: Returns JSON directly
 * 
 * Responsibilities:
 * - Handle browser requests
 * - Return HTML views
 * - Pass model data to templates
 * 
 * Use @Controller when:
 * - Building traditional web applications (not APIs)
 * - Using Thymeleaf, JSP templates
 * - Returning HTML responses
 * 
 * Use @RestController when:
 * - Building REST APIs
 * - Returning JSON
 */
@Controller
@RequestMapping("/users")
@Slf4j
public class UserWebController {
    
    private final UserService userService;
    
    /**
     * Constructor injection.
     * 
     * @param userService user service
     */
    public UserWebController(UserService userService) {
        this.userService = Objects.requireNonNull(userService);
    }
    
    /**
     * GET endpoint to display user list page.
     * 
     * MANDATORY Annotations:
     * - @GetMapping: HTTP GET method
     * - Returns view name (Thymeleaf template name)
     * 
     * @param model Spring model for view
     * @return view name to render
     */
    @GetMapping
    public String listUsers(Model model) {
        List<UserResponse> users = userService.findAllUsers();
        model.addAttribute("users", users);
        return "users/list";  // Renders users/list.html
    }
}
```

> See also: **15-ERROR-RESPONSE.md** — `GlobalExceptionHandler` with `@RestControllerAdvice` is the mandatory companion to `@Service` exception throwing.

### @Component - Generic Spring Component

**Purpose**: Mark class as generic Spring-managed component.

**Characteristics**:
- Generic stereotype annotation
- Used when above stereotypes don't fit
- Still dependency-injected
- Singleton by default

**When to Use**:
- Utility components
- Converters, formatters
- Custom listeners
- Configuration helpers
- When no other stereotype fits

```java
/**
 * Component for user DTO conversion.
 * 
 * MANDATORY: @Component annotation required
 * 
 * When to use @Component:
 * - Not a repository (@Repository)
 * - Not a service (@Service)
 * - Not a controller (@RestController/@Controller)
 * - Utility/helper functionality
 * 
 * Alternative: Could be @Bean in @Configuration class
 */
@Component
public class UserDtoConverter {
    
    /**
     * Convert domain User to DTO.
     * 
     * @param user domain entity
     * @return user DTO
     */
    public UserResponse userToResponse(User user) {
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

/**
 * Component for email validation.
 * 
 * MANDATORY: @Component annotation
 */
@Component
public class EmailValidator {
    
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$");
    
    /**
     * Validate email format.
     * 
     * @param email email to validate
     * @return true if valid
     */
    public boolean isValid(String email) {
        return email != null && EMAIL_PATTERN.matcher(email).matches();
    }
}
```

### @Configuration - Configuration Class

**Purpose**: Mark class as source of Spring bean definitions.

**Characteristics**:
- Contains @Bean methods
- Provides configuration
- Defines custom beans
- Not for business logic

**When to Use**:
- Bean definitions
- Configuration setup
- Conditional bean registration
- Complex bean creation

```java
/**
 * Configuration for user module dependencies.
 * 
 * MANDATORY: @Configuration annotation required
 * 
 * Responsibilities:
 * - Define beans not annotated with @Component/@Service/@Repository
 * - Conditional bean registration
 * - Complex bean creation logic
 * - Module-level configuration
 * 
 * DO NOT:
 * - Put business logic here
 * - Use for simple service classes (@Service instead)
 * - Override framework defaults without reason
 */
@Configuration
public class UserModuleConfig {
    
    /**
     * Define password encoder bean.
     * 
     * MANDATORY: @Bean annotation
     * 
     * Returns: Spring-managed bean instance
     * Scope: Singleton (default)
     * Injection: Via constructor or field
     * 
     * @return configured password encoder
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // 12 rounds
    }
    
    /**
     * Define email validator bean.
     * 
     * Note: Could also use @Component instead of @Bean
     * - @Component: Simpler for straightforward classes
     * - @Bean: More control over bean creation
     * 
     * @return email validator instance
     */
    @Bean
    public EmailValidator emailValidator() {
        return new EmailValidator();
    }
}
```

---

## Annotation Checklist by Class Type

### Repository Class
- [ ] @Repository annotation present
- [ ] Extends MongoRepository or JpaRepository or implements custom DAO
- [ ] No business logic (only data access)
- [ ] Uses @Query or Spring Data query methods for custom queries
- [ ] Database-focused methods only

### Service Class
- [ ] @Service annotation present
- [ ] @Transactional on class or methods (essential for JPA relational writes)
- [ ] No HTTP knowledge (no servlet imports)
- [ ] Uses injected repositories
- [ ] Contains business logic
- [ ] Returns DTOs, not entities

### REST Controller
- [ ] @RestController annotation present (not @Controller)
- [ ] @RequestMapping or @PathMapping on class
- [ ] Methods use @GetMapping, @PostMapping, @PutMapping, @DeleteMapping
- [ ] Returns ResponseEntity or POJO
- [ ] Uses @Valid for input validation
- [ ] Sets correct HTTP status codes

### View Controller
- [ ] @Controller annotation present (not @RestController)
- [ ] Returns String (view names)
- [ ] Uses Model to pass data
- [ ] Methods mapped with @GetMapping, @PostMapping
- [ ] Renders HTML templates

### Component/Utility
- [ ] @Component or @Bean annotation present
- [ ] Single responsibility
- [ ] No dependencies on service layer
- [ ] Stateless or immutable

### Configuration
- [ ] @Configuration annotation present
- [ ] Contains @Bean methods
- [ ] Returns bean instances
- [ ] No business logic
- [ ] Module-focused setup

---

## Common Mistakes

### ❌ Mistake 1: Missing @Service on Service Class

```java
// WRONG: No @Service annotation
public class UserService {  // Spring doesn't recognize this!
    @Autowired
    private UserRepository repository;
}

// CORRECT: @Service annotation required
@Service
public class UserService {
    private final UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = Objects.requireNonNull(repository);
    }
}
```

### ❌ Mistake 2: Using @Controller for REST API

```java
// WRONG: @Controller returns view names
@Controller
@RequestMapping("/api/users")
public class UserController {
    @GetMapping
    public String getUser() {
        return "user/list";  // Returns view name, not JSON!
    }
}

// CORRECT: @RestController returns JSON
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping
    public UserResponse getUser() {
        return userService.findUser();  // Returns JSON directly
    }
}
```

### ❌ Mistake 3: Missing @Repository on Repository

```java
// WRONG: No @Repository annotation
// For MongoDB:
public interface UserRepository extends MongoRepository<User, String> { }
// For JPA:
public interface UserRepository extends JpaRepository<User, Long> { }

// CORRECT: @Repository annotation
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    Optional<User> findByEmail(String email);
}
```

### ❌ Mistake 4: @Service on Controller

```java
// WRONG: Mixing responsibilities
@Service
@RestController
public class UserService {
    // Service logic + HTTP endpoints = confusion!
    @PostMapping
    public UserResponse create() { }
}

// CORRECT: Separate responsibilities
@Service
public class UserService {
    public UserResponse create(CreateUserRequest request) { }
}

@RestController
public class UserController {
    private final UserService service;
    
    @PostMapping
    public ResponseEntity<UserResponse> create(
        @RequestBody CreateUserRequest request
    ) {
        return ResponseEntity.ok(service.create(request));
    }
}
```

---

## Stereotype Annotation Comparison

| Annotation | Layer | Purpose | Transaction |
|-----------|-------|---------|-------------|
| @Repository | Data | Database ops | No (optional) |
| @Service | Business | Logic | Yes (@Transactional) |
| @RestController | Presentation | REST API | No |
| @Controller | Presentation | Web views | No |
| @Component | Utility | Generic | No |
| @Configuration | Config | Bean definition | No |

---

## Validation Rules for AI

When refactoring code:

1. **Identify Layer**: Data, Business, or Presentation
2. **Apply Stereotype**: @Repository, @Service, or @RestController
3. **Verify Responsibility**: Only methods for that layer
4. **Check Dependencies**: Inject via constructor
5. **Validate Annotations**: All required annotations present

**Before committing:**
- [ ] Every class with specific role has appropriate annotation
- [ ] No missing @Repository, @Service, @RestController
- [ ] @Transactional on service methods modifying data
- [ ] Controller methods use @GetMapping, @PostMapping, etc.
- [ ] No @Service on controller, no @Controller on service
- [ ] All dependencies injected via constructor
- [ ] Entities instantiated correctly according to project's Lombok context (Builder vs manual setter/constructor)

