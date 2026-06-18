# 17 - Pagination Standard

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory standard for all endpoints that return lists.
> Every listing endpoint MUST be paginated — unpaginated responses are FORBIDDEN.

---

## 1. Strategy: Offset-Based (Default)

Use **offset pagination** via Spring Data MongoDB's `Pageable`.

> Use **cursor pagination** only for real-time feeds with high write volume
> (e.g., notification timelines). For all other cases, offset is mandatory.

---

## 2. Input Parameters (Query String)

| Parameter | Type | Default | Maximum | Description |
|---|---|---|---|---|
| `page` | int | `0` | — | Page number (zero-based) |
| `size` | int | `20` | `100` | Items per page |
| `sort` | String | `createdAt,desc` | — | Field and direction (`field,asc\|desc`) |

**Mandatory rules:**
- `size` maximum: **100**. Values above → `400 BAD REQUEST`
- Negative `page` → `400 BAD REQUEST`
- Invalid `sort` → silently ignored, use default

---

## 3. Response Envelope

```json
{
  "content": [ ],
  "page": {
    "number": 0,
    "size": 20,
    "totalElements": 152,
    "totalPages": 8
  }
}
```

| Field | Type | Description |
|---|---|---|
| `content` | Array | Items on the current page |
| `page.number` | int | Current page number (zero-based) |
| `page.size` | int | Requested page size |
| `page.totalElements` | long | Total items in the database for the applied filter |
| `page.totalPages` | int | Total available pages |

---

## 4. `PageResponse<T>` Record

```java
/**
 * Paginated response envelope for all list endpoints.
 * Wraps page content and pagination metadata in a standardized format.
 *
 * @param <T>     type of items returned in the page
 * @param content list of items on the current page
 * @param page    pagination metadata
 */
public record PageResponse<T>(
    List<T> content,
    PageMetadata page
) {
    /**
     * Converts a Spring Data {@link Page} to the standard envelope.
     *
     * @param <T>        item type
     * @param springPage page returned by the repository
     * @return PageResponse with content and metadata populated
     */
    public static <T> PageResponse<T> from(Page<T> springPage) {
        return new PageResponse<>(
            springPage.getContent(),
            new PageMetadata(
                springPage.getNumber(),
                springPage.getSize(),
                springPage.getTotalElements(),
                springPage.getTotalPages()
            )
        );
    }

    /**
     * Pagination metadata.
     *
     * @param number        current page number (zero-based)
     * @param size          page size
     * @param totalElements total elements in the database for the applied filter
     * @param totalPages    total available pages
     */
    public record PageMetadata(
        int number,
        int size,
        long totalElements,
        int totalPages
    ) {}
}
```

---

## 5. Controller — Example

```java
/**
 * Lists users in a paginated manner.
 *
 * @param pageable pagination and sorting parameters injected by Spring
 *                 (query params: page, size, sort)
 * @return 200 with PageResponse envelope containing users for the requested page
 */
@GetMapping
public ResponseEntity<PageResponse<UserResponse>> listUsers(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable) {
    validatePageSize(pageable.getPageSize());
    PageResponse<UserResponse> response = userService.findAll(pageable);
    return ResponseEntity.ok(response);
}

/**
 * Validates that the page size does not exceed the allowed maximum.
 *
 * @param size requested size
 * @throws IllegalArgumentException if size > 100
 */
private void validatePageSize(int size) {
    if (size > 100) {
        throw new IllegalArgumentException("Page size must not exceed 100");
    }
}
```

---

## 6. Service — Example

```java
/**
 * Retrieves all active users in a paginated manner.
 *
 * @param pageable pagination and sorting parameters
 * @return PageResponse with users for the requested page
 */
public PageResponse<UserResponse> findAll(Pageable pageable) {
    Page<User> page = userRepository.findAllByActiveTrue(pageable);
    Page<UserResponse> mapped = page.map(UserResponse::from);
    return PageResponse.from(mapped);
}
```

---

## 7. Allowed Sort Fields per Endpoint

Each endpoint MUST document allowed sort fields in `openapi.yaml`.
Disallowed fields must be silently ignored (use default) or return `400`, as documented.

**Example in openapi.yaml:**
```yaml
parameters:
  - name: sort
    in: query
    description: "Field and direction. Allowed: createdAt, name. Ex: createdAt,desc"
    schema:
      type: string
      default: "createdAt,desc"
```

---

## 8. Per-Listing-Endpoint Checklist

- [ ] No endpoint returns a list without pagination
- [ ] `@PageableDefault` defined with `size=20` and default sort `createdAt,desc`
- [ ] Maximum `size` validation (> 100 → 400)
- [ ] `PageResponse<T>` used as return type (never Spring's `Page<T>` directly)
- [ ] Allowed sort fields documented in `openapi.yaml`
- [ ] Unit test covers: valid page, max size exceeded, empty result

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
