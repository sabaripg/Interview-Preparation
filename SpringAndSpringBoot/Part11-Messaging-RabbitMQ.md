# 🐰 Part 11 — Messaging: RabbitMQ

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 📬 Basic Producer-Consumer Flow

```java
@Bean
public Queue queue() { return new Queue("my-queue", true); }
@Bean
public DirectExchange exchange() { return new DirectExchange("my-exchange"); }
@Bean
public Binding binding(Queue queue, DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with("my-routing-key");
}

// Producer
@Autowired private RabbitTemplate rabbitTemplate;
public void sendMessage(String message) {
    rabbitTemplate.convertAndSend("my-exchange", "my-routing-key", message);
}

// Consumer
@RabbitListener(queues = "my-queue")
public void receiveMessage(String message) { System.out.println("Received: " + message); }
```
- Producers publish to an **exchange**, which routes to bound queue(s) via the routing key and exchange type.
- `@RabbitListener` registers a message handler — Spring manages the listener container for you.
- `Queue`, `Exchange`, `Binding` are declared as beans, auto-declared against the broker on startup.

> [!IMPORTANT]
> Producers publish to an **exchange**, never directly to a queue — the exchange+routing-key+binding is the actual routing mechanism.

---

## 📦 Sending Java Objects (Not Just Plain Text)

```java
@Bean
public MessageConverter jsonMessageConverter() { return new Jackson2JsonMessageConverter(); }
```
- Registering a `MessageConverter` (`Jackson2JsonMessageConverter`) on the `RabbitTemplate` and listener factory auto-serializes/deserializes objects to/from JSON.

> [!WARNING]
> Without a converter, `RabbitTemplate` falls back to Java serialization — fragile and not interoperable with non-Java consumers. Explicitly configuring JSON should be the standard, not an opt-in.

---

## ✅ Manual Acknowledgment — Surviving a Consumer Crash

```java
@RabbitListener(queues = "my-queue", ackMode = "MANUAL")
public void receiveMessage(Message message, Channel channel) throws IOException {
    try {
        // business logic
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
    }
}
```

| Ack mode | Behavior on crash mid-processing |
|---|---|
| **Auto** (default) | Message marked delivered the moment it's sent, before your handler runs → crash = message **lost** |
| **Manual** | Message stays unacked until you explicitly ack → crash = RabbitMQ **redelivers** it |

> [!CAUTION]
> Auto-ack is the default and a **silent data-loss trap**. Manual ack (only after successful processing) is correct for anything where losing a message is unacceptable.

---

## ☠️ Dead Letter Queues

```java
@Bean
public Queue queue() {
    return QueueBuilder.durable("my-queue")
        .withArgument("x-dead-letter-exchange", "dlx-exchange")
        .withArgument("x-dead-letter-routing-key", "dlx-routing-key")
        .build();
}
```
- RabbitMQ automatically routes a message to the DLQ when it's rejected (`basicNack`/`basicReject` with `requeue=false`), exceeds a TTL, or the queue hits a length limit.

> [!WARNING]
> A DLQ with no consumer/alerting is a common real gap — messages silently pile up until a customer complains. Always pair a DLQ with monitoring on its depth.

---

## 🔢 Guaranteeing Message Order

- RabbitMQ preserves order **within a single queue delivered to a single consumer** — concurrent consumers break ordering across messages.

| To guarantee order | How |
|---|---|
| Strict, low-throughput | Single consumer instance, no concurrency |
| High-throughput, per-entity order | **Consistent hashing** (`x-consistent-hash` exchange) keyed on entity ID |

> [!IMPORTANT]
> A sequence number alone doesn't guarantee ordered *processing* with concurrent consumers — it only lets you *detect* misordering after the fact.

---

## 📈 Scaling Consumers for High Traffic

```java
@RabbitListener(queues = "my-queue", concurrency = "5-10")
public void receiveMessage(String message) { ... }
```
- `concurrency = "min-max"` scales concurrent consumer threads within one instance.
- Beyond that: deploy multiple instances (competing-consumers model), or shard/cluster the queue at the broker level for very high throughput.

> [!WARNING]
> Bumping `concurrency` doesn't help if the real bottleneck is a downstream dependency (DB, external API) — check what the handler is actually bottlenecked on first.

---

## ⏳ Delaying Message Processing

```java
@Bean
public Queue delayedQueue() {
    return QueueBuilder.durable("delayed-queue")
        .withArgument("x-message-ttl", 10000)
        .withArgument("x-dead-letter-exchange", "main-exchange")
        .withArgument("x-dead-letter-routing-key", "main-routing-key")
        .build();
}
```
- **TTL + DLX trick:** publish to a "holding" queue with a fixed TTL and no consumer — once it expires, the message dead-letters into the real target queue.
- **Alternative:** the RabbitMQ Delayed Message Exchange plugin — supports arbitrary per-message delays, but requires a community plugin.

> [!WARNING]
> TTL+DLX only supports a **single fixed delay per queue** — different delays per message need multiple TTL-tiered queues or the delayed-message plugin.

---

## 📏 Large Message Payloads

- Store the large payload externally (S3, DB, Redis) and publish only a lightweight **reference** (ID or URL) through RabbitMQ.

> [!CAUTION]
> Sending large binary blobs directly through RabbitMQ degrades broker performance for **all** queues sharing that broker's resources, not just the one carrying large messages.

---

## 1️⃣ Exactly-Once Processing

```java
@RabbitListener(queues = "my-queue")
public void processMessage(String messageId, String content) {
    if (redisTemplate.hasKey(messageId)) return; // already processed
    // process
    redisTemplate.opsForValue().set(messageId, "processed", 10, TimeUnit.MINUTES);
}
```
- RabbitMQ only guarantees **at-least-once** delivery by default.
- Fix: idempotency at the consumer — dedup store (Redis/DB) keyed on a unique message ID.

> [!CAUTION]
> Don't claim RabbitMQ provides exactly-once delivery natively. The correct framing: "at-least-once delivery + idempotent consumer = effectively-once processing."

---

## 🔄 Request-Response (RPC) Pattern

```java
public String sendRpcRequest(String message) {
    String correlationId = UUID.randomUUID().toString();
    MessageProperties properties = new MessageProperties();
    properties.setCorrelationId(correlationId);
    properties.setReplyTo("reply-queue");
    Message requestMessage = new Message(message.getBytes(), properties);
    rabbitTemplate.convertAndSend("rpc-exchange", "rpc-routing-key", requestMessage);
    Message responseMessage = rabbitTemplate.receive("reply-queue", 5000);
    return responseMessage != null ? new String(responseMessage.getBody()) : "No response";
}
```
- Requester sets `correlationId` + `replyTo`, blocks waiting for a response (with timeout).
- Responder copies the `correlationId` onto its reply, publishes to `replyTo`.

> [!WARNING]
> Blocking the calling thread for a response is a real scalability concern at high volume — a tradeoff, not a free upgrade over REST/gRPC.

---

## ⭐ Prioritizing Messages

```java
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable("priority-queue").withArgument("x-max-priority", 10).build();
}
```
- `x-max-priority` defines the range; producers set `MessageProperties.setPriority(n)` per message.

> [!NOTE]
> Priority only matters when there's an actual backlog to reorder — steady-state throughput rarely benefits.

---

## 🔁 Retry Before Moving to a DLQ

```java
@RabbitListener(queues = "my-queue")
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 2000))
public void processMessage(String message) {
    if (message.contains("fail")) throw new RuntimeException("Processing failed");
}
```
- **Spring Retry** retries in-process with backoff before letting the exception propagate to the normal nack/DLQ path.
- **Alternative** for restart-surviving delay retry: a short-TTL "retry queue" that dead-letters back into the main queue, incrementing a retry-count header.

> [!CAUTION]
> Spring Retry's state is entirely **in-memory within one delivery attempt** — a consumer crash mid-retry loses that state; RabbitMQ's own redelivery takes over instead. Two different mechanisms at different layers.

---

## 📦 Batch Processing for Throughput

```java
@Bean
public SimpleRabbitListenerContainerFactory batchFactory(ConnectionFactory cf) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(cf);
    factory.setBatchSize(10);
    factory.setConsumerBatchEnabled(true);
    factory.setDeBatchingEnabled(true);
    return factory;
}
```
- Groups multiple deliveries into one `List<T>` handed to the listener — amortizes per-message overhead, at the cost of added latency.

> [!WARNING]
> Batching changes failure handling too — one message failing affects the whole batch's ack/retry unless you build per-item error handling in.

---

## 🔐 Securing RabbitMQ Communication

```yaml
spring:
  rabbitmq:
    port: 5671
    ssl:
      enabled: true
      trust-store: classpath:truststore.jks
```
- **Transport:** SSL/TLS (port 5671 AMQPS instead of 5672).
- **Auth:** real credentials, never default guest outside local dev.
- **Payload encryption:** for genuinely sensitive content, defense in depth.

> [!CAUTION]
> Relying on network-level security (VPC isolation) alone without AMQP-level TLS fails compliance audits.

---

## 📢 Broadcasting to Multiple Independent Consumers

```java
@Bean
public FanoutExchange fanoutExchange() { return new FanoutExchange("fanout-exchange"); }
```
- A **Fanout Exchange** ignores routing keys entirely, delivers to **every** bound queue — the pub-sub/broadcast mechanism.

> [!IMPORTANT]
> Contrast with Direct/Topic exchanges (routing-key-based, one message → specific matching queue(s)) — don't conflate all exchange types as "basically the same."

---

## 📊 Monitoring Queue Performance in Production

| Tool | What it gives you |
|---|---|
| RabbitMQ Management Plugin | Built-in dashboard — queue depth, message rates, consumer counts |
| Prometheus plugin | Broker metrics scraped into Grafana |

**Key metrics to alert on:** queue depth (backlog building), consumer count (dropping to zero), publish/ack rate gap (earliest backlog signal).

> [!TIP]
> Just enabling a dashboard isn't the full answer — naming *which* metrics matter for alerting is the senior-level detail.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Basic producer-consumer flow? | Producers → exchange → routing key → queue; @RabbitListener consumes |
| Sending Java objects, not just text? | Jackson2JsonMessageConverter bean |
| Consumer crashes mid-processing? | Manual ack + redelivery; auto-ack loses the message |
| Handling repeatedly-failing messages? | Dead Letter Queue via x-dead-letter-exchange |
| Guaranteeing message order? | Single consumer, or consistent-hash exchange for per-entity order |
| Scaling consumers for high traffic? | concurrency="min-max", multiple instances, sharding |
| Delaying message processing? | TTL + DLX trick, or delayed-message plugin |
| Large message payloads? | Store externally, publish a reference |
| Exactly-once processing? | At-least-once + idempotent consumer |
| RPC pattern over RabbitMQ? | correlationId + replyTo, blocking receive with timeout |
| Prioritizing messages? | x-max-priority queue argument |
| Retry before DLQ? | @Retryable (in-memory) or TTL-based retry queue (survives restarts) |
| Batch processing for throughput? | SimpleRabbitListenerContainerFactory with batching enabled |
| Securing RabbitMQ communication? | TLS on the AMQP connection, real credentials |
| Broadcasting to multiple consumers? | Fanout exchange |
| Monitoring in production? | Management plugin / Prometheus — alert on queue depth & consumer count |
