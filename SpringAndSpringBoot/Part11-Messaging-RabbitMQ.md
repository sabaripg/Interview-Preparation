# Part 11 — Messaging: RabbitMQ

> Producer-consumer flow, object serialization, acknowledgment, dead-letter queues, ordering, scaling, delay, large payloads, exactly-once, RPC, priority, retry, batching, security, fanout, monitoring. Interview Q&A at the end.

## Basic Producer-Consumer Flow

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
**What's happening:** the producer publishes to an **exchange**, which routes to bound queue(s) based on the routing key and exchange type. `@RabbitListener(queues = "...")` registers a method as a message handler — Spring manages the underlying `SimpleMessageListenerContainer` polling/consuming for you. Declare `Queue`, `Exchange`, and `Binding` as beans — Spring AMQP auto-declares them against the broker on startup if they don't already exist.

> ⚠️ **Pitfall:** producers publish to an **exchange**, never directly to a queue. Thinking of `rabbitTemplate.send(queueName, message)` as the primary model misses that the exchange+routing-key+binding is the actual routing mechanism, not the queue name itself.

## Sending Java Objects (Not Just Plain Text)

```java
@Bean
public MessageConverter jsonMessageConverter() { return new Jackson2JsonMessageConverter(); }
```
**What it does:** registering a `MessageConverter` bean (typically `Jackson2JsonMessageConverter`) on the `RabbitTemplate` and listener container factory makes Spring AMQP automatically serialize outgoing objects to JSON and deserialize incoming JSON back into the target Java type.

> ⚠️ **Pitfall:** without a converter, `RabbitTemplate` falls back to Java serialization (`SimpleMessageConverter`) by default — fragile (requires `Serializable`, ties both ends to the same class definitions) and not interoperable with non-Java consumers. Explicitly configuring `Jackson2JsonMessageConverter` should be treated as the standard, not an opt-in.

## Manual Acknowledgment — Surviving a Consumer Crash Mid-Processing

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
**What it does:** with **manual acknowledgment**, a message stays "unacked" until you explicitly `basicAck` — if the consumer crashes/disconnects before acking, RabbitMQ detects the dropped connection and automatically redelivers the message to another available consumer.

> ⚠️ **Pitfall:** with the default **auto acknowledgment**, RabbitMQ marks the message as delivered/removed the moment it's sent over the wire, *before* your handler even runs — a crash mid-processing means the message is simply lost, no redelivery. Auto-ack is the default and a silent data-loss trap; manual ack (only after successful processing) is correct for anything where losing a message is unacceptable.

## Dead Letter Queues — Handling Repeatedly-Failing Messages

```java
@Bean
public Queue queue() {
    return QueueBuilder.durable("my-queue")
        .withArgument("x-dead-letter-exchange", "dlx-exchange")
        .withArgument("x-dead-letter-routing-key", "dlx-routing-key")
        .build();
}
```
**What it does:** configure the primary queue with `x-dead-letter-exchange`/`x-dead-letter-routing-key` arguments — RabbitMQ automatically routes a message there when it's rejected (`basicNack`/`basicReject` with `requeue=false`), exceeds a TTL, or the queue hits a length limit. The DLQ is bound to that exchange like a normal queue, with its own consumer for alerting/manual inspection/eventual reprocessing.

> ⚠️ **Pitfall:** a DLQ with no consumer/alerting is a common real gap — messages silently pile up there with nobody noticing until a customer complains. Always pair a DLQ with monitoring on its depth.

## Guaranteeing Message Order

**Why order isn't automatic:** RabbitMQ preserves order **within a single queue delivered to a single consumer** — the moment you add concurrent consumers (or `concurrency > 1`), ordering across messages is no longer guaranteed.

**To guarantee strict ordering:** use a **single consumer instance** for that queue, optionally with a sequence number in the payload so the consumer can detect out-of-sequence arrivals defensively.

**For higher-throughput ordered processing:** use **consistent hashing** (the `x-consistent-hash` exchange type) keyed on an entity ID (e.g. order ID) — guarantees all messages for the *same* entity always land on the *same* queue/consumer, preserving per-entity order while still allowing parallelism across different entities.

> ⚠️ **Pitfall:** "just add a sequence number" alone doesn't guarantee ordered *processing* with multiple concurrent consumers — it only lets you *detect* misordering after the fact. The actual fix is constraining concurrency or partitioning by key.

## Scaling Consumers for High Traffic

```java
@RabbitListener(queues = "my-queue", concurrency = "5-10")
public void receiveMessage(String message) { ... }
```
`concurrency = "min-max"` lets a single listener container spin up multiple concurrent consumer threads, elastically scaling within the bounds. Beyond a single instance, deploy **multiple instances** — RabbitMQ's competing-consumers model delivers each queue message to exactly one consumer across all connected instances. For very high throughput, consider queue sharding/clustering at the broker level.

> ⚠️ **Pitfall:** bumping `concurrency` alone doesn't help if the bottleneck is actually the downstream dependency (DB, external API) each handler calls — scaling consumer threads just shifts contention downstream.

## Delaying Message Processing

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
**TTL + DLX trick:** publish to a "holding" queue with a fixed `x-message-ttl` and no active consumer — once the TTL expires, the message is dead-lettered into the *real* target queue, effectively delaying delivery.

**Alternative:** the **RabbitMQ Delayed Message Exchange plugin** — a purpose-built exchange type supporting a per-message delay header directly, more flexible but requires a community plugin.

> ⚠️ **Pitfall:** the TTL+DLX approach only supports a **single fixed delay per queue** — for different delays per message, you need multiple TTL-tiered queues or the delayed-message plugin.

## Large Message Payloads

**What to do:** store the large payload externally (S3, database, Redis) and publish only a lightweight **reference** (an ID or URL) through RabbitMQ — the consumer fetches the actual payload using that reference.

> ⚠️ **Pitfall:** sending large binary blobs directly through RabbitMQ degrades broker performance for *all* queues sharing that broker's resources, not just the one carrying large messages — a shared-infrastructure concern, not just per-message efficiency.

## Exactly-Once Processing

```java
@RabbitListener(queues = "my-queue")
public void processMessage(String messageId, String content) {
    if (redisTemplate.hasKey(messageId)) return; // already processed
    // process
    redisTemplate.opsForValue().set(messageId, "processed", 10, TimeUnit.MINUTES);
}
```
**Why it's not free:** RabbitMQ (like virtually all brokers) only guarantees **at-least-once** delivery by default — redelivery after a nack/crash/requeue can cause a consumer to see the same message more than once.

**The standard fix:** idempotency at the consumer — tag each message with a unique ID, check a dedup store (Redis/DB) before processing — turning "at-least-once delivery" into "effectively-once processing" at the application layer.

> ⚠️ **Pitfall:** don't claim RabbitMQ provides exactly-once delivery natively — the correct framing is "at-least-once delivery + idempotent consumer = effectively-once processing."

## Request-Response (RPC) Pattern

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
The requester sets `correlationId` (to match a response to its request) and `replyTo` on the outgoing message, then blocks waiting for a response (with timeout). The responder copies the same `correlationId` onto its reply and publishes to the `replyTo` queue.

> ⚠️ **Pitfall:** `rabbitTemplate.receive(queue, timeout)` blocking the calling thread is a real scalability concern at high volume — worth naming as a tradeoff, not a free upgrade over REST/gRPC.

## Prioritizing Messages

```java
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable("priority-queue").withArgument("x-max-priority", 10).build();
}
// Producer sets MessageProperties.setPriority(priority) per message
```
Declare the queue with `x-max-priority` — RabbitMQ delivers higher-priority messages ahead of lower-priority ones **when there's a backlog** (not a hard real-time guarantee).

> ⚠️ **Pitfall:** priority queues only matter when there's an actual backlog — during steady-state operation with consumers keeping up, there's rarely a queue depth for priority to meaningfully affect.

## Retry Before Moving to a DLQ

```java
@RabbitListener(queues = "my-queue")
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 2000))
public void processMessage(String message) {
    if (message.contains("fail")) throw new RuntimeException("Processing failed");
}
```
**Spring Retry** (`@Retryable`) retries the handler in-process a bounded number of times with configurable backoff, before letting the exception propagate (triggering the normal nack/DLQ path) once attempts are exhausted.

**Alternative for delay-based retry that survives restarts:** reject into a short-TTL "retry queue" which dead-letters back into the main queue after the delay, incrementing a retry-count header each pass.

> ⚠️ **Pitfall:** Spring Retry's retries happen entirely **in-memory within a single delivery attempt** — if the consumer process crashes mid-retry, that state is lost and RabbitMQ's own redelivery takes over instead. These are two different retry mechanisms at different layers, not one unified system.

## Batch Processing for Throughput

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

@RabbitListener(queues = "batch-queue", containerFactory = "batchFactory")
public void processBatchMessages(List<String> messages) { ... }
```
Groups multiple deliveries into a single `List<T>` handed to the listener, amortizing per-message overhead. Trade-off: batching adds latency in exchange for throughput.

> ⚠️ **Pitfall:** batching changes failure-handling too — if one message in a batch fails, ack/retry semantics apply to the whole batch by default unless you build per-item error handling into the batch handler.

## Securing RabbitMQ Communication

```yaml
spring:
  rabbitmq:
    host: your-rabbitmq-host
    port: 5671
    username: user
    password: pass
    ssl:
      enabled: true
      trust-store: classpath:truststore.jks
      trust-store-password: yourpassword
```
- **Transport security** — SSL/TLS on the connection (port 5671 AMQPS instead of 5672 plaintext).
- **Authentication** — real credentials (never default guest outside local dev); production integration with LDAP/OAuth2 plugins.
- **Payload-level encryption** — for genuinely sensitive content, encrypt before publishing (defense in depth).

> ⚠️ **Pitfall:** relying on network-level security (VPC isolation) alone without AMQP-level TLS fails compliance audits and leaves traffic exposed to anything with network access inside that boundary.

## Broadcasting to Multiple Independent Consumers

```java
@Bean
public FanoutExchange fanoutExchange() { return new FanoutExchange("fanout-exchange"); }
// bind queue-1 and queue-2 to the fanout exchange, each with its own consumer
rabbitTemplate.convertAndSend("fanout-exchange", "", "Broadcast Message");
```
**What it does:** a **Fanout Exchange** ignores routing keys entirely and delivers every published message to **every** queue bound to it — the pub-sub/broadcast mechanism.

> ⚠️ **Pitfall:** contrast explicitly with Direct/Topic exchanges (routing-key-based, message goes to specific matching queue(s) only) — don't conflate all exchange types as "basically the same."

## Monitoring Queue Performance in Production

- **RabbitMQ Management Plugin** — built-in dashboard for queue depth, message rates, consumer counts, connection/channel stats.
- **Prometheus plugin** — exposes broker metrics, scraped by Prometheus and visualized in Grafana.
- **Key metrics to actually alert on:** queue depth (backlog building up), consumer count (unexpectedly dropping to zero), publish/ack rate gap (earliest signal of a growing backlog).

> ⚠️ **Pitfall:** just enabling the dashboard/exporter isn't the full answer — naming *which* metrics matter for alerting is the senior-level detail.

---

## Interview Q&A

**Q: How do you implement a basic producer-consumer flow with RabbitMQ in Spring Boot?**
Covered above.

**Q: How do you send and receive Java objects (not just plain text) over RabbitMQ?**
Covered above.

**Q: A consumer crashes after receiving a message but before finishing processing it — what happens, and how do you guarantee reprocessing?**
Covered above.

**Q: How do you handle messages that repeatedly fail to process (Dead Letter Queues)?**
Covered above.

**Q: How do you guarantee consumers process messages in a specific order?**
Covered above.

**Q: Your application has high message traffic — how do you scale consumers to keep up?**
Covered above.

**Q: How can you delay processing of a message?**
Covered above.

**Q: A message payload is too large to handle efficiently — what do you do?**
Covered above.

**Q: How do you ensure a message is processed exactly once, even if RabbitMQ retries delivery?**
Covered above.

**Q: How would you implement a request-response (RPC) pattern using RabbitMQ?**
Covered above.

**Q: How can you prioritize urgent messages over normal ones?**
Covered above.

**Q: A consumer fails to process a message — how do you retry before giving up and moving it to a DLQ?**
Covered above.

**Q: How can you process messages in batches for better throughput?**
Covered above.

**Q: How do you secure RabbitMQ communication in a Spring Boot application?**
Covered above.

**Q: How do you broadcast a message to multiple independent consumers?**
Covered above.

**Q: How do you monitor RabbitMQ queue performance and message rates in production?**
Covered above.
