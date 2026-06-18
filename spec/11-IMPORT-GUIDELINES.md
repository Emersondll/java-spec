# Import Guidelines - No Wildcards, Explicit Imports Only

## Fundamental Rule

**NEVER use wildcard imports (*). Every artifact must have an explicit import statement.**

---

## Why Avoid Wildcard Imports

### Problems with Wildcard Imports

1. **Ambiguity**: Which class actually used?
   ```java
   import java.util.*;
   import java.sql.*;
   
   List<String> list;  // Which List? java.util.List or sql.List?
   ```

2. **Namespace Pollution**: Loading unused classes
   ```java
   import java.util.*;  // Imports 50+ classes, using only 1
   ```

3. **Future Conflicts**: New imports break code
   ```java
   // Original code with wildcard
   import org.springframework.data.domain.*;
   Page<User> page;  // Works fine
   
   // Later: Spring adds Date class
   import java.util.*;  // Now Date is ambiguous!
   ```

4. **IDE Navigation Issues**: Can't jump to source
   - Click on `List` → IDE doesn't know which List
   - Jump to definition fails

5. **Code Review Difficulty**: Can't identify dependencies
   - Reader doesn't know which classes used
   - Makes impact analysis harder

---

## Correct Import Strategy

### Standard Organization (Java 26 Spring Boot)

```java
// 1. Java standard library (alphabetical)
import java.io.IOException;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

// 2. Jakarta EE (formerly javax.*)
// ─── JPA/Relational projects only (spring-boot-starter-data-jpa in pom.xml) ───
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
// Not needed for MongoDB projects
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

// 3. Spring Framework
// @Autowired is FORBIDDEN — use constructor injection only
// See spec/01-ARCHITECTURE.md — Dependency Injection section
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

// 4. Third-party libraries (alphabetical within each package)
import com.fasterxml.jackson.annotation.JsonIgnore;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// 5. Your company packages (alphabetical)
import com.company.domain.User;
import com.company.dto.UserResponse;
import com.company.exception.UserNotFoundException;
import com.company.repository.UserRepository;
import com.company.service.EmailService;
```

> See also: **01-ARCHITECTURE.md** — constructor injection is mandatory; `@Autowired` field injection is forbidden

### Blank Line Separators

- Blank line between each group
- **DO NOT** use blank lines within groups
- Always maintain order: Java → Jakarta → Spring → Third-party → Company

---

## Complete Examples

### Example 1: Repository Class

```java
/**
 * Repository for User entity database operations.
 * 
 * Notice: Every import is explicit (no wildcards)
 * Every imported class is used in the code below
 * 
 * @see User for entity definition
 * @see UserRepository for CRUD operations
 */

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import com.company.domain.User;
import com.company.domain.UserStatus;

/**
 * Repository interface for User persistence operations.
 * 
 * All imports are used:
 * ✓ List: findByStatusAndCreatedAfter return type
 * ✓ Optional: findByEmail return type
 * ✓ Page, Pageable: findByStatus method
 * ✓ MongoRepository: extends for CRUD
 * ✓ Query: @Query annotation (MongoDB JSON format)
 * ✓ Repository: @Repository annotation
 * ✓ User, UserStatus: method parameters/returns
 */
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    
    Optional<User> findByEmail(String email);
    
    List<User> findByStatusAndCreatedAfter(UserStatus status, Instant createdAt);
    
    Page<User> findByStatus(UserStatus status, Pageable pageable);
    
    boolean existsByEmail(String email);
    
    // MongoDB @Query uses JSON format, not JPQL
    @Query("{ 'status': ?0 }")
    List<User> findRecentByStatus(UserStatus status);
}
```

### Example 2: Service Class

```java
/**
 * Service for user business logic.
 * 
 * Import pattern:
 * - java.* : Objects (null checking)
 * - java.util.* : Optional (null-safe)
 * - jakarta.persistence.* : (none, not in service)
 * - org.springframework.* : Service, Transactional (Spring framework)
 * - org.springframework.data.* : (none)
 * - com.company.* : User, UserResponse, UserRepository, exceptions
 * - lombok.* : @Slf4j for logging
 * - io.micrometer.* : metrics
 */

import java.time.Instant;
import java.util.Objects;
import java.util.Optional;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.company.domain.User;
import com.company.domain.UserStatus;
import com.company.dto.CreateUserRequest;
import com.company.dto.UserResponse;
import com.company.exception.EmailAlreadyExistsException;
import com.company.exception.UserNotFoundException;
import com.company.repository.UserRepository;
import com.company.service.PasswordEncoder;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;

/**
 * Service for user account management.
 * 
 * All imports justified:
 * ✓ Objects: requireNonNull() method
 * ✓ Optional: return type
 * ✓ Service, Transactional: annotations
 * ✓ User, UserStatus: domain model
 * ✓ CreateUserRequest: DTO
 * ✓ UserResponse: DTO
 * ✓ Exceptions: error handling
 * ✓ UserRepository: injected dependency
 * ✓ PasswordEncoder: injected dependency
 * ✓ MeterRegistry: metrics
 * ✓ Slf4j: logging
 */
@Service
@Transactional
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final MeterRegistry meterRegistry;
    
    /**
     * Constructor injection with null checking.
     * 
     * @param userRepository user persistence
     * @param passwordEncoder password hashing
     * @param meterRegistry metrics collection
     */
    public UserService(
        UserRepository userRepository,
        PasswordEncoder passwordEncoder,
        MeterRegistry meterRegistry
    ) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.passwordEncoder = Objects.requireNonNull(passwordEncoder);
        this.meterRegistry = Objects.requireNonNull(meterRegistry);
    }
    
    /**
     * Find user by ID.
     * 
     * @param userId user identifier
     * @return UserResponse with user data
     * @throws UserNotFoundException if not found
     */
    public UserResponse findUserById(String userId) {
        Objects.requireNonNull(userId, "User ID cannot be null");
        
        log.debug("Finding user. userId={}", userId);
        
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        return UserResponse.fromDomain(user);
    }
    
    /**
     * Create new user.
     * 
     * @param request user creation request
     * @return created user response
     * @throws EmailAlreadyExistsException if email exists
     */
    public UserResponse createUser(CreateUserRequest request) {
        Objects.requireNonNull(request, "CreateUserRequest cannot be null");
        
        if (userRepository.existsByEmail(request.email())) {
            throw new EmailAlreadyExistsException(request.email());
        }
        
        // Context-aware Lombok: use @Builder only if project already uses it on entities
        // (see spec/02-CODE-QUALITY.md — Lombok Policy). Otherwise use manual setters:
        User user = new User();
        user.setName(request.name());
        user.setEmail(request.email());
        user.setPassword(passwordEncoder.encode(request.password()));
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(Instant.now());
        
        User saved = userRepository.save(user);
        
        meterRegistry.counter("user.created").increment();
        log.info("User created. userId={}", saved.getId());
        
        return UserResponse.fromDomain(saved);
    }
}
```

### Example 3: REST Controller

```java
/**
 * REST controller for user API endpoints.
 * 
 * All imports explicit and used:
 * ✓ jakarta.validation: validation annotations
 * ✓ org.springframework.http: HTTP utilities
 * ✓ org.springframework.web.bind.annotation: endpoint annotations
 * ✓ java.net.URI: for Location header
 * ✓ com.company.dto: DTOs
 * ✓ com.company.service: business logic
 */

import java.net.URI;

import jakarta.validation.Valid;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

import com.company.dto.CreateUserRequest;
import com.company.dto.UserResponse;
import com.company.service.UserService;
import lombok.extern.slf4j.Slf4j;

/**
 * REST controller for user endpoints.
 * 
 * All imports are used and explicit.
 */
@RestController
@RequestMapping("/api/v1/users")
@Slf4j
public class UserController {
    
    private final UserService userService;
    
    /**
     * Constructor injection.
     * 
     * @param userService user business logic service
     */
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    /**
     * GET user by ID.
     * 
     * @param id user identifier
     * @return user response
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable String id) {
        UserResponse user = userService.findUserById(id);
        return ResponseEntity.ok(user);
    }
    
    /**
     * POST create user.
     * 
     * @param request user creation request
     * @return created user response with 201 status
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
}
```

---

## IDE Configuration

### IntelliJ IDEA

**Settings → Editor → Code Style → Java → Imports**

```
☑ Use single class import
☐ Use wildcard imports
  ☐ for java.* and javax.*
  ☐ for other imports

Packages to Use Imports with '*': (leave empty)

Names to Use Full Qualified Names: (leave empty)
```

### Eclipse

**Window → Preferences → Java → Code Style → Organize Imports**

```
Number of imports needed to use '*': 999 (never)
☑ Do not create imports for types in the same package
```

### VS Code / Other IDEs

Configure to never allow wildcard imports:
```json
{
    "java.import.style": "staticLast",
    "java.import.gradle.wrapper": "on"
}
```

---

## Formatting Rules

### Organization Rules

```
1. java.* (alphabetical)
2. javax.* or jakarta.* (alphabetical)
3. org.* (alphabetical within package)
4. com.* company packages (alphabetical within package)
5. Special: static imports (if used)
```

### Alphabetical Ordering

```
✓ CORRECT
import java.io.File;
import java.io.IOException;
import java.util.List;
import java.util.Map;

✗ INCORRECT
import java.util.List;
import java.io.File;
import java.util.Map;
import java.io.IOException;
```

### One Import Per Line

```
✗ WRONG
import java.util.List, java.util.Map;

✓ CORRECT
import java.util.List;
import java.util.Map;
```

---

## Validation Checklist

Before submitting code:

- [ ] No wildcard imports (*) anywhere
- [ ] Every imported class is used in the code
- [ ] Imports organized in groups (Java → Jakarta → Spring → Company)
- [ ] Imports alphabetical within each group
- [ ] One import per line
- [ ] No unused imports
- [ ] IDE configured to prevent wildcards

### Auto-cleanup (Most IDEs Support)

- IntelliJ: Code → Optimize Imports
- Eclipse: Source → Organize Imports
- VS Code: Java: Organize Imports (command palette)

Run before commit:
```bash
mvn spotless:apply  # Format and organize imports
```

> See also: **20-MAVEN-POM.md** — Checkstyle and Spotless plugin configuration

---

## Common Mistakes

### ❌ Mistake 1: Wildcard Import

```java
// WRONG
import java.util.*;
import java.io.*;
import org.springframework.web.bind.annotation.*;

// CORRECT
import java.util.List;
import java.util.Optional;
import java.io.IOException;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
```

### ❌ Mistake 2: Unused Imports

```java
// WRONG: List imported but never used
import java.util.List;
import java.util.Map;

public class UserResponse {
    private String name;  // Only String used, not List or Map
}

// CORRECT
public class UserResponse {
    private String name;
}
```

### ❌ Mistake 3: Wrong Order

```java
// WRONG: Not grouped properly
import org.springframework.stereotype.Service;
import java.util.Objects;
import jakarta.persistence.Entity;
import com.company.domain.User;

// CORRECT
import java.util.Objects;

import jakarta.persistence.Entity;

import org.springframework.stereotype.Service;

import com.company.domain.User;
```

### ❌ Mistake 4: Ambiguous Imports

```java
// WRONG: Could be java.sql.Date or java.util.Date
import java.sql.*;
import java.util.*;

Date date;  // Which Date?

// CORRECT
import java.time.LocalDate;

LocalDate date;  // Explicit, unambiguous
```

---

## Static Imports (Use Sparingly)

```java
// Static imports for constants and static methods
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;
import static org.assertj.core.api.Assertions.assertThat;

// Use in tests (acceptable)
@Test
void testUserCreation() {
    assertThat(user).isNotNull();  // Clear usage of static import
}

// Avoid in production code
public class UserService {
    public double calculateArea(double radius) {
        return PI * radius * radius;  // Unclear where PI comes from
    }
}
```

---

## Code Review Checklist for Imports

When reviewing code:

- [ ] No wildcard imports used
- [ ] Every import is justified (used somewhere)
- [ ] Imports properly organized
- [ ] Alphabetical order maintained
- [ ] No duplicate imports
- [ ] Static imports used sparingly

> See also: **23-GIT-BRANCHING.md** — commit conventions complement import discipline (Conventional Commits, `mvn verify` before push)

