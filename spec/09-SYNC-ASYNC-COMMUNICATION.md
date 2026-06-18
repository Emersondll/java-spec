# Synchronous & Asynchronous Communication - Best Practices

> **See also:**
> - **16-RABBITMQ-SAGA.md** — complete RabbitMQ topology, DLQ, exchange declarations, and idempotency
> - **10-CIRCUIT-BREAKER.md** — circuit breaker and fallback patterns for external HTTP calls
> - **24-CICD-SECRETS.md** — secrets management via environment variables, Vault, and Kubernetes Secrets

## Decision Matrix: When to Use Synchronous vs Asynchronous

| Scenario | Synchronous | Asynchronous | Rationale |
|----------|-------------|--------------|-----------|
| **API endpoint needing external data** | ✅ Preferred | ❌ Avoid | Client expects immediate response with data |
| **Background job/scheduled task** | ❌ Avoid | ✅ Preferred | No waiting client, async execution optimal |
| **Email notifications** | ❌ Avoid | ✅ Preferred | Email server latency, user doesn't need to wait |
| **Payment processing** | ✅ Preferred | ❌ Dangerous | Financial transaction needs immediate confirmation |
| **User action triggering cascade** | ❌ Questionable | ✅ Preferred | Complex workflows benefit from event-driven |
| **Report generation (immediate)** | ✅ Preferred | ❌ Avoid | User waits for PDF download |
| **Report generation (scheduled)** | ❌ Avoid | ✅ Preferred | Runs nightly, email result to user |
| **Database query** | ✅ Always | ❌ Never | Database operations are synchronous |
| **Audit logging** | ❌ Avoid | ✅ Preferred | Non-critical, fire-and-forget acceptable |
| **Cache warming** | ❌ Questionable | ✅ Preferred | Async startup logic improves boot time |

---

## Synchronous Communication - Best Practices

### When to Use Synchronous

```
Requirements for Synchronous:
✅ Caller MUST wait for response
✅ Response data needed immediately
✅ Short operation duration (< 5 seconds)
✅ Critical business operation (payments, orders)
✅ Direct request-response pattern
✅ Simple error handling
```

### Synchronous REST Endpoints

```java
/**
 * REST controller for synchronous user operations.
 * 
 * Synchronous Characteristics:
 * - Request blocks until response ready
 * - HTTP connection held open during operation
 * - Response payload sent to client
 * - Timeout: 30 seconds (nginx/load balancer default)
 * - Suitable for: lookups, validations, fast operations
 * 
 * Thread Usage: 1 thread per request (Tomcat thread pool)
 * Resource Cost: Medium (connection held)
 * Scalability: Limited by thread pool size
 */
@RestController
@RequestMapping("/api/v1/users")
@Validated
@Slf4j
public class UserSyncController {
    
    private final UserService userService;
    private final MeterRegistry meterRegistry;
    
    /**
     * Constructor injection of service dependencies.
     * 
     * @param userService user business logic service
     * @param meterRegistry metrics registry for monitoring
     */
    public UserSyncController(
        UserService userService,
        MeterRegistry meterRegistry
    ) {
        this.userService = Objects.requireNonNull(userService);
        this.meterRegistry = Objects.requireNonNull(meterRegistry);
    }
    
    /**
     * GET endpoint retrieving user by ID synchronously.
     * 
     * Synchronous Pattern:
     * 1. Client sends HTTP GET request
     * 2. Server thread acquires from pool
     * 3. Service executes database query
     * 4. Response sent immediately to client
     * 5. Thread returned to pool
     * 
     * Performance SLA:
     * - Target: < 50ms p95 (database cached)
     * - Timeout: 30 seconds (client-side)
     * - Thread hold time: < 100ms typical
     * 
     * @param id the user identifier to retrieve
     * @return ResponseEntity with user data
     * @throws UserNotFoundException if user not found (404)
     * @throws DataAccessException if database error (500)
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable String id) {
        log.debug("Synchronous GET user by ID. id={}", id);
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            UserResponse user = userService.findUserById(id);
            return ResponseEntity.ok(user);
            
        } finally {
            sample.stop(Timer.builder("user.get.duration")
                .description("User GET endpoint duration")
                .register(meterRegistry));
        }
    }
    
    /**
     * POST endpoint creating user synchronously.
     * 
     * Synchronous Pattern for Creation:
     * 1. Client POSTs create request
     * 2. Server validates input
     * 3. Server persists to database
     * 4. Server returns 201 CREATED with Location header
     * 5. Client receives response with new resource ID
     * 
     * Performance SLA:
     * - Target: < 200ms p95
     * - Timeout: 30 seconds
     * - Database time dominates
     * 
     * When to Use:
     * - User expects immediate confirmation
     * - Need to return generated ID
     * - Resource must exist immediately
     * - Part of critical business flow
     * 
     * @param request user creation request record
     * @return ResponseEntity with 201 status and created user
     * @throws EmailAlreadyExistsException if email exists (409)
     * @throws ValidationException if validation fails (400)
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserResponse> createUserSync(
        @Valid @RequestBody CreateUserRequest request
    ) {
        log.info("Synchronous POST create user. email={}", request.email());
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            UserResponse created = userService.createUser(request);
            
            // Blocking email sending (NOT RECOMMENDED)
            // sendVerificationEmailBlocking(created.email());
            
            // Better: Schedule async email (see async section)
            scheduleVerificationEmailAsync(created.email());
            
            return ResponseEntity
                .created(URI.create("/api/v1/users/" + created.id()))
                .body(created);
            
        } finally {
            sample.stop(Timer.builder("user.create.duration")
                .register(meterRegistry));
        }
    }
    
    /**
     * Delegate async email sending (don't block response).
     * 
     * @param email user email for verification
     */
    private void scheduleVerificationEmailAsync(String email) {
        // Implementation in async section below
    }
}
```

### Synchronous Service Layer

```java
/**
 * Service implementing synchronous operations with proper timeout handling.
 * 
 * Synchronous Service Characteristics:
 * - Direct return of business logic results
 * - All dependencies respond synchronously
 * - Transactional consistency guaranteed
 * - No async/event mechanisms
 * - Blocking database calls
 * 
 * Thread Safety: Stateless, thread-safe
 * Transactions: Managed with @Transactional
 * Timeouts: Database connection timeout only
 */
@Service
@Transactional
@Slf4j
public class UserSyncService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    /**
     * Constructor injection of dependencies.
     * 
     * @param userRepository user persistence repository
     * @param passwordEncoder password encoding service
     */
    public UserSyncService(
        UserRepository userRepository,
        PasswordEncoder passwordEncoder
    ) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.passwordEncoder = Objects.requireNonNull(passwordEncoder);
    }
    
    /**
     * Find user by ID synchronously.
     * 
     * Execution Flow:
     * 1. Validate input (userId not null)
     * 2. Query database (SELECT * FROM users WHERE id=?)
     * 3. If found: Return Optional with user
     * 4. If not found: Return empty Optional
     * 5. Caller handles empty (throws exception or returns 404)
     * 
     * Database Performance:
     * - Uses index on primary key (id)
     * - Expected: < 1ms with cache, < 10ms without
     * 
     * @param userId the user identifier
     * @return UserResponse with complete user data
     * @throws UserNotFoundException if user not found
     * @throws DataAccessException if database error
     */
    public UserResponse findUserById(String userId) {
        Objects.requireNonNull(userId, "User ID cannot be null");
        
        log.debug("Finding user synchronously. userId={}", userId);
        
        // Database call blocks until result returned
        User user = userRepository.findById(userId)
            .orElseThrow(() -> {
                log.warn("User not found. userId={}", userId);
                return new UserNotFoundException(userId);
            });
        
        return UserResponse.fromDomain(user);
    }
    
    /**
     * Create user synchronously with complete validation.
     * 
     * Execution Flow:
     * 1. Validate email doesn't exist
     * 2. Encode password using BCrypt
     * 3. Create User entity
     * 4. Persist to database (INSERT)
     * 5. Return response with generated ID
     * 
     * Transaction Behavior:
     * - All operations in single transaction
     * - Atomicity: All succeed or all rollback
     * - Consistency: Email unique constraint enforced
     * - Duration: Depends on database write speed
     * 
     * Performance SLA:
     * - Target: < 200ms p95
     * - Latency dominated by: INSERT operation
     * - Scalability: Limited by database write throughput
     * 
     * @param request user creation request record
     * @return UserResponse with created user details
     * @throws EmailAlreadyExistsException if email exists (409)
     * @throws DataAccessException if database fails (500)
     */
    public UserResponse createUser(CreateUserRequest request) {
        Objects.requireNonNull(request, "CreateUserRequest cannot be null");
        
        log.info("Creating user synchronously. email={}", request.email());
        
        // Check email doesn't exist (may block on database lock)
        if (userRepository.existsByEmail(request.email())) {
            log.warn("Email already exists. email={}", request.email());
            throw new EmailAlreadyExistsException(request.email());
        }
        
        // Encode password (CPU-intensive, ~100ms with bcrypt)
        String hashedPassword = passwordEncoder.encode(request.password());
        
        // Create and persist user (INSERT blocks until success)
        // Use manual setters or Lombok @Builder only if project already uses it on entities
        // (see spec/02-CODE-QUALITY.md — Lombok context-aware policy)
        User user = new User();
        user.setName(request.name());
        user.setEmail(request.email());
        user.setPassword(hashedPassword);
        user.setStatus(UserStatus.PENDING_VERIFICATION);
        user.setCreatedAt(Instant.now());
        
        User saved = userRepository.save(user);
        
        log.info("User created synchronously. userId={}", saved.getId());
        
        return UserResponse.fromDomain(saved);
    }
}
```

### Synchronous HTTP Client

```java
/**
 * Service making synchronous HTTP calls to external APIs.
 * 
 * Synchronous HTTP Characteristics:
 * - Thread blocks until response received
 * - Simple error handling
 * - Automatic retry on timeout
 * - Limited scalability
 * 
 * Use Case: Payment gateway, shipping APIs, internal services
 * Risk: If external service slow, entire request chain slow
 *
 * > For circuit breaker and fallback on external calls: see **10-CIRCUIT-BREAKER.md**
 */
@Service
@Slf4j
public class PaymentSyncClient {
    
    private final RestTemplate restTemplate;
    private final String paymentApiKey;
    private static final int TIMEOUT_SECONDS = 5;
    
    /**
     * Constructor with RestTemplate and API key injection.
     * Secrets injected via @Value — never use System.getenv().
     * See spec/24-CICD-SECRETS.md for secrets management.
     * 
     * @param restTemplate  configured HTTP client
     * @param paymentApiKey payment gateway API key from environment variable
     */
    public PaymentSyncClient(RestTemplate restTemplate,
                             @Value("${PAYMENT_API_KEY}") String paymentApiKey) {
        this.restTemplate = Objects.requireNonNull(restTemplate);
        this.paymentApiKey = Objects.requireNonNull(paymentApiKey, "PAYMENT_API_KEY must be set");
    }
    
    /**
     * Process payment synchronously against external API.
     * 
     * Synchronous HTTP Pattern:
     * 1. Create HTTP request
     * 2. Send to payment gateway
     * 3. BLOCK waiting for response (timeout = 5 seconds)
     * 4. Receive response with payment status
     * 5. Return result to caller
     * 
     * Thread Blocking:
     * - HTTP connection held open for duration
     * - Tomcat thread blocked (cannot serve other requests)
     * - If payment gateway slow: entire application slow
     * 
     * Timeout Strategy:
     * - Socket read timeout: 5 seconds
     * - If exceeded: SocketTimeoutException thrown
     * - Caller decides: retry, fail, or use fallback
     * 
     * @param orderId the order to process payment for
     * @param amount payment amount in cents
     * @return payment confirmation record
     * @throws HttpClientErrorException if payment fails (4xx)
     * @throws HttpServerErrorException if gateway error (5xx)
     * @throws ResourceAccessException if timeout (connection timeout)
     */
    public PaymentConfirmationRecord processPaymentSync(
        String orderId,
        BigDecimal amount
    ) {
        Objects.requireNonNull(orderId, "Order ID cannot be null");
        Objects.requireNonNull(amount, "Amount cannot be null");
        
        log.info("Processing payment synchronously. orderId={}, amount={}", orderId, amount);
        
        try {
            // Build HTTP request
            HttpEntity<PaymentRequestRecord> request = new HttpEntity<>(
                new PaymentRequestRecord(orderId, amount),
                createHeaders()
            );
            
            // Send synchronous HTTP request (BLOCKS until response)
            ResponseEntity<PaymentResponseRecord> response = restTemplate.exchange(
                "https://payment-gateway.com/api/v1/process",
                HttpMethod.POST,
                request,
                PaymentResponseRecord.class
            );
            
            log.info("Payment processed. status={}", response.getBody().status());
            
            return new PaymentConfirmationRecord(
                response.getStatusCode().value(),
                response.getBody().status(),
                response.getBody().transactionId()
            );
            
        } catch (ResourceAccessException e) {
            // Timeout or connection error
            log.error("Payment timeout. orderId={}", orderId, e);
            throw new PaymentTimeoutException("Payment gateway timeout", e);
            
        } catch (HttpClientErrorException e) {
            // 4xx error (invalid request)
            log.error("Payment validation error. status={}", e.getStatusCode(), e);
            throw new PaymentValidationException(e.getMessage(), e);
            
        } catch (HttpServerErrorException e) {
            // 5xx error (gateway error)
            log.error("Payment gateway error. status={}", e.getStatusCode(), e);
            throw new PaymentGatewayException(e.getMessage(), e);
        }
    }
    
    /**
     * Creates HTTP headers for API authentication.
     * 
     * @return HttpHeaders with authorization
     */
    private HttpHeaders createHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(paymentApiKey);
        return headers;
    }
}

/**
 * Record for payment processing request.
 * Note: illustrative example for sync HTTP pattern.
 * In a real project, define shared records once in the domain module.
 * See also: spec/10-CIRCUIT-BREAKER.md for the circuit breaker context.
 *
 * @param orderId the order identifier (String — MongoDB ID)
 * @param amount payment amount in cents
 */
public record PaymentRequestRecord(String orderId, BigDecimal amount) {
    /**
     * Compact constructor for validation.
     */
    public PaymentRequestRecord {
        Objects.requireNonNull(orderId, "Order ID cannot be null");
        Objects.requireNonNull(amount, "Amount cannot be null");
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
}

/**
 * Record for payment processing response.
 * 
 * @param status payment status (SUCCESS, FAILED, PENDING)
 * @param transactionId payment gateway transaction ID
 */
public record PaymentResponseRecord(String status, String transactionId) {
    /**
     * Compact constructor ensuring valid state.
     */
    public PaymentResponseRecord {
        Objects.requireNonNull(status, "Status cannot be null");
        Objects.requireNonNull(transactionId, "Transaction ID cannot be null");
    }
}
```

---

## Asynchronous Communication - Best Practices

### When to Use Asynchronous

```
Requirements for Asynchronous:
✅ Caller does NOT need immediate response
✅ Long-running operation (> 5 seconds)
✅ Non-critical business operation (notifications, reports)
✅ Event-driven architecture preferred
✅ Better resource utilization needed
✅ Complex workflows/sagas
```

### Spring @Async Pattern

```java
/**
 * Configuration for asynchronous task execution.
 * 
 * Asynchronous Characteristics:
 * - Non-blocking execution
 * - Separate thread pool
 * - Caller continues immediately
 * - Fire-and-forget or future-based
 * - Resource efficient (many async < many sync threads)
 */
@Configuration
@EnableAsync
@Slf4j
public class AsyncConfig {
    
    /**
     * Configure thread pool for async operations.
     * 
     * Thread Pool Strategy:
     * - Core: 10 threads always available
     * - Max: 50 threads under load
     * - Queue: 100 tasks buffered before rejection
     * - Timeout: Threads shutdown after 1 hour idle
     * 
     * Sizing Rationale:
     * - For 4-core CPU: 10-20 threads optimal
     * - Scale: 1 thread per core minimum
     * - Email operations: typically < 1 second per task
     * - Database operations: < 100ms per task
     * 
     * @return TaskExecutor for async method execution
     */
    @Bean(name = "asyncExecutor")
    public TaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);                    // Always active threads
        executor.setMaxPoolSize(50);                     // Maximum threads
        executor.setQueueCapacity(100);                  // Task queue buffer
        executor.setThreadNamePrefix("async-executor-"); // Thread naming
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        return executor;
    }
    
    /**
     * Alternative executor for email operations.
     * Separate pool prevents one type blocking others.
     * 
     * @return TaskExecutor for email tasks
     */
    @Bean(name = "emailExecutor")
    public TaskExecutor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("email-executor-");
        executor.initialize();
        return executor;
    }
}
```

### Email Service Asynchronously

```java
/**
 * Email service implementing asynchronous notifications.
 * 
 * Asynchronous Email Pattern:
 * - User action triggers async email task
 * - Email sent in background (separate thread)
 * - User gets response immediately
 * - Email delivery is non-critical
 * - If email fails: logged but doesn't affect user experience
 * 
 * Thread Usage: Minimal (1 thread per email)
 * Resource Cost: Low (thread pool shared)
 * Scalability: High (100s of async emails)
 */
@Service
@Slf4j
public class EmailAsyncService {
    
    private final JavaMailSender mailSender;
    private final MeterRegistry meterRegistry;
    
    /**
     * Constructor injection of dependencies.
     * 
     * @param mailSender Spring mail sender
     * @param meterRegistry metrics registry
     */
    public EmailAsyncService(JavaMailSender mailSender, MeterRegistry meterRegistry) {
        this.mailSender = Objects.requireNonNull(mailSender);
        this.meterRegistry = Objects.requireNonNull(meterRegistry);
    }
    
    /**
     * Send verification email asynchronously.
     * 
     * Asynchronous Execution:
     * - Method returns immediately (before email sent)
     * - Email sending happens in background thread
     * - No exception thrown to caller if email fails
     * - Failures logged for monitoring
     * 
     * Return Value: CompletableFuture<Void>
     * - Caller can wait if desired: future.get() (but usually doesn't)
     * - Allows caller to chain operations: future.thenRun(...)
     * - Non-blocking: Caller continues immediately
     * 
     * Thread: Executed in asyncExecutor thread pool (not request thread)
     * 
     * Error Handling:
     * - Caught and logged
     * - Metrics incremented
     * - Email retry not implemented (consider external queue for retry)
     * 
     * @param email recipient email address
     * @param verificationToken unique verification token
     * @return CompletableFuture completing when email sent
     *         Note: Caller usually ignores return value
     * 
     * @see #sendVerificationEmailAsync(String, String) for async execution
     */
    @Async("emailExecutor")
    public CompletableFuture<Void> sendVerificationEmailAsync(
        String email,
        String verificationToken
    ) {
        Objects.requireNonNull(email, "Email cannot be null");
        Objects.requireNonNull(verificationToken, "Token cannot be null");
        
        try {
            log.info("Sending verification email asynchronously. email={}", email);
            
            // Build email message
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(email);
            message.setSubject("Verify Your Email Address");
            message.setText("Click here to verify: https://app.com/verify?token=" + verificationToken);
            
            // Send email (may block for 1-5 seconds)
            mailSender.send(message);
            
            log.info("Verification email sent successfully. email={}", email);
            meterRegistry.counter("email.verification.sent").increment();
            
            return CompletableFuture.completedFuture(null);
            
        } catch (MailException e) {
            log.error("Failed to send verification email. email={}", email, e);
            meterRegistry.counter("email.verification.failed").increment();
            
            // Don't throw exception - email delivery is non-critical
            // In production: implement retry mechanism (message queue)
            return CompletableFuture.failedFuture(e);
        }
    }
    
    /**
     * Send order confirmation email asynchronously.
     * 
     * Async Pattern for Business Notifications:
     * - Order persisted and confirmed to user immediately
     * - Confirmation email sent in background
     * - If email fails: order still exists
     * - Retry mechanism recommended (not implemented here)
     * 
     * @param email customer email
     * @param orderId order identifier
     * @return CompletableFuture (usually ignored by caller)
     */
    @Async("emailExecutor")
    public CompletableFuture<Void> sendOrderConfirmationEmailAsync(
        String email,
        String orderId
    ) {
        try {
            log.info("Sending order confirmation email. orderId={}, email={}", orderId, email);
            
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(email);
            message.setSubject("Order Confirmation #" + orderId);
            message.setText("Thank you for your order!");
            
            mailSender.send(message);
            
            meterRegistry.counter("email.confirmation.sent").increment();
            return CompletableFuture.completedFuture(null);
            
        } catch (MailException e) {
            log.error("Failed to send confirmation email. orderId={}", orderId, e);
            meterRegistry.counter("email.confirmation.failed").increment();
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### Async Service with CompletableFuture

```java
/**
 * Service with async operations returning CompletableFuture.
 * 
 * CompletableFuture Pattern:
 * - Non-blocking future representation
 * - Caller can chain operations
 * - Combine multiple async operations
 * - Handle completion or exceptions
 */
@Service
@Slf4j
public class ReportAsyncService {
    
    private final ReportRepository reportRepository;
    private final EmailAsyncService emailService;
    
    /**
     * Constructor injection.
     * 
     * @param reportRepository report persistence
     * @param emailService async email service
     */
    public ReportAsyncService(
        ReportRepository reportRepository,
        EmailAsyncService emailService
    ) {
        this.reportRepository = Objects.requireNonNull(reportRepository);
        this.emailService = Objects.requireNonNull(emailService);
    }
    
    /**
     * Generate report asynchronously.
     * 
     * Async Pattern for Long-Running Operations:
     * 1. Return immediately to caller (HTTP 202 ACCEPTED)
     * 2. Generate report in background
     * 3. Email result when complete
     * 4. Caller polls for status or receives webhook
     * 
     * Duration: Minutes to hours
     * Thread: Background executor (separate from request)
     * Resource: Single thread consuming report generation
     * 
     * @param reportName report identifier
     * @param userEmail email to send result to
     * @return CompletableFuture<ReportStatusRecord> that completes when done
     */
    @Async("asyncExecutor")
    public CompletableFuture<ReportStatusRecord> generateReportAsync(
        String reportName,
        String userEmail
    ) {
        Objects.requireNonNull(reportName, "Report name cannot be null");
        Objects.requireNonNull(userEmail, "User email cannot be null");
        
        try {
            log.info("Starting async report generation. reportName={}", reportName);
            
            // Simulate long-running report generation
            String reportPath = generateReportFile(reportName);
            
            log.info("Report generated. path={}", reportPath);
            
            // Send completion email
            emailService.sendReportCompleteEmailAsync(userEmail, reportPath);
            
            return CompletableFuture.completedFuture(
                new ReportStatusRecord("COMPLETE", reportPath)
            );
            
        } catch (Exception e) {
            log.error("Report generation failed. reportName={}", reportName, e);
            return CompletableFuture.failedFuture(e);
        }
    }
    
    /**
     * Generate report file (blocking operation).
     * 
     * @param reportName report identifier
     * @return file path of generated report
     */
    private String generateReportFile(String reportName) throws InterruptedException {
        // Simulate CPU-intensive report generation
        Thread.sleep(5000); // 5 seconds
        return "/reports/" + reportName + ".pdf";
    }
}

/**
 * Record for report generation status.
 * 
 * @param status report status (GENERATING, COMPLETE, FAILED)
 * @param reportPath path to generated report file
 */
public record ReportStatusRecord(String status, String reportPath) {
    /**
     * Compact constructor for validation.
     */
    public ReportStatusRecord {
        Objects.requireNonNull(status, "Status cannot be null");
    }
}
```

### Using Async in Controllers

```java
/**
 * REST controller demonstrating async endpoint pattern.
 * 
 * Async HTTP Pattern:
 * - POST request triggers async operation
 * - Response: 202 ACCEPTED (operation accepted, processing async)
 * - Caller: Polls status endpoint or waits for webhook
 * - Non-blocking: Server resources freed immediately
 */
@RestController
@RequestMapping("/api/v1/reports")
@Slf4j
public class ReportAsyncController {
    
    private final ReportAsyncService reportService;
    
    /**
     * Constructor injection.
     * 
     * @param reportService async report service
     */
    public ReportAsyncController(ReportAsyncService reportService) {
        this.reportService = Objects.requireNonNull(reportService);
    }
    
    /**
     * POST endpoint to generate report asynchronously.
     * 
     * Async Endpoint Pattern:
     * - Client POSTs request
     * - Server returns 202 ACCEPTED immediately
     * - Operation continues in background
     * - Client polls status or receives webhook
     * 
     * HTTP Status:
     * - 202 ACCEPTED: Operation accepted, not yet complete
     * - Location header: Status endpoint to poll
     * 
     * Thread Blocking:
     * - Minimal: Only validation and task submission
     * - Fast response: milliseconds
     * - Release: Request thread freed immediately
     * 
     * @param request report generation request
     * @return ResponseEntity with 202 status
     */
    @PostMapping
    public ResponseEntity<ReportStatusResponse> generateReportAsync(
        @Valid @RequestBody GenerateReportRequest request
    ) {
        log.info("Generating report asynchronously. reportName={}", request.reportName());
        
        // Submit async task (returns immediately)
        CompletableFuture<ReportStatusRecord> future = reportService
            .generateReportAsync(request.reportName(), request.userEmail());
        
        // Return 202 ACCEPTED (operation accepted, processing async)
        return ResponseEntity
            .accepted()
            .location(URI.create("/api/v1/reports/status/" + request.reportName()))
            .body(new ReportStatusResponse(
                "PROCESSING",
                "/api/v1/reports/status/" + request.reportName()
            ));
    }
    
    /**
     * GET endpoint to check report generation status.
     * 
     * Polling Pattern:
     * - Client polls status endpoint periodically
     * - Returns current status (PROCESSING, COMPLETE, FAILED)
     * - If COMPLETE: Download URL provided
     * - Typical: Client polls every 5 seconds
     * 
     * @param reportName report identifier
     * @return ResponseEntity with current status
     */
    @GetMapping("/status/{reportName}")
    public ResponseEntity<ReportStatusResponse> getReportStatus(
        @PathVariable String reportName
    ) {
        // Implementation: query database for status
        return ResponseEntity.ok(new ReportStatusResponse("COMPLETE", "/reports/report.pdf"));
    }
}

/**
 * Request record for async report generation.
 * 
 * @param reportName report identifier
 * @param userEmail email to send result to
 */
public record GenerateReportRequest(
    @NotBlank String reportName,
    @Email String userEmail
) {
    /**
     * Compact constructor.
     */
    public GenerateReportRequest {
        Objects.requireNonNull(reportName);
        Objects.requireNonNull(userEmail);
    }
}

/**
 * Response record for async operation status.
 * 
 * @param status current status (PROCESSING, COMPLETE, FAILED)
 * @param statusUrl URL to check status
 */
public record ReportStatusResponse(String status, String statusUrl) {
    /**
     * Compact constructor.
     */
    public ReportStatusResponse {
        Objects.requireNonNull(status);
    }
}
```

### Message Queue Pattern (Spring Integration/RabbitMQ)

```java
/**
 * Event-driven async pattern using message queue.
 * 
 * Message Queue Characteristics:
 * - Producer: Sends message to queue
 * - Consumer: Processes message from queue
 * - Decoupled: Producer doesn't wait for consumer
 * - Durable: Messages persist if consumer down
 * - Scalable: Multiple consumers = parallel processing
 * 
 * Use Case:
 * - Email campaigns (1000s of emails)
 * - Order processing (complex workflow)
 * - Data synchronization between systems
 * - Event sourcing / event-driven architecture
 */
@Configuration
public class MessageQueueConfig {
    
    // Queue names
    public static final String USER_CREATED_QUEUE = "user.created.queue";
    public static final String EMAIL_SEND_QUEUE = "email.send.queue";
    
    /**
     * Declare durable queue for user creation events.
     * Durable = survives broker restart
     * 
     * @return Queue bean
     */
    @Bean
    public Queue userCreatedQueue() {
        return QueueBuilder.durable(USER_CREATED_QUEUE).build();
    }
    
    /**
     * Declare durable queue for email sending.
     * 
     * @return Queue bean
     */
    @Bean
    public Queue emailSendQueue() {
        return QueueBuilder.durable(EMAIL_SEND_QUEUE)
            .withArgument("x-message-ttl", 3600000) // 1 hour TTL
            .build();
    }
}

/**
 * Producer: Publishes events to message queue.
 * 
 * Producer Pattern:
 * 1. Business event occurs (user created)
 * 2. Publish event to queue (non-blocking)
 * 3. Return to caller immediately
 * 4. Queue persistence guarantees delivery
 * 5. Consumer processes message asynchronously
 */
@Service
@Slf4j
public class UserEventProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    /**
     * Constructor injection.
     * 
     * @param rabbitTemplate Spring RabbitMQ template
     */
    public UserEventProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = Objects.requireNonNull(rabbitTemplate);
    }
    
    /**
     * Publish user created event to message queue.
     * 
     * Producer Pattern:
     * - Non-blocking: Publishes to queue and returns immediately
     * - Durable: Message persists in queue
     * - Guaranteed: If queue available, message delivered eventually
     * - Decoupled: Producer doesn't know about consumers
     * 
     * @param event user created event
     * @throws AmqpException if queue unavailable
     */
    public void publishUserCreatedEvent(UserCreatedEvent event) {
        Objects.requireNonNull(event, "Event cannot be null");
        
        try {
            log.info("Publishing user created event. userId={}", event.userId());
            
            // Send message to queue (non-blocking)
            rabbitTemplate.convertAndSend(
                MessageQueueConfig.USER_CREATED_QUEUE,
                event
            );
            
            log.info("User created event published. userId={}", event.userId());
            
        } catch (AmqpException e) {
            log.error("Failed to publish user created event. userId={}", event.userId(), e);
            throw new EventPublishException("Cannot publish user created event", e);
        }
    }
}

/**
 * Consumer: Processes events from message queue.
 * 
 * Consumer Pattern:
 * - Listens to queue continuously
 * - Processes messages one at a time
 * - Acknowledges message after processing
 * - If processing fails: message requeued
 * - Scalable: Multiple consumers process in parallel
 *
 * > Full idempotency pattern with DLQ and manual ACK: **16-RABBITMQ-SAGA.md**
 */
@Service
@Slf4j
public class UserEventConsumer {
    
    private final EmailAsyncService emailService;
    private final ProcessedEventRepository processedEventRepository;
    
    /**
     * Constructor injection.
     * 
     * @param emailService async email service
     * @param processedEventRepository idempotency repository
     */
    public UserEventConsumer(EmailAsyncService emailService,
                             ProcessedEventRepository processedEventRepository) {
        this.emailService = Objects.requireNonNull(emailService);
        this.processedEventRepository = Objects.requireNonNull(processedEventRepository);
    }
    
    /**
     * Listen to user created events and process.
     * Implements idempotency guard to prevent duplicate processing on retry.
     * Full idempotency pattern: see spec/16-RABBITMQ-SAGA.md §Consumer with Idempotency
     * 
     * @param event user created event from queue
     * @throws Exception if processing fails (causes requeue)
     */
    @RabbitListener(queues = MessageQueueConfig.USER_CREATED_QUEUE)
    public void handleUserCreatedEvent(UserCreatedEvent event) throws Exception {
        Objects.requireNonNull(event, "Event cannot be null");

        // Idempotency guard — discard duplicate deliveries
        if (processedEventRepository.existsByEventId(event.userId())) {
            log.warn("Duplicate event discarded: eventId={}", event.userId());
            return;
        }
        // Full idempotency pattern: see spec/16-RABBITMQ-SAGA.md §Consumer with Idempotency
        
        try {
            log.info("Processing user created event. userId={}", event.userId());
            
            // Send verification email
            emailService.sendVerificationEmailAsync(
                event.email(),
                generateToken(event.userId())
            ).get(10, TimeUnit.SECONDS); // Wait max 10 seconds

            processedEventRepository.markAsProcessed(event.userId());
            log.info("User created event processed. userId={}", event.userId());
            
        } catch (TimeoutException e) {
            log.error("Timeout processing user event. userId={}", event.userId(), e);
            // Requeue: Will retry later
            throw new RuntimeException("Processing timeout", e);
            
        } catch (Exception e) {
            log.error("Error processing user event. userId={}", event.userId(), e);
            // Requeue: Will retry later
            throw e;
        }
    }
    
    /**
     * Generate verification token.
     * 
     * @param userId user identifier
     * @return verification token
     */
    private String generateToken(String userId) {
        return UUID.randomUUID().toString();
    }
}

/**
 * Event record for user creation.
 * 
 * @param userId created user identifier (String — MongoDB ID)
 * @param email user's email address
 * @param createdAt UTC timestamp of creation
 */
public record UserCreatedEvent(
    String userId,
    String email,
    Instant createdAt
) {
    /**
     * Compact constructor for validation.
     */
    public UserCreatedEvent {
        Objects.requireNonNull(userId);
        Objects.requireNonNull(email);
        Objects.requireNonNull(createdAt);
    }
}
```

---

## Comparison Table

| Aspect | Synchronous | Asynchronous |
|--------|-------------|--------------|
| **Caller waits?** | Yes | No |
| **Response time** | Slower (depends on operation) | Faster (immediately) |
| **Thread usage** | 1 thread per request | Shared thread pool |
| **Scalability** | Limited by thread pool | High (many async tasks) |
| **Error handling** | Immediate exception | Logged, metrics tracked |
| **User feedback** | Direct (in response) | Polling or webhook |
| **Complexity** | Simple | More complex (state tracking) |
| **Use cases** | Payments, lookups, validations | Emails, reports, notifications |
| **Timeout risk** | High (slow external service) | Low (queued if service slow) |
| **Transaction scope** | Single transaction | Multiple transactions |

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Blocking Async Operations

```java
// WRONG: Blocking on async operation (defeats purpose)
@PostMapping("/users")
public UserResponse createUserWrong(@RequestBody CreateUserRequest request) {
    UserResponse user = userService.createUser(request);
    
    try {
        // This blocks the request thread!
        Future<Void> emailFuture = emailService.sendVerificationEmailAsync(user.email());
        emailFuture.get(); // WRONG: Waits for async operation (not async anymore!)
    } catch (Exception e) {
        // Error
    }
    
    return user;
}

// CORRECT: Fire-and-forget async operation
@PostMapping("/users")
public UserResponse createUserRight(@RequestBody CreateUserRequest request) {
    UserResponse user = userService.createUser(request);
    
    // Schedule async email (don't wait for result)
    emailService.sendVerificationEmailAsync(user.email());
    
    // Return immediately to caller
    return user;
}
```

### ❌ Anti-Pattern 2: Synchronous Email in Critical Path

```java
// WRONG: Blocking email prevents user response
@PostMapping("/users")
public UserResponse createUserWrong(CreateUserRequest request) {
    UserResponse user = service.createUser(request);
    
    try {
        // If email server slow: entire HTTP request slow!
        mailSender.send(buildMessage(user));
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    
    return user;
}

// CORRECT: Async email allows fast response
@PostMapping("/users")
public UserResponse createUserRight(CreateUserRequest request) {
    UserResponse user = service.createUser(request);
    
    // Email sent in background (doesn't block)
    emailService.sendVerificationEmailAsync(user.email());
    
    return user; // Return immediately
}
```

### ❌ Anti-Pattern 3: Missing Timeout on Synchronous HTTP

```java
// WRONG: No timeout on external API call (can hang forever)
public PaymentResult processPayment(String orderId) {
    // If payment gateway hangs: request never returns!
    ResponseEntity<PaymentResponse> response = restTemplate.exchange(
        "https://payment-gateway.com/api/process",
        HttpMethod.POST,
        request,
        PaymentResponse.class
    );
    
    return convertResponse(response);
}

// CORRECT: Always set timeout
public PaymentResult processPayment(String orderId) {
    try {
        // 5-second timeout prevents hanging
        ResponseEntity<PaymentResponse> response = restTemplate.exchange(
            "https://payment-gateway.com/api/process",
            HttpMethod.POST,
            createRequestWithTimeout(5),
            PaymentResponse.class
        );
        
        return convertResponse(response);
        
    } catch (ResourceAccessException e) {
        // Timeout or connection error
        throw new PaymentTimeoutException("Payment gateway timeout", e);
    }
}
```

### ❌ Anti-Pattern 4: Unhandled Async Exceptions

```java
// WRONG: Exception in async method is lost
@Async
public void sendEmailAsync(String email) {
    mailSender.send(message); // If fails: no one knows! No error handling.
}

// CORRECT: Handle async exceptions with monitoring
@Async
public void sendEmailAsync(String email) {
    try {
        mailSender.send(message);
        meterRegistry.counter("email.sent").increment();
    } catch (MailException e) {
        log.error("Email send failed. email={}", email, e);
        meterRegistry.counter("email.failed").increment();
        // Consider: retry mechanism, dead letter queue
    }
}
```

---

## Decision Tree for Synchronous vs Asynchronous

```
Start: Need to call external system / long operation?
│
├─ YES: Does caller need response immediately?
│   ├─ YES: Must have result now
│   │   ├─ YES: Critical operation (payment)?
│   │   │   └─ SYNCHRONOUS + TIMEOUT (5 seconds)
│   │   └─ NO: Non-critical (validation)?
│   │       └─ SYNCHRONOUS (short operation < 1 second)
│   │
│   └─ NO: Can caller wait / poll / use webhook?
│       ├─ Can wait 30 seconds?
│       │   └─ SYNCHRONOUS + LONG TIMEOUT (slow external)
│       └─ Cannot wait?
│           └─ ASYNCHRONOUS (queue / background task)
│
└─ NO: Is it a background job / notification?
    ├─ YES: Email, report, audit log?
    │   └─ ASYNCHRONOUS (async method / message queue)
    └─ NO: User action requiring confirmation?
        └─ SYNCHRONOUS (database, cache lookup)
```

---

## Monitoring Checklist

### Synchronous Operations

- [ ] Track response time (p50, p95, p99)
- [ ] Monitor external service timeout rate
- [ ] Alert on slow endpoints (> 1000ms p95)
- [ ] Track error rate by endpoint
- [ ] Monitor connection pool utilization
- [ ] Database query performance

### Asynchronous Operations

- [ ] Track queue depth (messages pending)
- [ ] Monitor consumer lag (processing delay)
- [ ] Track async task execution time
- [ ] Monitor thread pool utilization
- [ ] Alert on task failures / exceptions
- [ ] Track message dead letter queue (failed processing)
- [ ] Consumer throughput (messages processed per second)

---

## Configuration Summary

**Production Checklist:**

- [ ] Synchronous HTTP calls have 5-30 second timeout
- [ ] Async thread pools configured (core, max, queue)
- [ ] Database connections pooled (max 20)
- [ ] Message queue durable and persistent
- [ ] Email failures monitored and alerted
- [ ] Async task failures logged and tracked
- [ ] Request timeout < 30 seconds (nginx default)
- [ ] Payment operations always synchronous
- [ ] Notifications always asynchronous
- [ ] Circuit breaker for flaky external services
