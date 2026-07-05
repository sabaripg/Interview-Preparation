# Part 12 — Messaging: Event-Driven (Kafka)

> The same event-driven pattern applied with Kafka instead of RabbitMQ. Interview Q&A at the end.

## Event-Driven Communication Between Spring Boot Services

```java
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;
    public void publish(OrderCreatedEvent event) {
        kafkaTemplate.send("order-topic", event);
    }
}
@Service
public class OrderEventConsumer {
    @KafkaListener(topics = "order-topic", groupId = "order-group")
    public void consume(OrderCreatedEvent event) { /* handle */ }
}
```

**What it does:** the same structural pattern regardless of broker — define an event class (a simple data carrier, a `record` is a natural fit), publish it via a template (`KafkaTemplate`/`RabbitTemplate`), consume it via a listener annotation (`@KafkaListener`/`@RabbitListener`, the latter covered extensively in Part 11).

**Why:** decoupling (producer doesn't know who consumes), independent scalability (add consumers without touching the producer), asynchronous, non-blocking communication between services.

**Choosing Kafka vs RabbitMQ:** Kafka for high-throughput, replayable event logs. RabbitMQ for simpler task-queue/routing-flexible messaging.

> ⚠️ **Pitfall:** don't present this as "new" territory if you've already covered RabbitMQ in depth — the strong answer explicitly says "same pattern I use for RabbitMQ, just swapping `RabbitTemplate`/`@RabbitListener` for `KafkaTemplate`/`@KafkaListener`," which signals the pattern is understood generically, not memorized per-broker.

---

## Interview Q&A

**Q: How would you implement event-driven communication between Spring Boot services?**
Covered above.
