# OpenAPI REST Reference — Spring Boot Java 26

> **Maintainer:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
> **Last Updated:** 2026-06-17
> **API Version:** v1
> **Base URL:** `http://localhost:8080/api/v1`

---

## 🚨 MANDATORY RULE — Static File Documentation

> **OpenAPI/Swagger documentation MUST always be a static YAML file.**
> **Using annotations in code (`@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition`) is FORBIDDEN.**

### Why a static file?

| Approach | Problem |
|---|---|
| Code annotations (`@Tag`, `@Operation`) | Pollutes code with documentation responsibilities; violates SRP; annotations silently become stale |
| Static file (`openapi.yaml`) | Documentation versioned independently; readable without compiling; editable by any tool (Swagger Editor, Postman, Insomnia) |

### Required structure

```
src/main/resources/
└── static/
    └── openapi.yaml          ← complete OpenAPI 3.0.x specification
```

### springdoc configuration (application.yml)

```yaml
springdoc:
  swagger-ui:
    path: /swagger-ui.html
    url: /openapi.yaml          # points to the static file
    operations-sorter: method
  api-docs:
    enabled: false              # disables annotation-driven dynamic generation
```

### Minimal openapi.yaml structure

```yaml
openapi: 3.0.3
info:
  title: Service Name API
  description: |
    Full description of the service and its domains.
  version: 1.0.0
  contact:
    name: Emerson Lima
    url: https://github.com/Emersondll

servers:
  - url: http://localhost:8080
    description: Local development server

tags:
  - name: ResourceName
    description: Endpoint group description

paths:
  /resource:
    post:
      tags: [ResourceName]
      summary: Short summary
      description: Detailed description
      operationId: uniqueOperationId
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RequestSchema'
            examples:
              main_case:
                summary: Main use case
                value:
                  field: "value"
      responses:
        '201':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ResponseSchema'
        '400':
          $ref: '#/components/responses/ValidationError'
        '500':
          $ref: '#/components/responses/InternalError'

components:
  schemas:
    RequestSchema:
      type: object
      required: [field]
      properties:
        field:
          type: string
          description: Field description
          example: "example value"

    ResponseSchema:
      type: object
      properties:
        result:
          type: string
          example: "success"

    ValidationErrorResponse:
      type: object
      properties:
        code:
          type: string
          example: "VALIDATION_ERROR"
        message:
          type: string
          example: "Input validation failed"
        errors:
          type: object
          additionalProperties:
            type: string
          example:
            field: "Field is required"
        timestamp:
          type: string
          format: date-time

    ErrorResponse:
      type: object
      properties:
        code:
          type: string
          example: "INTERNAL_ERROR"
        message:
          type: string
        timestamp:
          type: string
          format: date-time

  responses:
    ValidationError:
      description: Validation failure
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationErrorResponse'

    InternalError:
      description: Unexpected server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

### openapi.yaml checklist

- [ ] File located at `src/main/resources/static/openapi.yaml`
- [ ] springdoc configured with `url: /openapi.yaml` and `api-docs.enabled: false`
- [ ] Zero Swagger annotations in code (`@Tag`, `@Operation`, `@ApiResponse`, `@Schema`, `@OpenAPIDefinition`)
- [ ] All endpoints documented with `summary`, `description`, `operationId`
- [ ] All request bodies with `schema` + `examples`
- [ ] All responses with `schema` + `examples` (including 400 and 500 errors)
- [ ] All schema fields with `description` and `example`
- [ ] Swagger UI accessible at `/swagger-ui.html` with the application running

---

## Table of Contents

1. [Global Conventions](#1-global-conventions)
2. [Standard HTTP Status Codes](#2-standard-http-status-codes)
3. [Error Response Schemas](#3-error-response-schemas)
4. [User Endpoints](#4-user-endpoints-apiv1users)
5. [Product Endpoints](#5-product-endpoints-apiv1products)
6. [Order Endpoints](#6-order-endpoints-apiv1orders)
7. [Authentication Endpoints](#7-authentication-endpoints-apiv1auth)
8. [Health & Actuator Endpoints](#8-health--actuator-endpoints)
9. [Pagination Contract](#9-pagination-contract)
10. [Common Headers](#10-common-headers)

---

## 1. Global Conventions

| Convention | Rule |
|---|---|
| **Base path** | `/api/v1/` |
| **Content-Type** | `application/json` |
| **Date format** | ISO 8601 UTC — `yyyy-MM-ddTHH:mm:ssZ` |
| **ID type** | `String` (MongoDB ObjectId as hex string) |
| **Null fields** | Omitted from response (not returned as `null`) |
| **Pagination** | Via `?page=0&size=20&sort=createdAt,desc` — response uses `PageResponse<T>` |
| **Versioning** | URL path: `/api/v1/`, `/api/v2/` |
| **Correlation ID** | Header `X-Correlation-Id` required on all requests |

---

## 2. Standard HTTP Status Codes

| Code | Meaning | When Used |
|---|---|---|
| `200 OK` | Success | GET, PUT with body returned |
| `201 Created` | Resource created | POST — includes `Location` header |
| `204 No Content` | Success, no body | DELETE, PUT/PATCH with no return |
| `400 Bad Request` | Validation failed | Invalid input, missing required fields |
| `401 Unauthorized` | Not authenticated | Missing or invalid JWT token |
| `403 Forbidden` | Not authorized | Authenticated but lacks permission |
| `404 Not Found` | Resource not found | Entity does not exist |
| `409 Conflict` | Duplicate resource | Email/unique constraint violation |
| `422 Unprocessable Entity` | Business rule violation | Domain invariant not satisfied |
| `500 Internal Server Error` | Unexpected error | Unhandled exception, DB unavailable |
| `503 Service Unavailable` | Downstream failure | Circuit breaker open, external service down |

---

## 3. Error Response Schemas

### 3.1 Generic Error (`ErrorResponse`)

Returned for `4xx` and `5xx` errors. Schema defined in `15-ERROR-RESPONSE.md`.

```json
{
  "timestamp": "2026-06-17T10:30:00Z",
  "status": 404,
  "error": "Not Found",
  "code": "RESOURCE_NOT_FOUND",
  "message": "User with id a1b2c3d4 not found",
  "path": "/api/v1/users/a1b2c3d4",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

| Field | Type | Description |
|---|---|---|
| `timestamp` | `String` | ISO 8601 UTC — when the error occurred |
| `status` | `int` | Numeric HTTP status code |
| `error` | `String` | Standard HTTP status description |
| `code` | `String` | Machine-readable error code (UPPER_SNAKE_CASE) |
| `message` | `String` | Human-readable description |
| `path` | `String` | URI of the request |
| `traceId` | `String` | Distributed tracing ID from MDC |

---

### 3.2 Validation Error

Returned when `@Valid` constraint fails — HTTP `400`.

```json
{
  "timestamp": "2026-06-17T10:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDATION_ERROR",
  "message": "email: Email must be valid; name: Name is required",
  "path": "/api/v1/users",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

---

### 3.3 Error Codes Reference

| Code | HTTP | Description |
|---|---|---|
| `RESOURCE_NOT_FOUND` | 404 | Entity not found by ID |
| `VALIDATION_ERROR` | 400 | Bean validation failure |
| `CONFLICT` | 409 | Unique constraint violation |
| `UNAUTHORIZED` | 401 | Invalid or missing auth token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `INTERNAL_ERROR` | 500 | Unexpected server failure |

---

## 4. User Endpoints `/api/v1/users`

### 4.1 Create User

```
POST /api/v1/users
```

**Request Headers:**

| Header | Required | Value |
|---|---|---|
| `Content-Type` | Yes | `application/json` |
| `X-Correlation-Id` | Yes | UUID v4 |

**Request Body:**

```json
{
  "name": "Emerson Lima",
  "email": "emerson@example.com",
  "password": "Secure@123"
}
```

| Field | Type | Constraint | Description |
|---|---|---|---|
| `name` | `String` | NotBlank, 1–100 chars | User full name |
| `email` | `String` | NotBlank, valid email | Unique email address |
| `password` | `String` | NotBlank, min 8 chars | Plain text (hashed in service) |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `201 Created` | User created successfully | `UserResponse` + `Location` header |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `409 Conflict` | Email already registered | `ErrorResponse` — `CONFLICT` |
| `500 Internal Server Error` | Database failure | `ErrorResponse` — `INTERNAL_ERROR` |

**201 Response Body:**

```json
{
  "id": "a1b2c3d4e5f6",
  "name": "Emerson Lima",
  "email": "emerson@example.com",
  "createdAt": "2026-06-17T10:30:00Z",
  "status": "PENDING_VERIFICATION"
}
```

**Response Headers (201):**

```
Location: /api/v1/users/a1b2c3d4e5f6
```

---

### 4.2 Get User by ID

```
GET /api/v1/users/{id}
```

**Path Variables:**

| Variable | Type | Description |
|---|---|---|
| `id` | `String` | MongoDB user identifier |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | User found | `UserResponse` |
| `404 Not Found` | User does not exist | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` — `INTERNAL_ERROR` |

**200 Response Body:**

```json
{
  "id": "a1b2c3d4e5f6",
  "name": "Emerson Lima",
  "email": "emerson@example.com",
  "createdAt": "2026-06-17T10:30:00Z",
  "status": "ACTIVE"
}
```

---

### 4.3 List Users (Paginated)

```
GET /api/v1/users
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | `int` | `0` | Zero-based page number |
| `size` | `int` | `20` | Items per page (max 100) |
| `sort` | `String` | `createdAt,desc` | Sort field and direction |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Success | `PageResponse<UserResponse>` |
| `400 Bad Request` | Invalid pagination params | `ErrorResponse` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

**200 Response Body:**

```json
{
  "content": [
    {
      "id": "a1b2c3d4e5f6",
      "name": "Emerson Lima",
      "email": "emerson@example.com",
      "createdAt": "2026-06-17T10:30:00Z",
      "status": "ACTIVE"
    }
  ],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 1,
    "totalPages": 1
  }
}
```

---

### 4.4 Update User

```
PUT /api/v1/users/{id}
```

**Path Variables:**

| Variable | Type | Description |
|---|---|---|
| `id` | `String` | MongoDB user identifier |

**Request Body:**

```json
{
  "name": "Emerson Lima Updated"
}
```

| Field | Type | Constraint | Description |
|---|---|---|---|
| `name` | `String` | NotBlank, 1–100 chars | Updated full name |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Updated successfully | `UserResponse` |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `404 Not Found` | User does not exist | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

---

### 4.5 Delete User

```
DELETE /api/v1/users/{id}
```

**Path Variables:**

| Variable | Type | Description |
|---|---|---|
| `id` | `String` | MongoDB user identifier |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `204 No Content` | Deleted successfully | _(empty)_ |
| `404 Not Found` | User does not exist | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

---

### 4.6 UserResponse Schema

```json
{
  "id": "a1b2c3d4e5f6",
  "name": "Emerson Lima",
  "email": "emerson@example.com",
  "createdAt": "2026-06-17T10:30:00Z",
  "status": "ACTIVE"
}
```

| Field | Type | Description |
|---|---|---|
| `id` | `String` | System-generated MongoDB identifier |
| `name` | `String` | Full display name |
| `email` | `String` | Registered email (never updated) |
| `createdAt` | `String` | ISO 8601 UTC creation timestamp |
| `status` | `Enum` | `ACTIVE`, `INACTIVE`, `PENDING_VERIFICATION`, `SUSPENDED` |

---

## 5. Product Endpoints `/api/v1/products`

### 5.1 Create Product

```
POST /api/v1/products
```

**Request Body:**

```json
{
  "name": "Product Name",
  "description": "Detailed description",
  "price": 99.99,
  "stockQuantity": 100,
  "categoryId": "cat001"
}
```

| Field | Type | Constraint | Description |
|---|---|---|---|
| `name` | `String` | NotBlank, max 200 chars | Product name |
| `description` | `String` | max 1000 chars | Optional description |
| `price` | `BigDecimal` | NotNull, min 0.01 | Unit price |
| `stockQuantity` | `Integer` | NotNull, min 0 | Available stock |
| `categoryId` | `String` | NotNull | Reference to category |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `201 Created` | Created successfully | `ProductResponse` + `Location` header |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `404 Not Found` | Category not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `409 Conflict` | Duplicate product name | `ErrorResponse` — `CONFLICT` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

---

### 5.2 Get Product by ID

```
GET /api/v1/products/{id}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Found | `ProductResponse` |
| `404 Not Found` | Product not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |

---

### 5.3 List Products

```
GET /api/v1/products
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | `int` | `0` | Page number |
| `size` | `int` | `20` | Page size (max 100) |
| `sort` | `String` | `name,asc` | Sort field and direction |
| `categoryId` | `String` | _(none)_ | Filter by category |
| `minPrice` | `BigDecimal` | _(none)_ | Minimum price filter |
| `maxPrice` | `BigDecimal` | _(none)_ | Maximum price filter |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Success | `PageResponse<ProductResponse>` |
| `400 Bad Request` | Invalid params | `ErrorResponse` |

---

### 5.4 Update Product

```
PUT /api/v1/products/{id}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Updated | `ProductResponse` |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `404 Not Found` | Not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |

---

### 5.5 Delete Product

```
DELETE /api/v1/products/{id}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `204 No Content` | Deleted | _(empty)_ |
| `404 Not Found` | Not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `422 Unprocessable Entity` | Product has active orders | `ErrorResponse` |

---

### 5.6 ProductResponse Schema

```json
{
  "id": "prod001",
  "name": "Product Name",
  "description": "Detailed description",
  "price": 99.99,
  "stockQuantity": 100,
  "category": {
    "id": "cat001",
    "name": "Electronics"
  },
  "createdAt": "2026-06-17T10:30:00Z",
  "status": "AVAILABLE"
}
```

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique MongoDB identifier |
| `name` | `String` | Product name |
| `description` | `String` | Optional description |
| `price` | `BigDecimal` | Unit price |
| `stockQuantity` | `Integer` | Available stock |
| `category` | `CategorySummary` | Embedded category reference |
| `createdAt` | `String` | ISO 8601 UTC timestamp |
| `status` | `Enum` | `AVAILABLE`, `OUT_OF_STOCK`, `DISCONTINUED` |

---

## 6. Order Endpoints `/api/v1/orders`

### 6.1 Create Order

```
POST /api/v1/orders
```

**Request Body:**

```json
{
  "customerId": "usr001",
  "items": [
    {
      "productId": "prod010",
      "quantity": 2
    },
    {
      "productId": "prod025",
      "quantity": 1
    }
  ]
}
```

| Field | Type | Constraint | Description |
|---|---|---|---|
| `customerId` | `String` | NotNull | Reference to existing user |
| `items` | `List<OrderItemRequest>` | NotEmpty | Minimum 1 item |
| `items[].productId` | `String` | NotNull | Product reference |
| `items[].quantity` | `Integer` | NotNull, min 1 | Quantity ordered |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `201 Created` | Order created | `OrderResponse` + `Location` header |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `404 Not Found` | Customer or product not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `422 Unprocessable Entity` | Insufficient stock | `ErrorResponse` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

---

### 6.2 Get Order by ID

```
GET /api/v1/orders/{id}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Found | `OrderResponse` |
| `404 Not Found` | Order not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |

---

### 6.3 List Orders by Customer

```
GET /api/v1/orders?customerId={customerId}
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `customerId` | `String` | _(required)_ | Filter by customer |
| `status` | `String` | _(none)_ | Filter by order status |
| `page` | `int` | `0` | Page number |
| `size` | `int` | `20` | Page size |
| `sort` | `String` | `createdAt,desc` | Sort |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Success | `PageResponse<OrderResponse>` |
| `400 Bad Request` | Missing customerId | `ErrorResponse` |

---

### 6.4 Cancel Order

```
PATCH /api/v1/orders/{id}/cancel
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Cancelled | `OrderResponse` (status = `CANCELLED`) |
| `404 Not Found` | Order not found | `ErrorResponse` — `RESOURCE_NOT_FOUND` |
| `422 Unprocessable Entity` | Order already shipped/delivered | `ErrorResponse` |

---

### 6.5 OrderResponse Schema

```json
{
  "id": "ord100",
  "customerId": "usr001",
  "items": [
    {
      "productId": "prod010",
      "productName": "Product Name",
      "quantity": 2,
      "unitPrice": 99.99,
      "subtotal": 199.98
    }
  ],
  "totalAmount": 199.98,
  "status": "CONFIRMED",
  "createdAt": "2026-06-17T10:30:00Z",
  "updatedAt": "2026-06-17T10:35:00Z"
}
```

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique order MongoDB identifier |
| `customerId` | `String` | Reference to customer |
| `items` | `List<OrderItemResponse>` | Order line items |
| `totalAmount` | `BigDecimal` | Sum of all subtotals |
| `status` | `Enum` | `PENDING`, `CONFIRMED`, `SHIPPED`, `DELIVERED`, `CANCELLED` |
| `createdAt` | `String` | ISO 8601 UTC creation timestamp |
| `updatedAt` | `String` | ISO 8601 UTC last update timestamp |

---

## 7. Authentication Endpoints `/api/v1/auth`

### 7.1 Login

```
POST /api/v1/auth/login
```

**Request Body:**

```json
{
  "email": "emerson@example.com",
  "password": "Secure@123"
}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Authenticated | `AuthResponse` |
| `400 Bad Request` | Validation failed | `ErrorResponse` — `VALIDATION_ERROR` |
| `401 Unauthorized` | Invalid credentials | `ErrorResponse` — `UNAUTHORIZED` |
| `500 Internal Server Error` | Unexpected failure | `ErrorResponse` |

**200 Response Body:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

| Field | Type | Description |
|---|---|---|
| `accessToken` | `String` | JWT bearer token |
| `refreshToken` | `String` | Token for renewal |
| `tokenType` | `String` | Always `Bearer` |
| `expiresIn` | `int` | Seconds until expiry |

---

### 7.2 Refresh Token

```
POST /api/v1/auth/refresh
```

**Request Body:**

```json
{
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | Token refreshed | `AuthResponse` |
| `401 Unauthorized` | Refresh token expired/invalid | `ErrorResponse` — `UNAUTHORIZED` |

---

### 7.3 Logout

```
POST /api/v1/auth/logout
```

**Request Headers:**

| Header | Required | Value |
|---|---|---|
| `Authorization` | Yes | `Bearer {accessToken}` |

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `204 No Content` | Logged out | _(empty)_ |
| `401 Unauthorized` | Token invalid | `ErrorResponse` — `UNAUTHORIZED` |

---

## 8. Health & Actuator Endpoints

### 8.1 Health Check

```
GET /actuator/health
```

**Responses:**

| Status | Condition | Body |
|---|---|---|
| `200 OK` | All components healthy | `HealthResponse` |
| `503 Service Unavailable` | One or more components degraded | `HealthResponse` |

**200 Response Body:**

```json
{
  "status": "UP",
  "components": {
    "mongo": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "ping": { "status": "UP" }
  }
}
```

---

### 8.2 Liveness Probe

```
GET /actuator/health/liveness
```

**Responses:**

| Status | Body |
|---|---|
| `200 OK` | `{ "status": "UP" }` |
| `503 Service Unavailable` | `{ "status": "DOWN" }` |

---

### 8.3 Readiness Probe

```
GET /actuator/health/readiness
```

**Responses:**

| Status | Body |
|---|---|
| `200 OK` | `{ "status": "UP" }` |
| `503 Service Unavailable` | `{ "status": "OUT_OF_SERVICE" }` |

---

### 8.4 Metrics

```
GET /actuator/metrics
GET /actuator/metrics/{metricName}
GET /actuator/prometheus
```

**Responses:** All return `200 OK` with respective metric payloads. `metrics` and `prometheus` require authentication (see `14-SECURITY.md`).

---

## 9. Pagination Contract

All list endpoints use the `PageResponse<T>` standard wrapper (see `17-PAGINATION.md`):

```json
{
  "content": [],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

| Field | Type | Description |
|---|---|---|
| `content` | `Array` | Page items |
| `page.size` | `int` | Items per page |
| `page.number` | `int` | Current page (zero-based) |
| `page.totalElements` | `long` | Total records |
| `page.totalPages` | `int` | Total pages |

**Sort direction values:** `asc`, `desc`

**Example:** `GET /api/v1/users?page=2&size=10&sort=name,asc`

---

## 10. Common Headers

### Request Headers

| Header | Required | Description |
|---|---|---|
| `Content-Type` | On POST/PUT | `application/json` |
| `Authorization` | On secured endpoints | `Bearer {JWT}` |
| `X-Correlation-Id` | Yes | UUID v4 for distributed tracing |
| `Accept` | Optional | `application/json` (default) |

### Response Headers

| Header | Present | Description |
|---|---|---|
| `Content-Type` | Always | `application/json` |
| `Location` | On 201 | URI of created resource |
| `X-Correlation-Id` | Always | Echoed from request or generated |
| `X-Request-Id` | Always | Unique request identifier |

---

## Quick Reference Summary

| Method | Path | Status | Description |
|---|---|---|---|
| `POST` | `/api/v1/users` | 201 / 400 / 409 | Create user |
| `GET` | `/api/v1/users/{id}` | 200 / 404 | Get user by ID |
| `GET` | `/api/v1/users` | 200 | List users paginated |
| `PUT` | `/api/v1/users/{id}` | 200 / 400 / 404 | Update user |
| `DELETE` | `/api/v1/users/{id}` | 204 / 404 | Delete user |
| `POST` | `/api/v1/products` | 201 / 400 / 409 | Create product |
| `GET` | `/api/v1/products/{id}` | 200 / 404 | Get product by ID |
| `GET` | `/api/v1/products` | 200 | List products paginated |
| `PUT` | `/api/v1/products/{id}` | 200 / 400 / 404 | Update product |
| `DELETE` | `/api/v1/products/{id}` | 204 / 404 / 422 | Delete product |
| `POST` | `/api/v1/orders` | 201 / 400 / 404 / 422 | Create order |
| `GET` | `/api/v1/orders/{id}` | 200 / 404 | Get order by ID |
| `GET` | `/api/v1/orders` | 200 | List orders paginated |
| `PATCH` | `/api/v1/orders/{id}/cancel` | 200 / 404 / 422 | Cancel order |
| `POST` | `/api/v1/auth/login` | 200 / 400 / 401 | Authenticate user |
| `POST` | `/api/v1/auth/refresh` | 200 / 401 | Refresh JWT token |
| `POST` | `/api/v1/auth/logout` | 204 / 401 | Logout user |
| `GET` | `/actuator/health` | 200 / 503 | Application health |
| `GET` | `/actuator/health/liveness` | 200 / 503 | Liveness probe |
| `GET` | `/actuator/health/readiness` | 200 / 503 | Readiness probe |

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
