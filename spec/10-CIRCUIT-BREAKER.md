# Circuit Breaker Pattern - Resilience4j Best Practices

> See also: **09-SYNC-ASYNC-COMMUNICATION.md** — HTTP client patterns and timeout configuration
> See also: **05-OBSERVABILITY.md** — full Micrometer and Prometheus setup
> See also: **24-CICD-SECRETS.md** — secrets management via environment variables

## Circuit Breaker Fundamentals

**Circuit Breaker** is a design pattern that prevents cascading failures when calling external services.

### States

```
CLOSED (Normal)
    ↓ [Failure threshold reached]
OPEN (Rejecting calls)
    ↓ [Wait timeout]
HALF_OPEN (Testing)
    ↓ [Success] → CLOSED
    ↓ [Failure] → OPEN
```

### When to Use Circuit Breaker

✅ **Always use for:**
- External API calls (payment gateway, shipping, weather)
- Database calls to different cluster
- Message queue operations
- Cache failures
- Any network-based operation

❌ **Never use for:**
- Local method calls
- In-memory operations
- Synchronous database on same cluster

---

## Resilience4j Setup

### Maven Dependencies

> **Spring Boot 4 note:** With Spring Cloud 2025.1.0 BOM managing the versions, use the Spring Cloud starter — the BOM handles version compatibility with Spring Boot 4.0.7. Do NOT pin version numbers manually.

```xml
<!-- Preferred: managed by Spring Cloud BOM -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

<!-- Metrics bridge for Micrometer / Prometheus -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-micrometer</artifactId>
</dependency>
```

If using Resilience4j directly without Spring Cloud (rare), use artifact `resilience4j-spring-boot3` — the artifact name has not changed for Spring Boot 4 compatibility (as of Resilience4j 2.x). Version is managed by the Spring Cloud BOM.

```xml
<!-- Direct use (only if Spring Cloud starter is not available) -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

### Configuration

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    configs:
      default:
        # Failure threshold: 50% = circuit opens after 50% failures
        failure-rate-threshold: 50
        # Minimum calls: Circuit only evaluates after 10 calls
        minimum-number-of-calls: 10
        # Automatic retry: Wait 1 minute before testing again
        wait-duration-in-open-state: 60000
        # Half-open state: Try 3 calls to verify service recovered
        permitted-number-of-calls-in-half-open-state: 3
        # Slow call rate: 50% slow calls = open circuit
        slow-call-rate-threshold: 50
        # Slow call duration: Calls > 2 seconds = slow
        slow-call-duration-threshold: 2000
        # Record exceptions: These exceptions count as failures
        recorded-exception-types:
          - java.net.SocketTimeoutException
          - java.net.ConnectException
          - org.springframework.web.client.HttpClientErrorException
          - org.springframework.web.client.HttpServerErrorException
        # Ignore exceptions: These don't count as failures
        ignored-exception-types:
          - java.io.IOException
    
    instances:
      # Payment gateway circuit breaker
      payment-gateway:
        base-config: default
        failure-rate-threshold: 30  # Strict: 30% = fail
        wait-duration-in-open-state: 120000  # Wait 2 minutes
      
      # Shipping API circuit breaker
      shipping-api:
        base-config: default
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60000
      
      # User service (internal) - lenient
      user-service:
        base-config: default
        failure-rate-threshold: 80  # 80% before fail
        wait-duration-in-open-state: 30000

  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 1000
        retry-exceptions:
          - java.net.SocketTimeoutException
          - java.net.ConnectException
    
    instances:
      payment-gateway:
        base-config: default
        max-attempts: 3
        wait-duration: 2000

  # Metrics export to Prometheus
  metrics:
    enabled: true
    export:
      prometheus:
        enabled: true
```

---

## Circuit Breaker Implementation

### Payment Service with Circuit Breaker

```java
/**
 * Service for payment processing with circuit breaker resilience.
 * 
 * Circuit Breaker Strategy:
 * - Protects against payment gateway failures
 * - Fails fast instead of waiting for timeout
 * - Automatic recovery after cooldown period
 * - Metrics and monitoring of circuit state
 * 
 * Thread Safety: Stateless, thread-safe
 * @see PaymentGatewayClient for external API integration
 */
@Service
@Slf4j
public class PaymentCircuitBreakerService {
    
    private final PaymentGatewayClient paymentClient;
    private final MeterRegistry meterRegistry;
    
    /**
     * Constructor injection of dependencies.
     * 
     * @param paymentClient external payment gateway client
     * @param meterRegistry metrics registry
     */
    public PaymentCircuitBreakerService(
        PaymentGatewayClient paymentClient,
        MeterRegistry meterRegistry
    ) {
        this.paymentClient = Objects.requireNonNull(paymentClient);
        this.meterRegistry = Objects.requireNonNull(meterRegistry);
    }
    
    /**
     * Process payment with circuit breaker protection.
     * 
     * Circuit Breaker Protection:
     * - CLOSED: Normal operation, all calls go through
     * - OPEN: Gateway failing, reject calls immediately (fail fast)
     * - HALF_OPEN: Testing if gateway recovered
     * 
     * Failure Handling:
     * - Timeout: Exception after 5 seconds
     * - 5xx errors: Count as failure
     * - 4xx errors: Don't count as failure (client error)
     * 
     * @param orderId order identifier for payment
     * @param amount payment amount in cents
     * @return payment confirmation
     * @throws CircuitBreakerOpenException if gateway circuit is open
     * @throws PaymentException if payment processing fails
     */
    @CircuitBreaker(
        name = "payment-gateway",
        fallback = "processPaymentFallback"
    )
    public PaymentConfirmationRecord processPayment(
        String orderId,
        BigDecimal amount
    ) {
        Objects.requireNonNull(orderId, "Order ID cannot be null");
        Objects.requireNonNull(amount, "Amount cannot be null");
        
        log.info("Processing payment via circuit breaker. orderId={}, amount={}", orderId, amount);
        
        try {
            PaymentResponseRecord response = paymentClient.processPayment(
                new PaymentRequestRecord(orderId, amount)
            );
            
            log.info("Payment successful. orderId={}, transactionId={}", 
                orderId, response.transactionId());
            
            meterRegistry.counter("payment.success").increment();
            
            return new PaymentConfirmationRecord(
                "SUCCESS",
                response.transactionId(),
                Instant.now()
            );
            
        } catch (HttpServerErrorException e) {
            // 5xx: Gateway error, counts as failure for circuit breaker
            log.error("Payment gateway error. orderId={}, status={}", orderId, e.getStatusCode(), e);
            meterRegistry.counter("payment.gateway_error").increment();
            throw new PaymentGatewayException("Payment gateway unavailable", e);
            
        } catch (HttpClientErrorException e) {
            // 4xx: Client error, doesn't count as failure
            log.error("Payment validation error. orderId={}, status={}", orderId, e.getStatusCode());
            meterRegistry.counter("payment.validation_error").increment();
            throw new PaymentValidationException(e.getMessage(), e);
            
        } catch (ResourceAccessException e) {
            // Timeout or connection error, counts as failure
            log.error("Payment timeout. orderId={}", orderId, e);
            meterRegistry.counter("payment.timeout").increment();
            throw new PaymentTimeoutException("Payment processing timeout", e);
        }
    }
    
    /**
     * Fallback method when circuit breaker is OPEN.
     * 
     * Fallback Strategy:
     * - Circuit OPEN: Gateway failing, don't call it
     * - Return cached result or fail gracefully
     * - Reduce load on failing gateway
     * 
     * Called by Resilience4j when circuit opens.
     * Method signature must match original (same parameters).
     * 
     * @param orderId order identifier
     * @param amount payment amount
     * @param throwable exception that triggered fallback
     * @return payment confirmation with pending status
     */
    public PaymentConfirmationRecord processPaymentFallback(
        String orderId,
        BigDecimal amount,
        Throwable throwable
    ) {
        log.warn("Payment circuit breaker OPEN. Using fallback. orderId={}", orderId, throwable);
        
        meterRegistry.counter("payment.circuit_open").increment();
        
        // Strategy 1: Return pending status (customer can retry later)
        return new PaymentConfirmationRecord(
            "PENDING",
            "CIRCUIT_BREAKER_OPEN_" + UUID.randomUUID(),
            Instant.now()
        );
        
        // Strategy 2: Check local cache if available
        // return paymentCache.getLastSuccessful(orderId)
        //     .orElseThrow(() -> new PaymentCircuitOpenException("Payment service unavailable"));
        
        // Strategy 3: Use backup gateway
        // return backupPaymentClient.processPayment(orderId, amount);
    }
}

/**
 * Record for payment confirmation response.
 * 
 * @param status payment status (SUCCESS, PENDING, FAILED)
 * @param transactionId payment gateway transaction ID
 * @param processedAt UTC instant of processing
 */
public record PaymentConfirmationRecord(
    String status,
    String transactionId,
    Instant processedAt
) {
    /**
     * Compact constructor for validation.
     */
    public PaymentConfirmationRecord {
        Objects.requireNonNull(status, "Status cannot be null");
        Objects.requireNonNull(transactionId, "Transaction ID cannot be null");
        Objects.requireNonNull(processedAt, "ProcessedAt cannot be null");
    }
}
```

### External API Client with Retry

```java
/**
 * HTTP client for payment gateway with retry policy.
 * 
 * Resilience Pattern:
 * - Circuit breaker: Protects against cascading failures
 * - Retry: Automatic retry on transient errors
 * - Timeout: Fail fast if service slow
 * - Metrics: Track all failures
 * 
 * Thread Safety: Stateless, thread-safe
 */
@Component
@Slf4j
public class PaymentGatewayClient {
    
    private final RestTemplate restTemplate;
    private final String paymentApiKey;
    private static final String GATEWAY_URL = "https://payment-gateway.com/api/v1/process";
    private static final int TIMEOUT_SECONDS = 5;
    
    /**
     * Constructor injection. API key injected via @Value — never use System.getenv().
     * See spec/24-CICD-SECRETS.md for secrets management.
     *
     * @param restTemplate  configured HTTP client
     * @param paymentApiKey payment gateway API key from environment variable
     */
    public PaymentGatewayClient(RestTemplate restTemplate,
                                @Value("${PAYMENT_API_KEY}") String paymentApiKey) {
        this.restTemplate = Objects.requireNonNull(restTemplate);
        this.paymentApiKey = Objects.requireNonNull(paymentApiKey, "PAYMENT_API_KEY must be set");
    }
    
    /**
     * Call payment gateway with retry and timeout protection.
     * 
     * Retry Strategy:
     * - Max 3 attempts for transient errors
     * - Wait 2 seconds between retries
     * - Exponential backoff recommended in production
     * 
     * Timeout:
     * - 5 second socket timeout
     * - Fail fast instead of hanging
     * 
     * @param request payment request
     * @return payment response
     * @throws HttpClientErrorException if 4xx error
     * @throws HttpServerErrorException if 5xx error
     * @throws ResourceAccessException if timeout/connection error
     */
    @Retry(name = "payment-gateway")
    public PaymentResponseRecord processPayment(PaymentRequestRecord request) {
        Objects.requireNonNull(request, "Request cannot be null");
        
        log.debug("Calling payment gateway. orderId={}", request.orderId());
        
        try {
            HttpEntity<PaymentRequestRecord> httpRequest = new HttpEntity<>(
                request,
                createHeaders()
            );
            
            ResponseEntity<PaymentResponseRecord> response = restTemplate.exchange(
                GATEWAY_URL,
                HttpMethod.POST,
                httpRequest,
                PaymentResponseRecord.class
            );
            
            log.debug("Payment gateway response. status={}", response.getStatusCode());
            
            return response.getBody();
            
        } catch (HttpStatusCodeException e) {
            // Re-throw for Resilience4j to handle
            throw e;
        }
    }
    
    /**
     * Creates HTTP headers for gateway authentication.
     * 
     * @return headers with API key
     */
    private HttpHeaders createHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(paymentApiKey);
        return headers;
    }
}

/**
 * Record for payment gateway request.
 * Note: these records are illustrative examples for the circuit breaker pattern.
 * In a real project, define them once in the shared domain module.
 * See also: spec/09-SYNC-ASYNC-COMMUNICATION.md for the HTTP client context.
 *
 * @param orderId order identifier (String — MongoDB ID)
 * @param amount payment amount
 */
public record PaymentRequestRecord(String orderId, BigDecimal amount) {
    /**
     * Compact constructor for validation.
     */
    public PaymentRequestRecord {
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(amount);
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
}

/**
 * Record for payment gateway response.
 * 
 * @param status payment status
 * @param transactionId transaction identifier
 */
public record PaymentResponseRecord(String status, String transactionId) {
    /**
     * Compact constructor for validation.
     */
    public PaymentResponseRecord {
        Objects.requireNonNull(status);
        Objects.requireNonNull(transactionId);
    }
}
```

### REST Controller with Circuit Breaker

```java
/**
 * REST controller for payment endpoints with circuit breaker protection.
 * 
 * Circuit breaker automatically:
 * - Fails fast when gateway is down
 * - Returns appropriate HTTP status
 * - Tracks metrics for monitoring
 */
@RestController
@RequestMapping("/api/v1/payments")
@Validated
@Slf4j
public class PaymentCircuitBreakerController {
    
    private final PaymentCircuitBreakerService paymentService;
    
    /**
     * Constructor injection.
     * 
     * @param paymentService payment processing service
     */
    public PaymentCircuitBreakerController(PaymentCircuitBreakerService paymentService) {
        this.paymentService = Objects.requireNonNull(paymentService);
    }
    
    /**
     * POST endpoint to process payment.
     * 
     * Circuit Breaker Behavior:
     * - CLOSED: Normal processing (timeout 5 seconds)
     * - OPEN: Returns 503 Service Unavailable (fail fast)
     * - HALF_OPEN: Attempts gateway (testing recovery)
     * 
     * Error Responses:
     * - 503 Service Unavailable: Payment gateway down (circuit open)
     * - 504 Gateway Timeout: Gateway slow (retry timeout)
     * - 400 Bad Request: Invalid payment data
     * - 500 Internal Server Error: Unexpected error
     * 
     * @param orderId order identifier
     * @param amount payment amount
     * @return ResponseEntity with confirmation
     */
    @PostMapping("/process")
    public ResponseEntity<PaymentConfirmationRecord> processPayment(
        @RequestParam String orderId,
        @RequestParam BigDecimal amount
    ) {
        log.info("Processing payment. orderId={}, amount={}", orderId, amount);
        
        try {
            PaymentConfirmationRecord confirmation = paymentService.processPayment(
                orderId,
                amount
            );
            
            return ResponseEntity.ok(confirmation);
            
        } catch (CircuitBreakerOpenException e) {
            // Circuit OPEN: Gateway failing, return 503
            log.warn("Payment circuit breaker open. orderId={}", orderId);
            return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(new PaymentConfirmationRecord(
                    "SERVICE_UNAVAILABLE",
                    "CIRCUIT_BREAKER_OPEN",
                    Instant.now()
                ));
            
        } catch (PaymentTimeoutException e) {
            // Timeout: Return 504
            log.error("Payment timeout. orderId={}", orderId);
            return ResponseEntity
                .status(HttpStatus.GATEWAY_TIMEOUT)
                .body(new PaymentConfirmationRecord(
                    "TIMEOUT",
                    "PAYMENT_TIMEOUT",
                    Instant.now()
                ));
            
        } catch (PaymentValidationException e) {
            // Validation error: Return 400
            log.error("Payment validation error. orderId={}", orderId);
            return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(new PaymentConfirmationRecord(
                    "INVALID",
                    "VALIDATION_ERROR",
                    Instant.now()
                ));
            
        } catch (Exception e) {
            // Unexpected error: Return 500
            log.error("Unexpected payment error. orderId={}", orderId, e);
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new PaymentConfirmationRecord(
                    "ERROR",
                    "INTERNAL_ERROR",
                    Instant.now()
                ));
        }
    }
}
```

---

## Monitoring Circuit Breaker Metrics

### Prometheus Queries

```promql
# Circuit breaker state: 0=CLOSED, 1=OPEN, 2=HALF_OPEN
resilience4j_circuitbreaker_state{name="payment-gateway"}

# Failure rate percentage
resilience4j_circuitbreaker_failure_rate{name="payment-gateway"}

# Number of calls in last period
rate(resilience4j_circuitbreaker_calls_total{name="payment-gateway"}[5m])

# Success rate
rate(resilience4j_circuitbreaker_calls_total{name="payment-gateway",kind="successful"}[5m]) / 
rate(resilience4j_circuitbreaker_calls_total{name="payment-gateway"}[5m])

# Circuit breaker open events
increase(resilience4j_circuitbreaker_state{name="payment-gateway",state="open"}[1h])
```

### Health Check Integration

```java
/**
 * Custom health indicator for circuit breaker status.
 */
@Component
public class CircuitBreakerHealthIndicator implements HealthIndicator {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    /**
     * Constructor injection.
     * 
     * @param circuitBreakerRegistry circuit breaker registry
     */
    public CircuitBreakerHealthIndicator(CircuitBreakerRegistry circuitBreakerRegistry) {
        this.circuitBreakerRegistry = Objects.requireNonNull(circuitBreakerRegistry);
    }
    
    /**
     * Report circuit breaker health.
     * 
     * @return DEGRADED if any circuit is OPEN, UP otherwise
     */
    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();
        
        circuitBreakerRegistry.getAllCircuitBreakers()
            .forEach(cb -> {
                String status = cb.getState().toString();
                builder.withDetail(cb.getName(), status);
                
                if (cb.getState() == CircuitBreaker.State.OPEN) {
                    builder.degraded();  // Downgrade to DEGRADED (not UP)
                }
            });
        
        return builder.build();
    }
}
```

---

## Circuit Breaker Checklist

- [ ] All external API calls protected by circuit breaker
- [ ] Failure threshold configured (30-50% for critical services)
- [ ] Timeout set to realistic value (5-30 seconds)
- [ ] Fallback strategy implemented (cache, default value, error response)
- [ ] Metrics exported to Prometheus
- [ ] Alerting configured for OPEN circuits
- [ ] Half-open state tests configured
- [ ] Retry policy configured for transient errors
- [ ] Error responses use correct HTTP status codes
- [ ] Circuit breaker state exposed in health endpoint
- [ ] Monitoring dashboard shows circuit state changes
- [ ] Documentation of expected failure modes

---

## Best Practices Summary

```
✅ DO:
- Use circuit breaker for ALL external service calls
- Set realistic timeouts (faster = more failures but better UX)
- Implement fallback strategy
- Monitor circuit state changes
- Log when circuit opens/closes
- Use appropriate HTTP status codes

❌ DON'T:
- Circuit breaker on local method calls
- Set timeout too high (defeats purpose)
- Ignore circuit breaker open events
- Call failing service during OPEN state
- Forget to implement fallback
- Use synchronous calls with tight timeouts
```
