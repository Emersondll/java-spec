# 18 - MongoDB Index Conventions

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory index conventions for MongoDB across all microservices.
> Every field used in filters, sorting, or uniqueness lookups MUST have a declared index.

---

## 1. Fundamental Rule

**Never** let MongoDB create indexes implicitly.
Every index MUST be declared explicitly via Spring Data annotations or `@Document`.

---

## 2. Annotations by Use Case

### Unique field

```java
@Indexed(unique = true)
private String email;
```

### Non-unique filter field

```java
@Indexed
private String status;
```

### Compound index (combined filter)

```java
@Document(collection = "orders")
@CompoundIndexes({
    @CompoundIndex(name = "idx_user_status", def = "{'userId': 1, 'status': 1}"),
    @CompoundIndex(name = "idx_created_status", def = "{'createdAt': -1, 'status': 1}")
})
public class Order { }
```

### TTL — automatic document expiration

```java
@Indexed(expireAfterSeconds = 86400)  // 24 hours
private Instant expiresAt;
```

### Text index (full-text search)

```java
@TextIndexed
private String description;
```

---

## 3. Naming Convention

Mandatory format: `idx_<field1>_<field2>`

| Pattern | Example |
|---|---|
| Single field | `idx_email` |
| Compound | `idx_user_status` |
| TTL | `idx_expires_at_ttl` |
| Text | `idx_description_text` |

---

## 4. Fields That ALWAYS Have an Index

| Field | Type | Reason |
|---|---|---|
| `createdAt` | Simple (desc) | Default sort for list endpoints |
| `updatedAt` | Simple (desc) | Sync queries |
| `userId` / `ownerId` | Simple | Owner filtering (anti-IDOR) |
| Status fields | Simple | State filters (`ACTIVE`, `PENDING`) |
| Business uniqueness fields | Unique | Email, SSN, slug, external code |

---

## 5. Complete Entity Example

```java
/**
 * Order entity stored in MongoDB.
 * Indexes explicitly declared for all fields used in queries.
 *
 * @since 1.0
 * @author Emerson Lima
 */
@Document(collection = "orders")
@CompoundIndexes({
    @CompoundIndex(name = "idx_user_status",    def = "{'userId': 1, 'status': 1}"),
    @CompoundIndex(name = "idx_created_status", def = "{'createdAt': -1, 'status': 1}")
})
public class Order {

    @Id
    private String id;

    /** Owner user ID — indexed for anti-IDOR filtering. */
    @Indexed
    private String userId;

    /** Order status — part of compound indexes. */
    private OrderStatus status;

    /** Unique order number per tenant. */
    @Indexed(unique = true)
    private String orderNumber;

    /** Creation date — indexed desc for default list sorting. */
    @Indexed
    private Instant createdAt;

    /** Last update date — indexed for sync queries. */
    @Indexed
    private Instant updatedAt;

    // Manual getters and setters (no Lombok @Data/@Getter/@Setter)
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

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
}
```

---

## 6. Automatic vs Manual Index Creation

### Development environment

```yaml
spring:
  data:
    mongodb:
      auto-index-creation: true   # convenient for dev/test
```

### Production environment

```yaml
spring:
  data:
    mongodb:
      auto-index-creation: false  # MANDATORY in production
```

In production, indexes MUST be created via **Mongock** (migration) or a versioned script before deployment.

---

## 7. Mongock — Index Migrations

For indexes on existing collections with data (to avoid long locks):

```java
/**
 * Index migration for the orders collection.
 * Creates missing indexes without dropping existing ones.
 *
 * @since db-version-002
 */
@ChangeUnit(id = "create-order-indexes", order = "002", author = "emerson")
public class CreateOrderIndexesMigration {

    /**
     * Creates compound and unique indexes on the orders collection.
     *
     * @param mongoTemplate MongoDB template for index creation
     */
    @Execution
    public void createIndexes(MongoTemplate mongoTemplate) {
        IndexOperations ops = mongoTemplate.indexOps("orders");

        ops.ensureIndex(new Index().on("userId", Sort.Direction.ASC)
                                   .on("status", Sort.Direction.ASC)
                                   .named("idx_user_status"));

        ops.ensureIndex(new Index().on("orderNumber", Sort.Direction.ASC)
                                   .unique()
                                   .named("idx_order_number_unique"));

        ops.ensureIndex(new Index().on("createdAt", Sort.Direction.DESC)
                                   .named("idx_created_at"));
    }

    /**
     * Rollback: removes indexes created by this migration.
     *
     * @param mongoTemplate MongoDB template
     */
    @RollbackExecution
    public void rollback(MongoTemplate mongoTemplate) {
        IndexOperations ops = mongoTemplate.indexOps("orders");
        ops.dropIndex("idx_user_status");
        ops.dropIndex("idx_order_number_unique");
        ops.dropIndex("idx_created_at");
    }
}
```

---

## 8. Anti-Pattern Rules

| Forbidden | Correct |
|---|---|
| Query on unindexed filter field | Declare `@Indexed` on the field |
| `auto-index-creation: true` in production | `false` in prod + Mongock migration |
| Compound index with high-cardinality field first | Low-cardinality field first (status before userId) |
| `$where` or unanchored regex | Spring Data derived queries |
| Index on a frequently mutated field | Assess write impact before indexing |

---

## 9. Per-Collection Checklist

- [ ] Every field used in `findBy*` has `@Indexed` or is part of `@CompoundIndex`
- [ ] Every business uniqueness field has `@Indexed(unique = true)`
- [ ] `createdAt` and `updatedAt` are indexed
- [ ] `userId` / `ownerId` is indexed
- [ ] Index names follow `idx_<field1>_<field2>`
- [ ] `auto-index-creation: false` in `application-prod.yml`
- [ ] Mongock migration created for indexes on collections with existing data
- [ ] TTL indexes documented with expiration time in the field comment

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
