# Topic 18: Kafka and Event Streaming

## Q126. Why Kafka over JMS/RabbitMQ?

### Concept
Kafka is a **distributed, durable, high-throughput event streaming platform**. JMS/RabbitMQ are traditional message brokers optimized for message routing and delivery to individual consumers.

### Simple Explanation
**RabbitMQ** = Post office — letter delivered once, disappears after delivery. Routes intelligently.  
**Kafka** = Immutable newspaper archive — message written permanently, anyone can read it anytime, seek to any date.

### Comparison Table

| Feature | Kafka | RabbitMQ / JMS |
|---|---|---|
| **Message retention** | Days/weeks (configurable) | Deleted after consumer ACKs |
| **Consumer model** | Pull — consumer controls pace | Push — broker pushes to consumer |
| **Throughput** | Millions messages/sec | Thousands messages/sec |
| **Replay** | ✅ Any consumer can re-read from offset | ❌ Once consumed, gone |
| **Consumer groups** | Multiple independent groups read same topic | One consumer "owns" each message |
| **Ordering** | Guaranteed within partition | Exchange-dependent |
| **Routing** | Simple topic/partition | Complex routing rules (exchanges, bindings) |
| **Scaling** | Horizontal (add partitions) | Limited horizontal scaling |
| **Use cases** | Event sourcing, audit, analytics, microservices | Task queues, RPC, point-to-point |

### When to Choose Each

```
Use Kafka when:
✅ High throughput (millions of events/sec — IoT, clickstream)
✅ Multiple teams consume same events (order created → billing, inventory, notifications)
✅ Replay needed (audit trail, event sourcing, debug)
✅ Long retention (events stored for days/months)
✅ Stream processing (Kafka Streams, real-time analytics)

Use RabbitMQ/JMS when:
✅ Complex routing logic (fan-out, topic exchange, routing keys)
✅ Simple task queues (email jobs, image processing jobs)
✅ RPC pattern (request/reply)
✅ Priority queues
✅ Small-scale messaging, lower throughput acceptable
```

### Interview Answer
> "I chose Kafka because our order events need to be consumed by multiple teams independently — billing, inventory, notifications, analytics. RabbitMQ would require copying the same message to multiple queues. With Kafka, each team is a separate consumer group reading the same topic at their own pace.
>
> Also, we needed replay capability — when we deployed a new analytics service, we could replay 90 days of events from Kafka. In RabbitMQ that data would be gone after first consumption.
>
> Trade-off: Kafka is overkill for simple task queues (send email, process image) — RabbitMQ's push model and complex routing are better there. Kafka's pull model means consumer controls pace — good for slow consumers processing heavy work."

---

## Additional Deep-Dive (Q126-Q135)

### Operational Checklist

- Partition key strategy is explicit and documented; ordering guarantees depend on it.
- Consumer lag alerts exist with thresholds by topic criticality.
- DLQ replay process is tested, not theoretical.
- Idempotency is implemented in consumers to tolerate reprocessing.

### Real Project Usage

- Teams often pair lag dashboards with deployment markers to quickly spot regressions introduced by new consumer versions.
- Most production incidents are not broker outages but consumer-side deserialization, schema drift, or poison message loops.

### Schema and Security Essentials

- Prefer schema-managed events (for example Avro + Schema Registry) to prevent producer/consumer contract drift.
- Enforce TLS and auth (SASL/SSL) for broker communication in production environments.

Interview point: event evolution strategy must be explicit (backward/forward compatibility rules), not ad-hoc.

## Q127. Kafka Internals — Producers, Topics, Partitions, Consumers, Groups

### Architecture Overview

```
Producer App                    Kafka Cluster (3 Brokers)                  Consumer Groups
                          ┌─────────────────────────────────────────────┐
         ─── Produce ───► │  Topic: "orders"  (3 partitions, RF=2)      │ ◄── Group A (Billing)
                          │                                               │     Consumer 1: P0
                          │  P0: [0,1,2,3,4,5]  → Broker 1 (Leader)     │     Consumer 2: P1
                          │  P1: [0,1,2,3]      → Broker 2 (Leader)     │     Consumer 3: P2
                          │  P2: [0,1,2,3,4]    → Broker 3 (Leader)     │
                          │                                               │ ◄── Group B (Analytics)
                          │  Replicas on other brokers                   │     Consumer 1: P0,P1,P2
                          └─────────────────────────────────────────────┘
```

### Topics & Partitions

```java
// Topic: logical grouping of events
// Partition: ordered, immutable log of messages
// Each message has: offset (position), key, value, timestamp, headers

// Partition assignment:
// No key     → Distributed across partitions (not strict per-message round-robin)
// With key   → hash(key) % numPartitions → Always same partition!
kafkaTemplate.send("orders", orderId.toString(), orderEvent);
// Same orderId → same partition → guaranteed ordering for that order
```

### Producers

```java
// Producer properties
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all          # Wait for leader + all replicas to confirm
      retries: 3
      batch-size: 16384  # Batch messages for efficiency
      linger-ms: 5       # Wait up to 5ms to fill batch

// acks=all: strongest guarantee — no message loss
// acks=1:   leader only — faster, small risk of loss on leader failure
// acks=0:   fire-and-forget — fastest, messages can be lost
```

### Consumers & Consumer Groups

```java
// Consumer group: Each partition assigned to ONE consumer in group
// If groupA has 3 consumers and 3 partitions:
//   Consumer 1 → Partition 0
//   Consumer 2 → Partition 1
//   Consumer 3 → Partition 2

// If groupA has 2 consumers and 3 partitions:
//   Consumer 1 → Partition 0, Partition 1
//   Consumer 2 → Partition 2

// If groupA has 4 consumers and 3 partitions:
//   Consumer 1 → Partition 0
//   Consumer 2 → Partition 1
//   Consumer 3 → Partition 2
//   Consumer 4 → IDLE (no partition to consume)

// Key insight: Max parallelism = number of partitions
// 6 partitions → max 6 parallel consumers per group
```

### Offset — The Position Tracker

```
Partition 0:   [msg0, msg1, msg2, msg3, msg4, msg5, msg6, msg7]
                                              ↑
                              Consumer committed offset = 4
                              (has processed 0,1,2,3 — will next read msg4)

Committed offset stored in __consumer_offsets topic
On consumer restart → resumes from offset 4
```

### Rebalancing

```
Event: New consumer joins group, or consumer crashes

Kafka Coordinator triggers rebalance:
1. All consumers pause
2. Partitions redistributed
3. Consumers resume from last committed offset

Result: Brief pause, no message loss (picks up where last committed)
```

### Retention Policy

```yaml
# Keep messages for 7 days regardless of consumption
log.retention.hours: 168

# Keep 50GB max per partition
log.retention.bytes: 52428800

# Compact: Keep only latest message per key (good for state)
log.cleanup.policy: compact  # or 'delete' (default)
```

### Interview Answer
> "Kafka topics are partitioned logs — each partition is an ordered, immutable sequence of messages. Producers write to topics, optionally with a partition key for ordering guarantee. With a key, same key always goes to same partition — all order events for orderID 123 are in order.
>
> Consumer groups: each partition assigned to exactly one consumer. Adds a consumer → rebalance, kafka redistributes. Max parallelism equals partition count — I size partitions upfront for expected scale. Offset is the consumer's position — committed after processing, so crashes resume from last committed.
>
> Follow-up: How do you ensure no message is processed twice? → Idempotent consumers, check processed status before acting."

---

## Q128. Kafka in Spring Boot — Configuration

### Producer Setup

```java
// application.yml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true      # Exactly-once delivery (no duplicates on retry)
        max.in.flight.requests.per.connection: 1  # Ordering with retries
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest    # Start from beginning if new group
      enable-auto-commit: false      # Manual commit after processing
      properties:
        spring.json.trusted.packages: "com.example.events"
```

### Kafka Config Bean

```java
@Configuration
public class KafkaConfig {
    
    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
                .partitions(6)       // Max 6 parallel consumers per group
                .replicas(3)         // Replication factor (fault tolerance)
                .config(TopicConfig.RETENTION_MS_CONFIG, "604800000")  // 7 days
                .build();
    }
    
    @Bean
    public NewTopic ordersDltTopic() {
        return TopicBuilder.name("orders.DLT")  // Dead letter topic
                .partitions(3)
                .replicas(3)
                .build();
    }
}
```

### Producer Service

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {
    
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
                .orderId(order.getId())
                .userId(order.getUserId())
                .total(order.getTotal())
                .timestamp(Instant.now())
                .build();
        
        // Partition key = orderId → ordering per order
        ListenableFuture<SendResult<String, OrderCreatedEvent>> future =
                kafkaTemplate.send("orders", order.getId().toString(), event);
        
        future.addCallback(
                result -> log.info("Published event offset={}", result.getRecordMetadata().offset()),
                ex -> log.error("Failed to publish order event: {}", order.getId(), ex)
        );
    }
}
```

### Consumer with Manual Commit

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    @Autowired
    private InventoryService inventoryService;
    
    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void onOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        
        log.info("Processing order={} partition={} offset={}", event.getOrderId(), partition, offset);
        
        try {
            inventoryService.reserveStock(event);
            ack.acknowledge();  // Manual commit — only after successful processing
        } catch (Exception e) {
            log.error("Failed to process order event: {}", event.getOrderId(), e);
            // Don't ack → message will be redelivered
            throw e;  // Let error handler (DLT or retry) take over
        }
    }
    
    // Batch listener for higher throughput
    @KafkaListener(topics = "audit-events", groupId = "audit-service", batch = "true")
    public void onAuditEvents(List<AuditEvent> events, Acknowledgment ack) {
        auditRepo.saveAll(events);  // Bulk insert
        ack.acknowledge();          // Commit entire batch
    }
}
```

### Error Handling & Retry Config

```java
@Configuration
public class KafkaListenerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.setConcurrency(3);  // 3 threads per listener → 3 partitions processed in parallel
        
        // Retry with exponential backoff
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate),  // Send to {topic}.DLT
                new FixedBackOff(1000L, 3)  // 3 retries, 1s apart
        );
        
        // Don't retry on these exceptions (re-queuing won't fix them)
        errorHandler.addNotRetryableExceptions(
                DeserializationException.class,
                IllegalArgumentException.class
        );
        
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

---

## Q129. Consumer Lag Mitigation

### What is Consumer Lag?
```
Producer writes offset 1000
Consumer committed offset 800
LAG = 1000 - 800 = 200 messages behind

Growing lag = consumer slower than producer → messages pile up in Kafka
```

### Diagnosis First

```bash
# Check lag per consumer group + partition
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-service

# Output:
GROUP           TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID
order-service   orders  0          800             1000             200    consumer-1
order-service   orders  1          500             1500             1000   consumer-2  ← HIGH LAG
order-service   orders  2          900             950              50     consumer-3
```

### Mitigation Strategies

```java
// Strategy 1: Increase consumer concurrency
@KafkaListener(topics = "orders", groupId = "order-service", concurrency = "6")
// Creates 6 consumer threads per listener — but max useful = number of partitions

// Strategy 2: Increase partitions (scale parallelism, with caveats)
// Before: 3 partitions, 3 max consumers
// After: 12 partitions, 12 max consumers (rebalance required)
// ⚠️ Adding partitions can change key→partition mapping for future records,
// so ordering guarantees are only per key within a partition and across time windows.
// Plan partition count early; increase only with operational coordination.
kafkaAdminClient.createPartitions(Map.of("orders", NewPartitions.increaseTo(12)));

// Strategy 3: Batch processing
@KafkaListener(topics = "orders", batch = "true")
public void processBatch(List<OrderEvent> events) {
    orderService.processBatch(events);  // Bulk operation faster than one-by-one
}
// application.yml
spring.kafka.consumer.max-poll-records: 500  // Read up to 500 at once

// Strategy 4: Optimize consumer logic (most impactful)
// Profile what's slow: DB calls, external API, complex computation
// Use async processing, batch DB inserts, caching

// Strategy 5: Horizontal scaling (add consumer instances)
// Deploy more pods in Kubernetes → each pod gets partitions assigned
// Rebalance happens automatically

// Strategy 6: Separate slow consumers to separate topics
// Fast consumers: notifications (< 1ms)
// Slow consumers: PDF generation (500ms)
// Don't mix — fast consumers lag behind slow ones
```

### Monitoring & Alerting

```yaml
# Prometheus alert: consumer lag > 10,000
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag > 10000
  for: 5m
  annotations:
    summary: "Consumer lag critical for {{ $labels.group }}/{{ $labels.topic }}"
    runbook: "Check consumer health, increase concurrency or add partitions"
```

### Interview Answer
> "Consumer lag means producer writes faster than consumers read. Diagnosis: check lag per partition with kafka-consumer-groups.sh. Common causes and fixes:
>
> If lag is uneven per partition → one consumer is slow/stuck — restart or replace it. If all partitions lagging → processing is just slow — optimize consumer logic first (batch DB inserts, async processing, caching).
>
> For sustained high volume: increase partitions (scales max parallelism), add consumer instances (Kubernetes scales pods), tune max.poll.records for batch processing. Long-term: profile what's slow in consumer — a 10× faster consumer is better than 10 more consumers."

---

## Q130. Poison Messages & DLQ Strategy

### What's a Poison Message?
A message the consumer can't process — bad format, invalid data, runtime error. Without handling, it blocks the partition indefinitely (consumer retries forever).

### Strategy 1: Dead Letter Topic (DLT)

```java
// Spring Kafka auto-configures DLT as {topic}.DLT
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
    DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                    (record, ex) -> new TopicPartition(
                            record.topic() + ".DLT",
                            record.partition()  // Same partition for ordering
                    ));
    
    // Retry 3 times with exponential backoff, then DLT
    ExponentialBackOff backOff = new ExponentialBackOff(1000, 2);
    backOff.setMaxElapsedTime(30000);  // Max 30s total
    
    return new DefaultErrorHandler(recoverer, backOff);
}
```

### Strategy 2: DLT Consumer — Alert + Manual Review

```java
@Component
public class DeadLetterConsumer {
    
    @KafkaListener(topics = "orders.DLT", groupId = "dlt-handler")
    public void handleDeadLetter(
            @Payload byte[] payload,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage,
            @Header(KafkaHeaders.ORIGINAL_TOPIC) String originalTopic) {
        
        log.error("DLT message from topic={}, reason={}", originalTopic, exceptionMessage);
        
        // 1. Save to database for review
        DeadLetterRecord record = DeadLetterRecord.builder()
                .payload(new String(payload, StandardCharsets.UTF_8))
                .originalTopic(originalTopic)
                .exception(exceptionMessage)
                .receivedAt(Instant.now())
                .status(DLTStatus.PENDING_REVIEW)
                .build();
        dltRepository.save(record);
        
        // 2. Alert on-call
        alertService.sendAlert("Poison message in " + originalTopic, exceptionMessage);
    }
}
```

### Strategy 3: Reprocessing DLT Messages

```java
// Admin endpoint to replay DLT messages after the bug is fixed
@RestController
@RequestMapping("/admin/dlt")
public class DltReprocessController {
    
    @Autowired
    private DltRepository dltRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @PostMapping("/reprocess/{id}")
    public ResponseEntity<String> reprocess(@PathVariable Long id) {
        DeadLetterRecord record = dltRepository.findById(id).orElseThrow();
        
        // Replay to original topic (bug is now fixed)
        kafkaTemplate.send(record.getOriginalTopic(), record.getPayload());
        record.setStatus(DLTStatus.REPROCESSED);
        dltRepository.save(record);
        
        return ResponseEntity.ok("Requeued message " + id);
    }
    
    @PostMapping("/reprocess/all")
    public ResponseEntity<String> reprocessAll() {
        dltRepository.findByStatus(DLTStatus.PENDING_REVIEW)
                .forEach(r -> kafkaTemplate.send(r.getOriginalTopic(), r.getPayload()));
        return ResponseEntity.ok("Reprocessed all pending DLT messages");
    }
}
```

### Don't-Retry Exceptions

```java
// Some exceptions won't succeed on retry — send straight to DLT
errorHandler.addNotRetryableExceptions(
    DeserializationException.class,    // Bad JSON — retrying won't fix
    IllegalArgumentException.class,    // Invalid business data — retrying won't fix
    DataIntegrityViolationException.class  // DB constraint — won't fix
);

// DO retry:
// - Network timeout (transient)
// - DB connection refused (transient)
// - External API 503 (transient)
```

### Interview Answer
> "Poison messages block partition if retried forever — DLT is the safety valve. Configure retry with exponential backoff (1s, 2s, 4s...) then route to {topic}.DLT. DLT consumer saves to database for investigation and alerts on-call.
>
> Don't retry deserialize errors or business validation errors — they'll never succeed. Do retry network timeouts and transient DB errors. After fixing the bug, replay DLT messages via admin endpoint.
>
> Risk: If DLT consumer is also down, messages queue in DLT topic — still no data loss, process them after recovery."

---

## Q131–Q133. Feature Flags, SLO/SLA, Rollback Triggers

### Feature Flags

```java
// Library: LaunchDarkly, Unleash, or simple DB/config-based
@Service
public class PaymentService {
    
    @Autowired
    private FeatureFlagClient featureFlags;
    
    public PaymentResult processPayment(PaymentRequest request) {
        // Gradual rollout: enable for % of users
        if (featureFlags.isEnabled("new-payment-flow", request.getUserId())) {
            return newPaymentFlow.process(request);
        } else {
            return legacyPaymentFlow.process(request);
        }
    }
}

// Flag configurations:
// Stage 1: 0% → deploy, no traffic
// Stage 2: 5% → canary (monitor error rate, latency)
// Stage 3: 25% → wider testing
// Stage 4: 100% → full rollout
// Disable: 0% → instant rollback (no redeploy needed)
```

### Rollout Thresholds

```yaml
# Automatic flag disable triggers:
- error_rate > 1%       → disable flag, alert team
- p99_latency > 2s      → disable flag, investigate
- 5xx_rate > 0.5%       → disable flag immediately
- conversion_rate drops 10%  → disable, A/B analysis
```

### SLO vs SLA

| | SLA | SLO |
|---|---|---|
| **Stands for** | Service Level Agreement | Service Level Objective |
| **What it is** | Contract with customer | Internal target |
| **Consequence** | Financial penalty if breached | Engineering action |
| **Example** | "99.9% uptime or refund 10% bill" | "Target 99.95% uptime" |

```
SLO is more aggressive than SLA to create buffer:
SLA: 99.9% uptime (allowed 8.7 hours downtime/year)
SLO: 99.95% uptime (4.4 hours downtime/year)
→ SLO breach alerts team before SLA breach costs money

Error Budget = 100% - SLO
If SLO = 99.9% → Error budget = 0.1% per month
= 43.8 minutes of downtime per month
```

### Error Budget in Practice

```
Month starts: 43 min error budget
Week 1: Incident → used 15 min
Week 2: Deploy issue → used 10 min
Week 3: Budget = 43 - 15 - 10 = 18 min left

If budget nearly exhausted:
→ Freeze deploys
→ Focus on reliability over features
→ SRE principle: burn rate too high
```

### Rollback Triggers in Production

```yaml
# Automated rollback triggers (Kubernetes/ArgoCD/Spinnaker):
error_rate: > 1% for 5 minutes
http_5xx_rate: > 0.5% for 3 minutes
p99_latency: > 2000ms for 5 minutes
health_check_failures: > 2 consecutive
memory_usage: > 90% for 10 minutes

Manual triggers:
- Business metric drops (conversion rate, revenue)
- User complaints spike
- Data inconsistency detected
```

---

## Q134. Dashboards & Monitoring (Datadog, Actuator, Splunk)

### Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,prometheus,loggers,env,threaddump,heapdump
  endpoint:
    health:
      show-details: always   # Show DB, Kafka, Redis health
  metrics:
    export:
      prometheus:
        enabled: true         # Expose /actuator/prometheus for Prometheus scraping

# Key endpoints:
# GET /actuator/health      → {"status":"UP", "components":{"db":...,"kafka":...}}
# GET /actuator/metrics     → Available metrics list
# GET /actuator/metrics/jvm.memory.used → JVM memory
# GET /actuator/prometheus  → All metrics in Prometheus format
# GET /actuator/threaddump  → Thread stack traces
# POST /actuator/loggers/com.example → Change log level at runtime
```

### Custom Business Metrics with Micrometer

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    private final Gauge activeOrdersGauge;
    
    @PostConstruct
    void initMetrics() {
        Counter.builder("orders.created")
               .description("Total orders created")
               .tag("env", "production")
               .register(meterRegistry);
        
        Timer.builder("orders.processing.duration")
             .description("Order processing time")
             .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
             .register(meterRegistry);
    }
    
    public Order createOrder(CreateOrderRequest req) {
        Timer.Sample timer = Timer.start(meterRegistry);
        
        try {
            Order order = doCreateOrder(req);
            meterRegistry.counter("orders.created", "status", "success").increment();
            return order;
        } catch (Exception e) {
            meterRegistry.counter("orders.created", "status", "failure").increment();
            throw e;
        } finally {
            timer.stop(meterRegistry.timer("orders.processing.duration"));
        }
    }
}
```

### Golden Signals to Monitor (Google SRE)

```
1. Latency — p50, p95, p99 response times
   Alert: p99 > 2s for 5 minutes

2. Traffic — requests/second per endpoint
   Alert: drops 50% (circuit breaker? error?) or spikes 10× (attack?)

3. Errors — 4xx, 5xx rate
   Alert: 5xx > 0.5%

4. Saturation — CPU, memory, thread pool, DB connection pool
   Alert: > 80% sustained
```

### Structured Logging for Splunk/ELK

```java
// MDC: Attach correlation IDs to all log lines
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws Exception {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-ID"))
                                       .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId);
        MDC.put("userId", extractUserId(request));
        MDC.put("requestId", UUID.randomUUID().toString());
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();  // Remove after request
        }
    }
}

// JSON log output (Logback with logstash-logback-encoder):
// {"timestamp":"2026-03-14T10:00:00Z","level":"INFO","logger":"OrderService",
//  "message":"Order created","correlationId":"abc-123","userId":"456","orderId":"789"}
```

### Splunk Dashboard Queries

```
# Error rate by endpoint
index=prod source=app level=ERROR
| stats count by uri, status_code
| sort -count

# Slow requests
index=prod source=app duration_ms > 2000
| stats avg(duration_ms) p99(duration_ms) by uri

# User activity
index=prod source=app userId=12345
| table timestamp, uri, method, status_code, duration_ms
```

---

## Q135. Secrets Management (AWS Secrets Manager / Parameter Store)

### Never Store Secrets in Properties Files

```properties
# ❌ NEVER — committed to Git, visible to everyone
spring.datasource.password=super-secret-password
api.key=sk-1234567890abcdef
```

### AWS Secrets Manager

```java
// application.yml — reference secrets by ARN/name
spring:
  config:
    import: aws-secretsmanager:/myapp/prod/db-credentials;/myapp/prod/api-keys

# Secret stored in AWS: myapp/prod/db-credentials
# Value (JSON): {"username":"app_user","password":"real-password","host":"prod.db.example.com"}
```

```java
// Or programmatic access
@Service
public class SecretsService {
    private final SecretsManagerClient secretsClient;
    
    @Cacheable(value = "secrets", key = "#secretName")  // Cache to avoid throttling
    public Map<String, String> getSecret(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();
        GetSecretValueResponse response = secretsClient.getSecretValue(request);
        return objectMapper.readValue(response.secretString(), Map.class);
    }
}
```

### AWS Parameter Store (Simpler, Cheaper)

```yaml
# application.yml
spring:
  config:
    import: aws-parameterstore:/myapp/prod/

# Parameters in SSM:
# /myapp/prod/spring.datasource.url    → jdbc:postgresql://...
# /myapp/prod/spring.datasource.password → ***
# Spring auto-maps parameter names to properties
```

### Kubernetes Secrets

```yaml
# Create secret
kubectl create secret generic app-secrets \
  --from-literal=db-password='secret123' \
  --from-literal=api-key='sk-abc123'

# Mount as env variables in deployment
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
```

### HashiCorp Vault (Enterprise)

```java
// Spring Vault integration
spring:
  cloud:
    vault:
      host: vault.company.com
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        application-name: order-service
```

---

