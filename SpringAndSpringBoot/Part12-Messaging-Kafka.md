# 📨 Part 12 — Messaging: Event-Driven (Kafka)

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🔗 Event-Driven Communication Between Spring Boot Services

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

- Same structural pattern regardless of broker: event class → publish via a template → consume via a listener annotation.

| Broker | Template | Listener |
|---|---|---|
| RabbitMQ | `RabbitTemplate` | `@RabbitListener` |
| Kafka | `KafkaTemplate` | `@KafkaListener` |

**Why this pattern:** decoupling (producer doesn't know who consumes), independent scalability, asynchronous non-blocking communication.

**Kafka vs RabbitMQ:** Kafka for high-throughput, replayable event logs. RabbitMQ for simpler task-queue/routing-flexible messaging. (See the full Kafka guide for the deep dive.)

> [!TIP]
> Don't present this as "new" territory if you've covered RabbitMQ in depth — the strong answer explicitly says "same pattern I use for RabbitMQ, just swapping the template/listener pair," signaling the pattern is understood generically.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Event-driven communication between services? | Same publish/consume pattern as RabbitMQ, via KafkaTemplate/@KafkaListener |
