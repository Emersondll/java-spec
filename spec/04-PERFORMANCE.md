# Performance Optimization - Java 26 Spring Boot

## Database Performance Optimization

### 1. Multiple Round-Trip Prevention (MongoDB)

MongoDB has no JPA-style N+1, but the same pattern appears as **N separate queries** when iterating over a list and querying references per item.

**Problem Pattern:**
```java
// ✗ WRONG - N separate MongoDB queries
public void processDashboard() {
    List<User> users = userRepository.findAll();        // 1 query
    for (User user : users) {
        List<Contract> contracts = contractRepository   // N queries
            .findByUserId(user.getId());
        calculateTotal(contracts);
    }
}
```

**Solutions:**

```java
/**
 * Solution 1: Embed related data directly in the document (preferred).
 *
 * Use case: Stable summary data rarely updated independently.
 * Trade-off: Document size grows; updates require two writes if denormalized.
 */
@Document(collection = "users")
public class User {
    @Id
    private String id;
    private List<ContractSummary> contractSummaries; // embedded
}

/**
 * Solution 2: Fetch all related IDs in one $in query.
 * Single query replaces N individual lookups.
 *
 * Use case: When embedding is not viable (large or frequently changing data).
 * Trade-off: Full documents loaded; filter in Java if projection needed.
 *
 * @param userIds list of user identifiers
 * @return all contracts belonging to the given users
 */
List<Contract> findByUserIdIn(List<String> userIds);

/**
 * Solution 3: MongoDB Aggregation pipeline for summary data.
 * Equivalent to SQL GROUP BY — executed server-side.
 *
 * @return dashboard DTO with contract count per user
 */
@Aggregation(pipeline = {
    "{ $group: { _id: '$userId', total: { $sum: '$amount' }, count: { $sum: 1 } } }"
})
List<UserDashboardDto> findDashboard();
```

### 2. MongoDB Indexing Strategy

```java
/**
 * Document with optimal index configuration.
 * Indexes accelerate queries but slow down writes.
 * Create indexes for fields used in find(), sort(), and aggregation $match.
 *
 * Index strategy:
 * - Unique index on email (unique constraint + fast lookup)
 * - Compound index on status+createdAt (common filter + sort combination)
 * - No index on name (high cardinality, rarely queried alone)
 *
 * IMPORTANT: auto-index-creation must be false in production.
 * Run index creation scripts during migration (see 18-MONGODB-INDEXES.md).
 */
@Document(collection = "users")
@CompoundIndex(name = "idx_status_created", def = "{'status': 1, 'createdAt': -1}")
public class User {

    @Id
    private String id;

    @Indexed(unique = true)
    private String email;

    private UserStatus status;

    private Instant createdAt;
}
```

### 3. Query Optimization

```java
/**
 * Repository with optimized queries for common access patterns.
 * Each method documented with execution plan rationale.
 */
public interface UserRepository extends MongoRepository<User, String> {

    /**
     * Find user by email with execution details.
     *
     * Execution Plan:
     * - Uses index idx_email for O(log n) lookup
     * - Single document result expected
     * - Response time: < 1ms
     *
     * @param email unique email address
     * @return user matching email
     */
    Optional<User> findByEmail(String email);

    /**
     * Find active users created after a given instant with pagination.
     *
     * Execution Plan:
     * - Uses composite index idx_status_created
     * - Filter: status=ACTIVE AND createdAt > cutoff
     * - Applies LIMIT + SKIP for pagination
     * - Response time: < 50ms for typical page
     *
     * @param status       user status filter
     * @param createdAfter instant threshold for record filtering
     * @param pageable     pagination specification
     * @return paginated list of users
     */
    Page<User> findByStatusAndCreatedAtAfterOrderByCreatedAtDesc(
        UserStatus status,
        Instant createdAfter,
        Pageable pageable
    );

    /**
     * Check email existence without loading the full document.
     *
     * Execution Plan:
     * - Uses index idx_email
     * - Returns boolean, not entity
     * - Response time: < 1ms
     *
     * @param email email to check
     * @return true if email registered, false otherwise
     */
    boolean existsByEmail(String email);
}
```

## Caching Strategy

### Query Caching

```yaml
# application.yml - Cache configuration (Spring Cache + Caffeine in-memory)
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=1h
```

### Application-Level Caching

```java
/**
 * Service with strategic caching for expensive operations.
 * Uses Spring Cache abstraction with Caffeine backend.
 *
 * Caching strategy:
 * - Cache user profiles (frequently accessed, slow aggregation query)
 * - Invalidate on updates via @CacheEvict
 */
@Service
@Slf4j
public class UserService {

    private final UserRepository repository;

    /**
     * @param repository user MongoDB repository
     */
    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    /**
     * Get complete user profile with cached result.
     *
     * First call: queries database.
     * Subsequent calls: served from cache until invalidation.
     *
     * Cache key: userId
     * Cache duration: 1 hour (configured in application.yml)
     *
     * @param userId the user identifier (MongoDB String ID)
     * @return complete user profile data
     * @throws ResourceNotFoundException if user not found
     */
    @Cacheable(value = "userProfiles", key = "#userId", unless = "#result == null")
    public UserProfile getUserProfile(String userId) {
        log.debug("Loading user profile from DB (cache miss). userId={}", userId);
        return repository.findById(userId)
            .map(UserProfile::from)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + userId));
    }

    /**
     * Update user profile and invalidate cache entry.
     *
     * @param userId  user identifier
     * @param request updated profile data
     * @return updated user response
     * @throws ResourceNotFoundException if user not found
     */
    @Transactional
    @CacheEvict(value = "userProfiles", key = "#userId")
    public UserResponse updateProfile(String userId, UpdateProfileRequest request) {
        User user = repository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + userId));
        user.updateProfile(request);
        User updated = repository.save(user);
        log.info("User profile updated and cache invalidated. userId={}", userId);
        return UserResponse.from(updated);
    }
}
```

## MongoDB Connection Pool

```yaml
# application.yml - MongoDB connection pool via URI parameters
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/app?maxPoolSize=20&minPoolSize=5&connectTimeoutMS=30000&serverSelectionTimeoutMS=30000}
      auto-index-creation: false
```

**URI parameter reference:**

| Parameter | Value | Description |
|---|---|---|
| `maxPoolSize` | `20` | Maximum connections in pool |
| `minPoolSize` | `5` | Minimum idle connections |
| `connectTimeoutMS` | `30000` | Timeout to establish a connection |
| `serverSelectionTimeoutMS` | `30000` | Fail fast if no server available |

## Paginated List Endpoints

```java
/**
 * Controller with paginated endpoint following PageResponse standard.
 * NEVER load all documents in memory — every list endpoint MUST paginate.
 *
 * @see spec/17-PAGINATION.md for the full pagination standard
 */
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    /**
     * @param userService user service
     */
    public UserController(UserService userService) {
        this.userService = userService;
    }

    /**
     * GET users with pagination and sorting.
     *
     * Query params: page (default 0), size (default 20, max 100), sort (default createdAt,desc)
     *
     * @param pageable pagination and sort injected by Spring from query params
     * @return 200 with PageResponse envelope
     */
    @GetMapping
    public ResponseEntity<PageResponse<UserResponse>> listUsers(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {
        validatePageSize(pageable.getPageSize());
        Page<UserResponse> page = userService.listUsers(pageable);
        return ResponseEntity.ok(PageResponse.from(page));
    }

    /**
     * Rejects requests with page size above the allowed maximum.
     *
     * @param size requested page size
     * @throws IllegalArgumentException if size exceeds 100
     */
    private void validatePageSize(int size) {
        if (size > 100) {
            throw new IllegalArgumentException("Page size must not exceed 100");
        }
    }
}
```

## JVM Tuning

```bash
#!/bin/bash
# application.sh - JVM launch script with performance tuning
#
# Flag reference:
#   -server                              : enable server JIT optimizations
#   -Xms / -Xmx                         : set initial and max heap (match to avoid resize pauses)
#   -XX:+UseG1GC                         : G1 garbage collector (low latency)
#   -XX:MaxGCPauseMillis                 : target GC pause time in ms
#   -XX:InitiatingHeapOccupancyPercent   : start concurrent GC at this heap % (default 45)
#   -XX:+ParallelRefProcEnabled          : process references in parallel during GC
#   -XX:+ExitOnOutOfMemoryError          : terminate on OOM rather than hang
#   -Dfile.encoding                      : enforce UTF-8 for file I/O

java -server \
    -Xms512m \
    -Xmx2048m \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:InitiatingHeapOccupancyPercent=35 \
    -XX:+ParallelRefProcEnabled \
    -XX:+ExitOnOutOfMemoryError \
    -Dfile.encoding=UTF-8 \
    -jar application.jar
```

## Performance Monitoring Metrics

```java
/**
 * Service with performance metrics collection via Micrometer.
 * Enables real-time monitoring and alerting through Prometheus + Grafana.
 */
@Service
@Slf4j
public class UserMetricsService {

    private final UserRepository repository;
    private final MeterRegistry meterRegistry;
    private final AtomicInteger activeRequests;

    /**
     * Constructor initializing custom metrics gauges.
     *
     * @param repository    user repository
     * @param meterRegistry Micrometer registry (auto-configured by Spring Boot)
     */
    public UserMetricsService(UserRepository repository, MeterRegistry meterRegistry) {
        this.repository = repository;
        this.meterRegistry = meterRegistry;
        this.activeRequests = meterRegistry.gauge(
            "users.active.requests", new AtomicInteger(0));
    }

    /**
     * Find user by ID with latency measurement.
     *
     * Metrics recorded:
     * - user.lookup.duration: timer with p50/p95/p99 percentiles
     * - user.not_found: counter when user is absent
     * - user.lookup.success: counter on successful lookup
     *
     * @param userId MongoDB String user identifier
     * @return user response
     * @throws ResourceNotFoundException if user not found
     */
    public UserResponse findUserById(String userId) {
        activeRequests.incrementAndGet();
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            User user = repository.findById(userId)
                .orElseThrow(() -> {
                    meterRegistry.counter("user.not_found", "userId", userId).increment();
                    return new ResourceNotFoundException("User not found: " + userId);
                });
            meterRegistry.counter("user.lookup.success").increment();
            return UserResponse.from(user);
        } finally {
            sample.stop(Timer.builder("user.lookup.duration")
                .description("Time to lookup user by ID")
                .publishPercentiles(0.5, 0.95, 0.99)
                .publishPercentileHistogram(true)
                .minimumExpectedValue(Duration.ofMillis(10))
                .maximumExpectedValue(Duration.ofSeconds(5))
                .register(meterRegistry));
            activeRequests.decrementAndGet();
        }
    }
}
```

## Performance Checklist

- [ ] All list endpoints paginated using `PageResponse<T>` (no unlimited queries, no raw `Page<T>`)
- [ ] N queries eliminated (use `$in` batch queries or embedding instead of per-item lookups)
- [ ] Database indexes declared for all filter, sort, and unique fields (see 18-MONGODB-INDEXES.md)
- [ ] MongoDB connection pool configured via URI parameters (`maxPoolSize`, `minPoolSize`)
- [ ] Cache strategy defined: what is cached, TTL, and invalidation trigger
- [ ] Query response time < 200ms p95
- [ ] JVM heap sized appropriately (`-Xms` matches `-Xmx` to avoid resize pauses)
- [ ] GC pause target < 200ms (`-XX:MaxGCPauseMillis=200`)
- [ ] Custom Micrometer metrics implemented for critical paths
- [ ] Performance tested with load testing (JMeter, Gatling, or k6)

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
