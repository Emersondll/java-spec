# 15 - Error Response Standard

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory HTTP error envelope for all microservices.
> Every error response MUST follow this format — no exceptions.

---

## 1. Standard Envelope

Every error returned by the API MUST have exactly this JSON body:

```json
{
  "timestamp": "2026-06-17T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_ERROR",
  "message": "Field 'email' is required",
  "path": "/api/v1/users",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | ISO 8601 UTC | Yes | Exact moment the error occurred |
| `status` | int | Yes | Numeric HTTP status code |
| `error` | String | Yes | Standard HTTP status description |
| `code` | String | Yes | Internal business code (SNAKE_UPPER_CASE) |
| `message` | String | Yes | Human-readable message for the API developer |
| `path` | String | Yes | URI of the request that generated the error |
| `traceId` | String | Yes | Distributed tracing ID from MDC `traceId` |

**Fields FORBIDDEN in the response:**
- Stack traces
- Internal database or framework messages
- Java class or package names
- Sensitive data (passwords, tokens, SSN)

---

## 2. Internal Codes (`code`) by Domain

### Validation
| `code` | Status | Situation |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Bean Validation failed (`@Valid`) |
| `INVALID_FORMAT` | 400 | Wrong type or format (e.g., invalid date) |
| `MISSING_REQUIRED_FIELD` | 400 | Required field is absent |

### Authentication / Authorization
| `code` | Status | Situation |
|---|---|---|
| `UNAUTHORIZED` | 401 | Token missing or invalid |
| `FORBIDDEN` | 403 | Valid token but insufficient permission (includes IDOR) |

### Resource
| `code` | Status | Situation |
|---|---|---|
| `RESOURCE_NOT_FOUND` | 404 | Entity does not exist |
| `CONFLICT` | 409 | Uniqueness violation (duplicate email, etc.) |

### Server
| `code` | Status | Situation |
|---|---|---|
| `INTERNAL_ERROR` | 500 | Unexpected error — never leaks internal detail |

---

## 3. Implementation — `ErrorResponse` Record

```java
/**
 * Standard HTTP error envelope returned by all microservices.
 * Immutable, JSON-serializable with no null or internal fields exposed.
 *
 * @param timestamp UTC instant when the error occurred
 * @param status    numeric HTTP status code
 * @param error     standard HTTP status description (e.g., "Bad Request")
 * @param code      internal business code in SNAKE_UPPER_CASE
 * @param message   human-readable message for the API developer
 * @param path      URI of the request that generated the error
 * @param traceId   distributed tracing identifier from MDC
 */
public record ErrorResponse(
    Instant timestamp,
    int status,
    String error,
    String code,
    String message,
    String path,
    String traceId
) {
    /**
     * Factory method to build the envelope from request data.
     *
     * @param status  HTTP status
     * @param code    internal business code
     * @param message descriptive message
     * @param request current HTTP request
     * @return ErrorResponse populated with timestamp and traceId from MDC
     */
    public static ErrorResponse of(HttpStatus status, String code,
                                   String message, HttpServletRequest request) {
        return new ErrorResponse(
            Instant.now(),
            status.value(),
            status.getReasonPhrase(),
            code,
            message,
            request.getRequestURI(),
            MDC.get("traceId")
        );
    }
}
```

---

## 4. Implementation — `GlobalExceptionHandler`

```java
/**
 * Centralized exception handler for all REST endpoints.
 * Maps domain and framework exceptions to the standard ErrorResponse envelope.
 */
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * Handles bean validation failures (@Valid / @Validated).
     *
     * @param ex      validation exception with details of invalid fields
     * @param request current HTTP request
     * @return 400 with all invalid field messages concatenated
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex,
                                          HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ErrorResponse.of(HttpStatus.BAD_REQUEST, "VALIDATION_ERROR", message, request);
    }

    /**
     * Handles resource not found errors.
     *
     * @param ex      resource not found exception
     * @param request current HTTP request
     * @return 404 with domain message
     */
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest request) {
        return ErrorResponse.of(HttpStatus.NOT_FOUND, "RESOURCE_NOT_FOUND", ex.getMessage(), request);
    }

    /**
     * Handles uniqueness constraint violations (duplicate email, existing slug, etc.).
     *
     * @param ex      conflict exception
     * @param request current HTTP request
     * @return 409 with domain message
     */
    @ExceptionHandler(ConflictException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleConflict(ConflictException ex,
                                        HttpServletRequest request) {
        return ErrorResponse.of(HttpStatus.CONFLICT, "CONFLICT", ex.getMessage(), request);
    }

    /**
     * Fallback handler for unmapped exceptions.
     * Never leaks internal detail — logs the stack trace server-side only.
     *
     * @param ex      unexpected exception
     * @param request current HTTP request
     * @return 500 with generic message
     */
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleUnexpected(Exception ex, HttpServletRequest request) {
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

---

## 5. Base Domain Exceptions

```java
/**
 * Base exception for resources not found.
 * Thrown when an entity does not exist by the given ID or search criteria.
 *
 * @param message descriptive message with the resource type and identifier searched
 */
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

/**
 * Base exception for uniqueness conflicts.
 * Thrown when an operation violates a unique constraint (e.g., duplicate email).
 *
 * @param message descriptive message with the conflicting field and value
 */
public class ConflictException extends RuntimeException {
    public ConflictException(String message) {
        super(message);
    }
}
```

---

## 6. Per-Service Checklist

- [ ] `ErrorResponse` record present in `common/dto/`
- [ ] `GlobalExceptionHandler` with `@RestControllerAdvice` in `common/exception/`
- [ ] `ResourceNotFoundException` and `ConflictException` in `common/exception/`
- [ ] No endpoint returns a stack trace or internal message in the response body
- [ ] Fallback handler (`Exception.class`) present and logging server-side only
- [ ] MDC `traceId` included in every error response
- [ ] Unit tests on `GlobalExceptionHandler` covering all handlers

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
