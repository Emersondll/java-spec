# 21 - HTTP Idempotency

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Pattern for preventing duplicate processing of mutation HTTP requests.
> Apply to any endpoint where a repeated call would cause unintended side effects.

---

## 1. When to Apply

| Requirement | Endpoints |
|---|---|
| **MUST** | Payment creation, refund processing, balance adjustments |
| **MUST** | Order creation, contract generation |
| **SHOULD** | Any POST with real-world side effects (emails, notifications) |
| **NOT needed** | GET (naturally idempotent), DELETE (idempotent by definition), PUT (idempotent by definition) |

---

## 2. The `Idempotency-Key` Header Pattern

```
1. Client generates a UUID v4 and sends it as Idempotency-Key on the request.
2. Server checks MongoDB for an existing entry with that key.
3a. Key NOT found → process normally, store key + response in MongoDB, return result.
3b. Key FOUND     → return cached response immediately, no reprocessing.
4. Key expiration: 24 hours via MongoDB TTL index.
5. Same key + different request body → 422 Unprocessable Entity.
```

**Flow diagram:**

```
Client                          Server                        MongoDB
  │                               │                              │
  │── POST /payments ─────────────▶│                              │
  │   Idempotency-Key: <uuid>      │── findById(<uuid>) ─────────▶│
  │                               │◀─ null (not found) ──────────│
  │                               │   [process payment]           │
  │                               │── save(key, response) ───────▶│
  │◀── 201 Created ───────────────│                              │
  │                               │                              │
  │── POST /payments (retry) ─────▶│                              │
  │   Idempotency-Key: <uuid>      │── findById(<uuid>) ─────────▶│
  │                               │◀─ {responseBody, status} ────│
  │◀── 201 Created (cached) ──────│                              │
```

---

## 3. `IdempotencyKey` MongoDB Document

```java
/**
 * MongoDB document storing processed idempotency keys and their cached responses.
 * TTL index on {@code expiresAt} automatically removes entries after 24 hours.
 *
 * @since 1.0
 * @author Emerson Lima
 */
@Document(collection = "idempotency_keys")
public class IdempotencyKey {

    /** The client-supplied Idempotency-Key header value — used as document _id. */
    @Id
    private String key;

    /** SHA-256 hex digest of the raw request body — detects body mismatch on replay. */
    private String requestHash;

    /** Cached JSON response body to replay on duplicate requests. */
    private String responseBody;

    /** Cached HTTP status code to replay on duplicate requests. */
    private int responseStatus;

    /** Name of the service that owns this key (e.g., "payment-service"). */
    private String serviceName;

    /** TTL field — MongoDB removes this document 24 hours after creation. */
    @Indexed(expireAfterSeconds = 86400)
    private Instant expiresAt;

    /** Timestamp when this key was first processed. */
    private Instant createdAt;

    public String getKey() { return key; }
    public void setKey(String key) { this.key = key; }

    public String getRequestHash() { return requestHash; }
    public void setRequestHash(String requestHash) { this.requestHash = requestHash; }

    public String getResponseBody() { return responseBody; }
    public void setResponseBody(String responseBody) { this.responseBody = responseBody; }

    public int getResponseStatus() { return responseStatus; }
    public void setResponseStatus(int responseStatus) { this.responseStatus = responseStatus; }

    public String getServiceName() { return serviceName; }
    public void setServiceName(String serviceName) { this.serviceName = serviceName; }

    public Instant getExpiresAt() { return expiresAt; }
    public void setExpiresAt(Instant expiresAt) { this.expiresAt = expiresAt; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
}
```

---

## 4. Repository

```java
/**
 * Repository for idempotency key storage and lookup.
 * No custom queries needed — standard CRUD suffices.
 */
public interface IdempotencyKeyRepository extends MongoRepository<IdempotencyKey, String> {
}
```

---

## 5. `IdempotencyFilter` — `OncePerRequestFilter`

```java
/**
 * Servlet filter enforcing idempotency for POST and PATCH requests.
 * Intercepts requests carrying an {@code Idempotency-Key} header,
 * replays cached responses for duplicates, and stores new responses after
 * successful processing.
 *
 * Excluded from JaCoCo coverage (*Filter pattern in pom.xml).
 */
@Slf4j
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private static final String IDEMPOTENCY_HEADER = "Idempotency-Key";

    private final IdempotencyKeyRepository repository;
    private final String serviceName;

    /**
     * @param repository  MongoDB repository for idempotency key storage
     * @param serviceName service name injected from {@code spring.application.name}
     */
    public IdempotencyFilter(IdempotencyKeyRepository repository,
                             @Value("${spring.application.name}") String serviceName) {
        this.repository = repository;
        this.serviceName = serviceName;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String key = request.getHeader(IDEMPOTENCY_HEADER);
        if (key == null || !isMutationMethod(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }

        Optional<IdempotencyKey> existing = repository.findById(key);
        if (existing.isPresent()) {
            replayResponse(existing.get(), request, response);
            return;
        }

        CachedBodyHttpServletRequest cachedRequest = new CachedBodyHttpServletRequest(request);
        String requestHash = sha256(cachedRequest.getBody());

        ContentCachingResponseWrapper wrappedResponse = new ContentCachingResponseWrapper(response);
        chain.doFilter(cachedRequest, wrappedResponse);

        storeResult(key, requestHash, wrappedResponse);
        wrappedResponse.copyBodyToResponse();
    }

    /**
     * Replays the cached response for a duplicate request.
     * Validates request body hash to detect conflicting retries.
     *
     * @param existing  stored idempotency key record
     * @param request   current HTTP request (for body hash comparison)
     * @param response  HTTP response to write the cached result into
     * @throws IOException if writing the response body fails
     */
    private void replayResponse(IdempotencyKey existing,
                                HttpServletRequest request,
                                HttpServletResponse response) throws IOException {
        CachedBodyHttpServletRequest cachedRequest = new CachedBodyHttpServletRequest(request);
        String incomingHash = sha256(cachedRequest.getBody());

        if (!existing.getRequestHash().equals(incomingHash)) {
            log.warn("Idempotency-Key reused with different body: key={}", existing.getKey());
            response.setStatus(HttpStatus.UNPROCESSABLE_ENTITY.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(
                "{\"status\":422,\"code\":\"IDEMPOTENCY_KEY_CONFLICT\"," +
                "\"message\":\"Idempotency-Key already used with a different request body\"}");
            return;
        }

        log.info("Replaying cached response for Idempotency-Key: {}", existing.getKey());
        response.setStatus(existing.getResponseStatus());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(existing.getResponseBody());
    }

    /**
     * Persists the key and response body after successful processing.
     *
     * @param key             idempotency key from request header
     * @param requestHash     SHA-256 of the request body
     * @param wrappedResponse response wrapper containing the captured body
     */
    private void storeResult(String key, String requestHash,
                             ContentCachingResponseWrapper wrappedResponse) {
        String responseBody = new String(wrappedResponse.getContentAsByteArray(),
                                         StandardCharsets.UTF_8);
        IdempotencyKey record = new IdempotencyKey();
        record.setKey(key);
        record.setRequestHash(requestHash);
        record.setResponseBody(responseBody);
        record.setResponseStatus(wrappedResponse.getStatus());
        record.setServiceName(serviceName);
        record.setCreatedAt(Instant.now());
        record.setExpiresAt(Instant.now().plus(24, ChronoUnit.HOURS));
        repository.save(record);
    }

    private boolean isMutationMethod(String method) {
        return "POST".equals(method) || "PATCH".equals(method);
    }

    /**
     * Computes a SHA-256 hex digest of the given byte array.
     *
     * @param body raw request body bytes
     * @return hex-encoded SHA-256 digest
     */
    private String sha256(byte[] body) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(body);
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

---

## 6. Register Filter in `SecurityConfig`

The `IdempotencyFilter` must run **before** the authentication filter so it can replay responses even for unauthenticated duplicates (e.g., retry storms before token validation).

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final GatewayAuthFilter gatewayAuthFilter;
    private final IdempotencyFilter idempotencyFilter;

    public SecurityConfig(GatewayAuthFilter gatewayAuthFilter,
                          IdempotencyFilter idempotencyFilter) {
        this.gatewayAuthFilter = gatewayAuthFilter;
        this.idempotencyFilter = idempotencyFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/health/**",
                                 "/openapi.yaml", "/swagger-ui/**", "/swagger-ui.html")
                    .permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(idempotencyFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(gatewayAuthFilter, IdempotencyFilter.class);
        return http.build();
    }
}
```

---

## 7. Mongock Migration — TTL Index

Since `auto-index-creation: false` in production, the TTL index on `expiresAt` must be created via Mongock.

```java
/**
 * Migration creating the TTL index on the idempotency_keys collection.
 * Must run before any service that uses IdempotencyFilter.
 *
 * @since db-version-003
 */
@ChangeUnit(id = "create-idempotency-ttl-index", order = "003", author = "emerson")
public class IdempotencyKeyIndexMigration {

    /**
     * Creates the TTL index on {@code expiresAt} — documents expire after 24 hours.
     *
     * @param mongoTemplate MongoDB template
     */
    @Execution
    public void createTtlIndex(MongoTemplate mongoTemplate) {
        mongoTemplate.indexOps("idempotency_keys")
            .ensureIndex(new Index()
                .on("expiresAt", Sort.Direction.ASC)
                .expire(Duration.ZERO)
                .named("idx_expires_at_ttl"));
    }

    /**
     * Rollback: drops the TTL index.
     *
     * @param mongoTemplate MongoDB template
     */
    @RollbackExecution
    public void rollback(MongoTemplate mongoTemplate) {
        mongoTemplate.indexOps("idempotency_keys").dropIndex("idx_expires_at_ttl");
    }
}
```

---

## 8. openapi.yaml — Documenting the Header

Add this parameter to every POST/PATCH endpoint that requires idempotency:

```yaml
parameters:
  - name: Idempotency-Key
    in: header
    required: true
    description: >
      UUID v4 generated by the client. The server uses this key to detect and
      replay duplicate requests within a 24-hour window.
      Reusing the same key with a different request body returns 422.
    schema:
      type: string
      format: uuid
      example: "550e8400-e29b-41d4-a716-446655440000"
```

Add the `422` response to the endpoint's `responses` block:

```yaml
responses:
  '201':
    description: Resource created
  '422':
    description: Idempotency-Key reused with a different request body
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
        example:
          status: 422
          code: IDEMPOTENCY_KEY_CONFLICT
          message: "Idempotency-Key already used with a different request body"
```

---

## 9. Error Responses

> Error codes follow the standard in `15-ERROR-RESPONSE.md`. `IDEMPOTENCY_KEY_CONFLICT` is an extension specific to this pattern.
> Note: the `replayResponse` method in `IdempotencyFilter` writes the 422 body directly (bypassing `GlobalExceptionHandler`) because the filter runs before the servlet dispatcher. The response must still match the `ErrorResponse` field set: `{status, code, message}` at minimum — `timestamp`, `path`, and `traceId` should be added for full conformance with `15-ERROR-RESPONSE.md`.

| Situation | Status | `code` |
|---|---|---|
| Same key, different request body | 422 | `IDEMPOTENCY_KEY_CONFLICT` |
| Missing key on protected endpoint | 400 | `VALIDATION_ERROR` |
| Malformed key (not UUID format) | 400 | `VALIDATION_ERROR` |

---

## 10. Per-Endpoint Checklist

- [ ] `Idempotency-Key` header declared in `openapi.yaml` for all financial/order POST/PATCH endpoints
- [ ] `IdempotencyFilter` added to `SecurityConfig` before `UsernamePasswordAuthenticationFilter`
- [ ] `idempotency_keys` TTL index created via Mongock migration (order `003` or higher)
- [ ] `requestHash` (SHA-256 of body) stored and compared on replay
- [ ] 422 response with `IDEMPOTENCY_KEY_CONFLICT` code returned on body mismatch
- [ ] `IdempotencyFilter` excluded from JaCoCo (`*Filter` pattern in `pom.xml`)
- [ ] Integration test covers three cases: first call processes, duplicate replays, body mismatch returns 422
- [ ] `auto-index-creation: false` in production profile (TTL index only via Mongock)

---

## Related Specs

| Spec | Relationship |
|---|---|
| `15-ERROR-RESPONSE.md` | Standard error envelope — `IDEMPOTENCY_KEY_CONFLICT` extends the code table |
| `18-MONGODB-INDEXES.md` | TTL index pattern used in the Mongock migration (Section 7) |
| `14-SECURITY.md` | `GatewayAuthFilter` registration order relative to `IdempotencyFilter` |
| `20-MAVEN-POM.md` | `*Filter` JaCoCo exclusion pattern |

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
