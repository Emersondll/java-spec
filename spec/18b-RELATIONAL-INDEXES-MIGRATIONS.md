# 18b - Relational Index & Migration Conventions

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory index and migration conventions for Relational Databases (PostgreSQL/MySQL)
> across all microservices using JPA.
> Every field used in filters, sorting, joins, or uniqueness lookups MUST have a declared index.

---

## 1. Fundamental Rule

**Never** let Hibernate generate or update database schemas in production (`ddl-auto: update` is strictly forbidden).
All schema modifications MUST be done via versioned SQL migration scripts using **Flyway**.
All JPA entities MUST have their indexes and constraints declared explicitly via JPA annotations to maintain parity with the physical schema.

---

## 2. Annotations by Use Case

### Unique Column

```java
@Column(nullable = false, unique = true)
private String email;
```

### Simple Index (Non-Unique Filter Column)

```java
@Table(name = "users", indexes = {
    @Index(name = "idx_users_status", columnList = "status")
})
```

### Compound Index (Combined Filter / Sort Column)

```java
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_created_status", columnList = "created_at DESC, status")
})
```

---

## 3. Naming Convention

Mandatory format: `idx_<table_name>_<columns>`

| Pattern | Example |
|---|---|
| Simple Index | `idx_users_status` |
| Compound Index | `idx_orders_user_status` |
| Foreign Key Index | `idx_orders_customer_id` |
| Unique Constraint | `idx_users_email_unique` |

---

## 4. Fields That ALWAYS Have an Index

| Field | SQL Type | Reason |
|---|---|---|
| Foreign Keys (`@ManyToOne`, `@OneToOne`) | simple index | Accelerates joins and prevents table scans |
| `created_at` | simple index (desc) | Default sort for list endpoints |
| `user_id` / `owner_id` | simple index | Owner filtering (anti-IDOR) |
| Status fields | simple index | State filters (`ACTIVE`, `PENDING`) |
| Business uniqueness fields | unique index | Email, SSN, slug, external code |

---

## 5. Complete Entity Example

```java
package com.company.domain;

import jakarta.persistence.*;
import java.time.Instant;

/**
 * Order entity stored in a relational database.
 * Indexes and constraints explicitly declared to match Flyway schema.
 *
 * @since 1.0
 * @author Emerson Lima
 */
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_created_at", columnList = "created_at DESC")
})
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private String userId; // Owner user ID (from gateway token)

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private OrderStatus status;

    @Column(name = "order_number", nullable = false, unique = true, length = 50)
    private String orderNumber;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    // Framework default constructor
    protected Order() { }

    // Manual getters and setters (no Lombok used unless project already has it)
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }

    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }

    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }

    public Instant getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Instant updatedAt) { this.updatedAt = updatedAt; }

    @PrePersist
    protected void onCreate() {
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = Instant.now();
    }
}
```

---

## 6. Flyway — Migrations Standard

Migration scripts MUST be placed in `src/main/resources/db/migration/`.

### Naming Convention

`V<YYYYMMDDHHMMSS>__<description>.sql` (e.g., `V20260617153000__create_orders_table.sql`)

### Database Migration Example (`V20260617153000__create_orders_table.sql`)

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    status VARCHAR(30) NOT NULL,
    order_number VARCHAR(50) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Unique index
CREATE UNIQUE INDEX idx_orders_number_unique ON orders(order_number);

-- Compound and search indexes
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

---

## 7. Hibernate DDL Auto Configurations

### Development Profile (`application-dev.yml`)

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate     # ALWAYS validate — Flyway manages schema, even in dev
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true
```

> Flyway is active in all environments including dev. Never use `ddl-auto: create`, `update`, or `create-drop` — schema changes go through versioned migration scripts.

> See also: **19-SPRING-PROFILES.md** — authoritative reference for all profile configuration including JPA/Flyway variants.

### Production / Staging Profiles (`application-prod.yml`)

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate     # MANDATORY in staging and prod
  flyway:
    enabled: true            # Database migrations executed on startup
```

---

## 8. Anti-Pattern Rules

| Forbidden | Correct |
|---|---|
| `ddl-auto: update` or `create` in production | `validate` in prod + Flyway migration scripts |
| Foreign Key columns without index | Declare `@Index` on join/FK columns |
| `TIMESTAMP` without timezone | Use `TIMESTAMP WITH TIME ZONE` (maps to `Instant`) |
| Unindexed query filter fields in `@Table` | Define corresponding JPA `@Index` on entity |

---

## 9. Per-Entity Checklist

- [ ] Every join/FK column has `@Index`
- [ ] Every query filter column has `@Index`
- [ ] Unique columns have `@Column(unique = true)` or unique index in `@Table`
- [ ] `createdAt` and `updatedAt` are mapped to `Instant` and indexed
- [ ] `ddl-auto: validate` is set for production configurations
- [ ] Corresponding Flyway migration file is placed under `db/migration/`
- [ ] Index names follow the `idx_<table_name>_<columns>` pattern
- [ ] Primary keys use `BIGSERIAL` (JPA `Long`) or database-appropriate autoincrement ID

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
