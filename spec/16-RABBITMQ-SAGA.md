# 16 - RabbitMQ & SAGA Event Patterns

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory patterns for asynchronous communication via RabbitMQ and SAGA orchestration.
> Every microservice that publishes or consumes events MUST follow this document.

---

## 1. Exchange and Queue Topology

### Naming Convention

```
Exchange:  <domain>.events           (topic exchange)
Queue:     <domain>.<service>.<action> (durable, auto-delete: false)
DLQ:       <domain>.<service>.<action>.dlq
Routing:   <domain>.<entity>.<event>
```

**Concrete example for the order flow:**

| Resource | Name |
|---|---|
| Exchange (publisher) | `order.events` |
| Queue (consumer - payment) | `order.payment.process` |
| DLQ | `order.payment.process.dlq` |
| Routing key | `order.created` |

### Exchange Types

| Exchange | Type | Purpose |
|---|---|---|
| `<domain>.events` | `topic` | Domain events (SAGA) |
| `<domain>.commands` | `direct` | Point-to-point commands |
| `dlx` | `direct` | Global Dead Letter Exchange |

---

## 2. Event Envelope (MUST)

Every published event MUST have exactly this envelope:

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "order.created",
  "aggregateId": "order-123",
  "aggregateType": "Order",
  "occurredAt": "2026-06-17T10:30:00Z",
  "version": 1,
  "correlationId": "req-abc123",
  "payload": { }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `eventId` | UUID | Yes | Unique event ID (used for idempotency) |
| `eventType` | String | Yes | `<domain>.<entity>.<past-tense>` |
| `aggregateId` | String | Yes | ID of the aggregate that generated the event |
| `aggregateType` | String | Yes | Aggregate type (e.g., `Order`, `User`) |
| `occurredAt` | ISO 8601 UTC | Yes | Moment the event occurred in the domain |
| `version` | int | Yes | Payload schema version (starts at 1) |
| `correlationId` | String | No | Correlation ID from the originating HTTP request |
| `payload` | Object | Yes | Event-specific data |

---

## 3. Envelope Record

```java
/**
 * Generic envelope for all domain events published via RabbitMQ.
 * Immutable, JSON-serializable. The {@code payload} field is typed by the concrete event.
 *
 * @param <T>           payload type of the event
 * @param eventId       unique UUID for this event — used for consumer idempotency
 * @param eventType     event type in {@code domain.entity.past-tense} format
 * @param aggregateId   identifier of the aggregate that originated the event
 * @param aggregateType name of the aggregate type (e.g., {@code Order})
 * @param occurredAt    UTC instant when the event occurred in the domain
 * @param version       payload schema version; increment when adding required fields
 * @param correlationId correlation ID from the originating HTTP request (nullable)
 * @param payload       event-specific data
 */
public record DomainEvent<T>(
    UUID eventId,
    String eventType,
    String aggregateId,
    String aggregateType,
    Instant occurredAt,
    int version,
    String correlationId,
    T payload
) {
    /**
     * Factory method to create a new event with auto-generated ID and timestamp.
     *
     * @param <T>           payload type
     * @param eventType     event type
     * @param aggregateId   aggregate ID
     * @param aggregateType aggregate type
     * @param correlationId correlation ID (nullable)
     * @param payload       event data
     * @return DomainEvent ready for publishing
     */
    public static <T> DomainEvent<T> of(String eventType, String aggregateId,
                                         String aggregateType, String correlationId,
                                         T payload) {
        return new DomainEvent<>(
            UUID.randomUUID(),
            eventType,
            aggregateId,
            aggregateType,
            Instant.now(),
            1,
            correlationId,
            payload
        );
    }
}
```

---

## 4. RabbitMQ Configuration

```java
/**
 * RabbitMQ infrastructure configuration: exchanges, queues, DLQ and bindings.
 * Declare ALL resources here — never rely on the broker to create them implicitly.
 */
@Configuration
public class RabbitMQConfig {

    public static final String ORDER_EXCHANGE    = "order.events";
    public static final String DLX              = "dlx";
    public static final String PAYMENT_QUEUE    = "order.payment.process";
    public static final String PAYMENT_DLQ      = "order.payment.process.dlq";
    public static final String ORDER_CREATED_RK = "order.created";

    /**
     * Main order events exchange (topic).
     *
     * @return durable TopicExchange
     */
    @Bean
    public TopicExchange orderEventsExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE).durable(true).build();
    }

    /**
     * Global Dead Letter Exchange (direct) for manual reprocessing.
     *
     * @return durable DirectExchange
     */
    @Bean
    public DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange(DLX).durable(true).build();
    }

    /**
     * Payment processing queue with DLQ configured.
     *
     * @return durable Queue with dead-letter-exchange and dead-letter-routing-key
     */
    @Bean
    public Queue paymentProcessQueue() {
        return QueueBuilder.durable(PAYMENT_QUEUE)
            .withArgument("x-dead-letter-exchange", DLX)
            .withArgument("x-dead-letter-routing-key", PAYMENT_DLQ)
            .build();
    }

    /**
     * Dead Letter Queue for payment messages that failed after all retries.
     *
     * @return durable DLQ
     */
    @Bean
    public Queue paymentDeadLetterQueue() {
        return QueueBuilder.durable(PAYMENT_DLQ).build();
    }

    /**
     * Binding: order.events → order.payment.process with routing key order.created.
     *
     * @param paymentProcessQueue consumer queue
     * @param orderEventsExchange source exchange
     * @return configured Binding
     */
    @Bean
    public Binding paymentQueueBinding(Queue paymentProcessQueue,
                                       TopicExchange orderEventsExchange) {
        return BindingBuilder.bind(paymentProcessQueue)
            .to(orderEventsExchange)
            .with(ORDER_CREATED_RK);
    }
}
```

---

## 5. Publisher

```java
/**
 * Domain event publisher via RabbitMQ.
 * Uses {@code RabbitTemplate} with delivery confirmation (publisher confirms).
 */
@Slf4j
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    /**
     * @param rabbitTemplate RabbitMQ template with publisher confirms enabled
     */
    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * Publishes the {@code order.created} event to the orders exchange.
     * The {@code correlationId} is extracted from MDC for distributed tracing.
     *
     * @param event order created payload
     */
    public void publishOrderCreated(OrderCreatedPayload event) {
        DomainEvent<OrderCreatedPayload> envelope = DomainEvent.of(
            "order.created",
            event.orderId(),
            "Order",
            MDC.get("traceId"),
            event
        );
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_CREATED_RK,
            envelope
        );
        log.info("Event published: eventType={} aggregateId={} eventId={}",
            envelope.eventType(), envelope.aggregateId(), envelope.eventId());
    }
}
```

---

## 6. Consumer with Idempotency

```java
/**
 * Consumer for the {@code order.created} event in the payment service.
 * Implements idempotency via {@code ProcessedEventRepository} to prevent
 * reprocessing on retry or broker redelivery.
 */
@Slf4j
@Service
public class OrderCreatedConsumer {

    private final PaymentService paymentService;
    private final ProcessedEventRepository processedEventRepository;

    /**
     * @param paymentService           payment processing service
     * @param processedEventRepository repository of already-processed event IDs
     */
    public OrderCreatedConsumer(PaymentService paymentService,
                                ProcessedEventRepository processedEventRepository) {
        this.paymentService = paymentService;
        this.processedEventRepository = processedEventRepository;
    }

    /**
     * Processes the {@code order.created} event. Silently discards it if the
     * {@code eventId} has already been processed (idempotency guard).
     *
     * @param event event envelope received from the broker
     */
    @RabbitListener(queues = RabbitMQConfig.PAYMENT_QUEUE)
    public void onOrderCreated(DomainEvent<OrderCreatedPayload> event) {
        if (processedEventRepository.existsByEventId(event.eventId())) {
            log.warn("Duplicate event discarded: eventId={}", event.eventId());
            return;
        }
        log.info("Processing event: eventType={} aggregateId={}", event.eventType(), event.aggregateId());
        paymentService.processPayment(event.payload());
        processedEventRepository.markAsProcessed(event.eventId());
    }
}
```

---

## 7. application.yml Configuration

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USER:guest}
    password: ${RABBITMQ_PASSWORD:guest}
    virtual-host: /
    publisher-confirm-type: correlated
    publisher-returns: true
    listener:
      simple:
        acknowledge-mode: manual
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          multiplier: 2.0
          max-interval: 10000
        default-requeue-rejected: false
```

**Mandatory rules:**
- `acknowledge-mode: manual` — ACK only after successful processing
- `default-requeue-rejected: false` — failed messages go to DLQ, not back to the queue
- `publisher-confirm-type: correlated` — confirms delivery to the exchange

---

## 8. SAGA Flow — Choreography

```
OrderService          PaymentService        ContractService
     │                     │                     │
     │──order.created──────▶│                     │
     │                     │──payment.approved───▶│
     │                     │                     │──contract.generated
     │                     │                     │
     │◀─────────order.completed (via order.events)─┘
```

- Each service publishes a **success** or **failure** event after its step
- On failure, the service publishes a compensation event (`order.cancelled`, `payment.refunded`)
- **No central orchestrator** — each service only knows its next step

---

## 9. Per-Service Checklist

- [ ] Exchange, queue and DLQ declared explicitly in `RabbitMQConfig`
- [ ] `DomainEvent<T>` envelope used for all published events
- [ ] Unique UUID `eventId` per published message
- [ ] Consumer checks idempotency before processing
- [ ] `acknowledge-mode: manual` + `default-requeue-rejected: false`
- [ ] `publisher-confirm-type: correlated` enabled on publishers
- [ ] DLQ configured on all consumer queues
- [ ] Credentials via environment variables (never hardcoded)
- [ ] Logs include `eventType`, `aggregateId` and `eventId` on every publish/consume

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
