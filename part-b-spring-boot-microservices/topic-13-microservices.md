# Topic 13: Microservices Architecture & Communication

## Question Bank Coverage Mapping (Q63-Q75)

| Question Bank Q# | Section In This File |
|---|---|
| Q63 | 1. Monolith vs Microservices |
| Q64 | 2. Sync vs Async Communication |
| Q65 | 3. FeignClient vs RestTemplate vs RestClient (2026) |
| Q66 | 4. API Gateway |
| Q67 | 5. Service Discovery — Eureka |
| Q68 | 6. Load Balancing |
| Q69 | 7. Distributed Tracing — Correlation IDs & Trace IDs |
| Q70 | 8. Event-Driven Architecture + Outbox Pattern |
| Q71 | 9. CQRS |
| Q72 | 10. Saga Pattern |
| Q73 | 11. Circuit Breaker |
| Q74 | 12. Fault Tolerance — Bulkhead, Retry, Timeout |
| Q75 | 13. Aggregator Pattern |

This file keeps the original detailed section flow from revision notes while preserving direct mapping to the master question bank.

## 1. Monolith vs Microservices

### Concept
Monolith is a single deployable application unit; microservices split the system into independently deployable services aligned to business capabilities.

### Simple Explanation
**Monolith** — One restaurant: same staff cooks, serves, cleans, does accounting. Simple when small. Kitchen slow → whole restaurant slow. Can't hire more waiters without more cooks.

**Microservices** — Food court. Each stall independent. Pizza stall slow → sushi unaffected. Scale just the pizza stall during lunch rush.

### How It Works
1. **Monolith flow**: one codebase + one deployable artifact + usually one shared DB.
2. **Microservices flow**: multiple services, each owning a bounded business area and its own data model.
3. Calls move from in-process method calls (monolith) to network calls/events (microservices).
4. This gives team-level independence but introduces distributed-system complexity.
5. Design choice should depend on team size, domain clarity, scaling needs, and operational maturity.

### Side-by-Side

| | Monolith | Microservices |
|---|---|---|
| Deployment | One unit | Many independent units |
| Scaling | Entire app | Individual services |
| Failure | One bug = everything down | Fault isolated |
| Data | Shared DB | Each service owns its DB |
| Team | Small ≤ 8 | Large distributed teams |
| Complexity | Low (infra) | High (distributed systems) |
| Testing | Simple | Integration testing hard |
| Latency | In-process (ns) | Network calls (ms) |

### When to Choose

**Monolith when:**
- Early product — domain not yet clear
- Small team, tight deadline
- No clear service boundaries

**Microservices when:**
- Different scaling needs per area (payments vs notifications)
- Large teams — Conway's Law: org structure = architecture
- Independent release cadence per domain
- Different SLA/reliability requirements per area

### Pitfalls of Microservices (what seniors must know)

| Pitfall | Real Impact |
|---|---|
| **Distributed transactions** | No ACID across services — need Saga, eventual consistency |
| **Cascade failures** | Inventory slow → Order threads back up → checkout degrades |
| **Data consistency** | Each service has its own DB — joins are hard, consistency is eventual |
| **Observability explosion** | 30 services × logs/metrics/traces — need centralised tooling |
| **Testing complexity** | Contract testing between services is hard |
| **Operational overhead** | 30 deployments, 30 CI/CD pipelines, 30 monitors |

### Real Usage
In practice, teams often start with a modular monolith to move quickly and discover domain boundaries. As traffic and team autonomy needs grow, they extract high-pressure areas first (payment, checkout, notifications) into microservices.

### Interview Answer
> "For a new product I start with a modular monolith — domain boundaries aren't clear yet. Once scaling bottlenecks emerge or team grows, I extract services. Teams go microservices-first and spend 80% on distributed systems infra instead of features.
>
> Honest trade-off: microservices replace big codebase complexity with distributed systems complexity. The pitfalls people underestimate: distributed transactions, cascade failures, and observability. Without mature DevOps and monitoring, microservices make things worse.
>
> **Follow-up:** *What is a modular monolith?*
> — Single deployable, code strictly in modules with enforced boundaries. Split into services later with minimal refactoring."

---

## 2. Sync vs Async Communication

### Concept
Synchronous communication blocks the caller until response; asynchronous communication publishes intent and decouples caller from immediate downstream execution.

### Simple Explanation
**Sync** — You call a friend and wait on hold until they answer. Your phone is blocked.

**Async** — You send WhatsApp and carry on with your day. They reply when ready. You're not blocked.

### How It Works
1. **Sync path**: service A calls service B over HTTP and waits.
2. If B is slow/down, A’s threads wait; latency and failure propagate upstream.
3. **Async path**: service A publishes event to broker; consumer services process later.
4. Producer returns early; downstream failures are handled by retries/DLQ without blocking request path.
5. Use sync when response is needed *now*; use async when eventual completion is acceptable.

### Side-by-Side

| | Synchronous (REST/FeignClient) | Asynchronous (Kafka/RabbitMQ) |
|---|---|---|
| Caller blocked? | ✅ Yes — waits for response | ❌ No — fire and continue |
| Coupling | Tight — receiver must be UP | Loose — receiver can be down temporarily |
| Latency | Low for single call | Higher (broker in the middle) |
| Failure | Circuit breaker needed | Message retained in broker, retried |
| Use when | Response needed to continue | Multiple consumers, eventual consistency OK |

### When to Choose

**Sync:** Inventory check before accepting order (answer changes next step), real-time auth check, simple request-response.

**Async:** Order placed → notify warehouse + send email + update analytics (all independent), burst traffic absorption, decoupled release cycles.

### Components You Should Know First
- `kafkaTemplate` = producer client used to publish events.
- `@KafkaListener` = consumer endpoint that receives events.
- `groupId` = consumer group identity; each group gets its own copy of events.
- `OrderEvent` = event payload contract shared by producer and consumer.

### Code — Sync (FeignClient)
```java
@FeignClient(name = "inventory-service", fallback = InventoryFallback.class)
public interface InventoryClient {
    @GetMapping("/inventory/{id}/available")
    boolean isAvailable(@PathVariable Long id, @RequestParam int qty);
}
```

### Code — Async (Kafka)
```java
public Order placeOrder(CreateOrderRequest req) {
    Order order = orderRepo.save(new Order(req));
    // fire event — doesn't wait for warehouse/email/analytics
    kafkaTemplate.send("order-placed", new OrderEvent(order.getId()));
    return order; // returns in <50ms
}

// Warehouse — independent consumer group
@KafkaListener(topics = "order-placed", groupId = "warehouse-service")
public void onOrderPlaced(OrderEvent e) { warehouseService.prepareShipment(e.getOrderId()); }

// Email — same event, different consumer group
@KafkaListener(topics = "order-placed", groupId = "email-service")
public void sendConfirmation(OrderEvent e) { emailService.send(e.getOrderId()); }
```

### Code Explained (What + Why)
- `kafkaTemplate.send("order-placed", ...)`
  - **What:** publishes an `OrderEvent` to Kafka topic `order-placed`.
  - **Why:** decouples order creation from downstream services (email, warehouse, analytics).
- `@KafkaListener(... groupId = "warehouse-service")`
  - **What:** warehouse service consumes the event asynchronously.
  - **Why:** warehouse can process at its own speed; order API is not blocked.
- `@KafkaListener(... groupId = "email-service")`
  - **What:** email service independently consumes the same event.
  - **Why:** one business action can trigger multiple downstream actions safely.

### Real Usage
Common production split: inventory validation is sync (decision-critical), while email, analytics, and loyalty updates are async (non-blocking, fan-out).

### Interview Answer
> "I choose based on whether caller needs response to continue. Inventory check before order — sync, the answer changes what happens next. Post-order notifications — async via Kafka, multiple services can react independently without the user waiting.
>
> Risk of sync: if Inventory is slow, Order threads pile up, cascade failure. That's why every sync call in microservices needs a circuit breaker. Async decouples this — order saved, event published, response in 50ms."

---

## 3. FeignClient vs RestTemplate vs RestClient (2026)

### Concept
Client choice depends on call context: internal service-to-service calls favor declarative clients; new external synchronous integrations favor modern Spring HTTP APIs.

### Simple Explanation
- `FeignClient` = internal company shuttle between services.
- `RestClient` = modern personal vehicle for external API calls.
- `RestTemplate` = older vehicle still running in legacy routes.

| | RestTemplate | RestClient | FeignClient |
|---|---|---|---|
| Status | Legacy / maintenance | Modern sync client (Spring 6.1+) | Declarative client for service-to-service |
| Style | Imperative, older API | Fluent, type-safe, modern API | Interface-driven |
| Best fit | Existing legacy code | New external HTTP integrations | Internal microservice calls |
| Discovery + LB | Manual | Manual unless combined with LB infra | Built-in via service name + Cloud LB |
| Resilience integration | Manual | Manual (cleaner than RestTemplate) | Natural fit with resilience patterns |
| 2026 recommendation | Avoid for new code | Preferred for new sync external calls | Preferred for internal service calls |

### Decision Rule (Interview Shortcut)
- Use `FeignClient` when calling internal services by logical name (`inventory-service`) with discovery/load balancing.
- Use `RestClient` for new synchronous third-party API integrations.
- Keep `RestTemplate` only where already present; do not choose it for new development.

### How It Works
1. `FeignClient` resolves service names via discovery/load balancer and generates HTTP client implementation from interface.
2. `RestClient` builds explicit HTTP requests and maps responses using modern fluent API.
3. `RestTemplate` provides older imperative style and is retained mainly for legacy code stability.
4. Resilience (retry, timeout, circuit breaker) should wrap all remote calls regardless of client choice.

```java
// FeignClient — declare interface, Spring generates implementation
@FeignClient(name = "product-service", fallback = ProductClientFallback.class)
public interface ProductClient {
    @GetMapping("/api/products/{id}")
    ProductDto getProduct(@PathVariable Long id);

    @GetMapping("/api/products")
    List<ProductDto> getByCategory(@RequestParam String category);
}

@Component
public class ProductClientFallback implements ProductClient {
    @Override
    public ProductDto getProduct(Long id) {
        return new ProductDto(id, "Unavailable", BigDecimal.ZERO); // graceful degradation
    }
    @Override
    public List<ProductDto> getByCategory(String category) { return Collections.emptyList(); }
}
```

### Code Explained (Feign)
- `@FeignClient(name = "product-service")`
  - **What:** defines an HTTP client by service name.
  - **Why:** avoids hardcoding host/port and works with service discovery + load balancing.
- `fallback = ProductClientFallback.class`
  - **What:** fallback path when downstream is failing.
  - **Why:** prevents cascading failures and gives graceful degradation.

```java
// RestClient — recommended for new synchronous external API integrations
@Bean
RestClient billingRestClient(RestClient.Builder builder) {
  return builder
    .baseUrl("https://billing-partner.example.com")
    .defaultHeader("X-Client", "order-service")
    .build();
}

public BillingResponse fetchInvoice(String invoiceId) {
  return billingRestClient.get()
    .uri("/api/invoices/{id}", invoiceId)
    .retrieve()
    .body(BillingResponse.class);
}
```

### Code Explained (RestClient)
- `RestClient.Builder` with `baseUrl(...)`
  - **What:** creates a reusable synchronous HTTP client.
  - **Why:** cleaner modern replacement for new `RestTemplate` use cases.
- `.retrieve().body(BillingResponse.class)`
  - **What:** executes request and maps response body to DTO.
  - **Why:** keeps external API calls concise and type-safe.

### How FeignClient Achieves Client-Side Load Balancing
```
FeignClient → asks Eureka for instances of "product-service"
           → gets [10.0.0.1:8080, 10.0.0.2:8080, 10.0.0.3:8080]
           → Spring Cloud LoadBalancer picks one (Round Robin)
           → direct HTTP call to chosen instance
           → all inside the JVM — NO external load balancer involved
```

### Real Usage
Typical pattern in Spring Boot microservices:
- Feign for internal domain services (`order → inventory`, `order → pricing`)
- RestClient for external partners (payment gateway, shipping provider)
- RestTemplate only where migration is still pending.

### Interview Answer
> "In 2026 I choose based on call type: FeignClient for internal service-to-service calls, RestClient for new synchronous external API calls, and WebClient for reactive/non-blocking flows. RestTemplate stays only in legacy code because it's in maintenance mode.
>
> **Follow-up:** *How does it handle retries?*
> — Configure `Retryer` bean or Spring Retry integration. Combine with circuit breaker — retry transient failures, but if failure rate exceeds threshold, circuit opens and fallback fires without retrying."

---

## 4. API Gateway

### Concept
API Gateway is a single entry point for clients that centralizes cross-cutting concerns (auth, routing, rate limiting, observability) so each service stays focused on business logic.

### Simple Explanation
Think of it like airport security + boarding gate: passengers (requests) are verified, routed to the correct flight (service), and monitored before they enter the terminal network.

### How It Works
1. Client calls gateway URL instead of calling services directly.
2. Gateway authenticates request and applies policy (rate limit, headers, filters).
3. Route predicates (path/header/method/weight) decide target service.
4. Gateway forwards request using service name (`lb://...`) and load balancing.
5. Gateway adds correlation metadata so logs can be traced end-to-end.

### What It Does
```
Client Request
    ↓
┌──────────────────────────────────────────────────┐
│                  API Gateway                      │
│  1. SSL Termination        6. Correlation ID      │
│  2. Authentication         7. Request Logging     │
│  3. Rate Limiting          8. Circuit Breaking    │
│  4. Path-based Routing     9. Response Caching    │
│  5. Load Balancing        10. Header Enrichment   │
└──────────────────────────────────────────────────┘
    ↓ routes to correct service
Order Service / Payment Service / User Service
```

### Spring Cloud Gateway Routing Predicates
```yaml
spring:
  cloud:
    gateway:
      routes:
        # 1. Path-based — most common
        - id: order-route
          uri: lb://order-service
          predicates: [Path=/api/orders/**]

        # 2. Weighted — canary deploys
        - id: order-v1
          uri: lb://order-service-v1
          predicates: [Path=/api/orders/**, "Weight=orders,90"]   # 90% → v1
        - id: order-v2
          uri: lb://order-service-v2
          predicates: [Path=/api/orders/**, "Weight=orders,10"]   # 10% → v2 (canary)

        # 3. Header-based — A/B testing, mobile vs web
        - id: mobile-route
          uri: lb://mobile-service
          predicates: [Path=/api/**, "Header=User-Agent,.*Mobile.*"]

        # 4. Method-based — CQRS: GET → read replica
        - id: orders-read
          uri: lb://order-read-service
          predicates: [Path=/api/orders/**, Method=GET]

        # 5. Rate limiting
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

### Code Explained (Gateway Routes)
- `uri: lb://order-service`
  - **What:** route by logical service name, not static IP.
  - **Why:** supports scaling/restarts without route changes.
- `Weight=orders,90` / `Weight=orders,10`
  - **What:** sends traffic by percentage to versions.
  - **Why:** safe canary rollout and quick rollback.
- `RequestRateLimiter`
  - **What:** limits request rate per route/key.
  - **Why:** protects services from abuse and sudden traffic spikes.

### Global Filter — Correlation ID Injection
```java
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = Optional
            .ofNullable(exchange.getRequest().getHeaders().getFirst("X-Correlation-Id"))
            .orElse(UUID.randomUUID().toString());

        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-Correlation-Id", correlationId).build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }
    @Override public int getOrder() { return -1; }
}
```

### Code Explained (Correlation Filter)
- Reads incoming `X-Correlation-Id` else generates one.
- Adds the ID to downstream request headers.
- **Why:** one ID ties together logs across gateway + all services for debugging.

### Real Usage
- One production gateway in front of all mobile/web clients.
- Team-level services avoid duplicating auth, throttling, and header policies.
- Canary rollout uses weighted routes to release risky changes safely.

### Diagnosing a Slow Service via Gateway
```
User reports: "checkout is slow"

1. Gateway metrics → /actuator/metrics/spring.cloud.gateway.requests
   Filter by route=order-service → p99 latency = 4200ms (normal: 200ms) ✓

2. Gateway access logs → all slow requests hit /api/orders/checkout since 14:32 UTC

3. Correlation ID → search ELK for that ID across all services
   → order-service calls inventory-service → inventory timing out

4. Circuit breaker state → /actuator/circuitbreakers
   → inventory-service = OPEN → root cause confirmed

5. Action: scale inventory-service or serve from cache
```

### Interview Answer
> "API Gateway is the front door for microservices. I keep routing, auth, rate limiting, and correlation ID propagation at gateway level so business services stay clean. In Spring Cloud Gateway, I mostly use path-based routing, weighted routing for canary rollout, and global filters for traceability.
>
> In production, when checkout becomes slow, I check gateway route-level p95/p99 first, then correlate logs using `X-Correlation-Id`, and verify whether a downstream circuit is open. That gives fast root-cause isolation.
>
> **Follow-up:** *Gateway vs Load Balancer?*
> — Gateway handles L7 policies + multi-service routing. Load balancer distributes across instances of the same service. Real systems typically use both."

---

## 5. Service Discovery — Eureka

### Concept
Service Discovery allows services to find each other dynamically by logical name instead of hardcoded host/port, which is critical when instances scale up/down frequently.

### Simple Explanation
It’s like a live company directory: instead of memorizing everyone’s desk number, you look up the latest number in the directory each time.

### How It Works
1. Eureka Server runs as registry.
2. Each service instance registers itself with metadata and sends heartbeat.
3. Consumer services fetch registry and cache it locally.
4. At call time, client-side load balancer picks one healthy instance by name.
5. Missing heartbeats expire stale instances automatically.

```
1. Eureka Server starts — acts as the registry

2. Service starts:
   → Registers: POST /eureka/apps/ORDER-SERVICE with host:port
   → Every 30s: heartbeat (PUT /eureka/apps/ORDER-SERVICE/instanceId)

3. Consumer (e.g., Payment) at startup:
   → Fetches full registry → caches locally
   → Every 30s: fetches delta updates only

4. At request time:
   → Looks up "ORDER-SERVICE" in LOCAL cache (zero Eureka network call)
   → LoadBalancer picks instance → direct HTTP call

5. Instance dies:
   → 3 missed heartbeats (90s) → Eureka deregisters
   → Next delta fetch excludes dead instance
```

### Components You Should Know First
- `Eureka Server` = service registry.
- `EnableDiscoveryClient` service = registers itself + discovers others.
- `lease-renewal` heartbeat interval = how often service says “I’m alive”.
- `lease-expiration` = when registry removes stale instance.

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp { }

// Any microservice
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApp { }
```

```yaml
# Service registration
spring:
  application:
    name: order-service   # MUST match — used for discovery + routing
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
eureka:
  client:
    healthcheck:
      enabled: true  # Actuator /health DOWN → Eureka marks OUT_OF_SERVICE
```

### Code Explained (Service Discovery)
- `spring.application.name: order-service`
  - **What:** logical service identity in registry.
  - **Why:** consumers resolve by name; no hardcoded network addresses.
- `defaultZone: http://eureka-server:8761/eureka/`
  - **What:** registry endpoint.
  - **Why:** registration + discovery lifecycle works through this server.
- `healthcheck.enabled: true`
  - **What:** integrates Actuator health with Eureka status.
  - **Why:** unhealthy instances are automatically removed from traffic.

### Real Usage
- New pods/instances join traffic automatically after startup.
- Dead instances are removed without redeploying callers.
- Teams avoid environment-specific service URL configs in code.

### Eureka Design Choice — AP (CAP theorem)
Eureka prioritises **Availability over Consistency**. In a network partition it keeps serving stale registry data rather than refusing requests. Clients may occasionally try a dead instance — circuit breakers + retry handle recovery.

### Alternative: Kubernetes Native Discovery
In K8s you often don't need Eureka — `order-service.namespace.svc.cluster.local` resolves automatically via CoreDNS. Spring Cloud Kubernetes uses K8s Services directly.

### Interview Answer
> "Service discovery removes hardcoded endpoints. In Eureka, services register and heartbeat, consumers keep a local registry cache, and client-side load balancing picks instances by service name. This gives fast lookups and handles autoscaling naturally.
>
> Important trade-off: Eureka is AP. During partition it may return stale data, but availability stays high. We mitigate stale endpoints using retries + circuit breakers and health-check integration with Actuator."

---

## 6. Load Balancing

### Concept
Load balancing distributes requests across multiple service instances to improve availability, throughput, and latency.

### Simple Explanation
Like a supermarket manager sending customers to multiple billing counters instead of letting everyone queue at one counter.

### How It Works
1. Client gets list of healthy instances (from Eureka/K8s).
2. A balancing strategy selects one instance per request.
3. Failed/slow instances are bypassed based on health and failure signals.
4. Traffic redistributes automatically when instances scale.

### Client-Side vs Server-Side

```
Server-Side (nginx / AWS ALB):
  Client → ALB → picks instance
  One extra network hop. Scales independently. External clients need this.

Client-Side (FeignClient + Spring Cloud LB):
  Client → Eureka (get instances) → pick locally → direct HTTP
  No extra hop. Lower latency. LB logic inside every client JVM.
```

### Algorithms

| Algorithm | How | Best For |
|---|---|---|
| Round Robin (default) | Rotate through instances | Homogeneous instances |
| Weighted | More traffic to higher-spec | Mixed instance sizes |
| Random | Pick randomly | Simple, homogeneous |
| Least Connections | Route to least busy | Variable request duration |

### Real Usage
- External traffic: server-side LB (ALB/Ingress).
- Internal service calls: client-side LB (Feign + Spring Cloud LoadBalancer).
- Canary + blue/green: combine weighted routing with LB health.

### Interview Answer
> "I use both types together: server-side load balancing at edge for internet traffic, and client-side load balancing for service-to-service calls. Client-side avoids extra network hop and works tightly with discovery.
>
> Strategy choice depends on workload. Round-robin is fine for equal nodes, weighted for mixed capacity, least-connections for variable request cost."

---

## 7. Distributed Tracing — Correlation IDs & Trace IDs

### Concept
Distributed tracing tracks one request across multiple microservices using shared tracing identifiers, making root-cause analysis feasible in distributed systems.

### Simple Explanation
Think of a courier parcel with one tracking number that is scanned at every hub; you can see where delay happened without calling every hub manually.

### How It Works
1. Gateway receives request and starts trace (or continues incoming trace).
2. Each service creates a new span while preserving same trace ID.
3. Trace headers propagate to downstream services automatically.
4. Logs include trace/span IDs via MDC.
5. Zipkin/Jaeger visualizes timeline to identify bottlenecks.

### The Problem
30 services, one user request touches 8 of them. Without correlation, debugging means grepping 8 separate log files and guessing the sequence — impossible at scale.

### Trace ID vs Span ID vs Correlation ID

| | Scope | Purpose |
|---|---|---|
| **Trace ID** | Same across ALL services for one request | Stitch the full journey |
| **Span ID** | New per service hop / operation | Time each individual step |
| **Correlation ID** | Often same as trace ID | Business-level request tracking |

```
User request → Gateway assigns Trace ID: abc-123
  Order Service:   [abc-123, span-001] Order saved: 456
  Payment Service: [abc-123, span-002] Charging card for order: 456
  Inventory:       [abc-123, span-003] Deducting stock for order: 456

Search "abc-123" in ELK → complete story across all services, in sequence
Zipkin shows: visual waterfall timeline of all spans
```

### Setup — Micrometer Tracing (Spring Boot 3)
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1    # 10% sampling in prod — 100% in dev
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

```java
// Zero code needed in services — Micrometer auto-injects traceId + spanId into MDC
// Log output automatically:
// 2026-03-13 [order-service,traceId=abc123,spanId=def456] Order saved: 789

// FeignClient automatically propagates trace headers to downstream services

// Manual span for a specific critical operation
@Autowired Tracer tracer;

public void processPayment(Payment p) {
    Span span = tracer.nextSpan().name("process-payment").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        span.tag("payment.amount", p.getAmount().toString());
        doProcess(p);
    } finally {
        span.end();
    }
}
```

### Real Usage
- Incident response starts with trace ID from gateway/access log.
- Same trace ID searched in ELK quickly reveals failing hop.
- Hot paths (checkout/payment) get custom spans for deeper visibility.

### Interview Answer
> "In microservices, tracing is non-negotiable. I use Micrometer Tracing in Spring Boot 3 so every log line carries trace and span IDs automatically. Trace ID is constant across the full request path; span IDs capture each hop.
>
> In production, this cuts MTTR significantly: take one trace ID from user request, search ELK, open Zipkin waterfall, and identify exactly which downstream call caused latency or failure. Sampling is tuned by environment to control overhead."

---

## 8. Event-Driven Architecture + Outbox Pattern

### Concept
Event-Driven Architecture (EDA) lets services communicate via events instead of tight synchronous calls, and the Outbox pattern guarantees reliable event publication alongside DB writes.

### Simple Explanation
Instead of calling every department directly when an order is placed, you post one official notice on a board; each department reads it and acts independently.

### How It Works
1. Command service persists business data.
2. In the same DB transaction, it writes event record to outbox table.
3. Background publisher (or CDC) reads pending outbox rows.
4. Publisher sends events to Kafka and marks rows as published.
5. Consumers process events independently and idempotently.

### When to Choose EDA

| Choose EDA | Choose Sync |
|---|---|
| Multiple services react to one event | Only one service needs to react |
| Downstream can be slow/unavailable | Immediate response needed to continue |
| High throughput, burst traffic | Tight latency SLA required |
| Services should evolve independently | Transactional consistency critical |

### Outbox Pattern — Solving the Dual-Write Problem

```
PROBLEM: Two separate writes — not atomic

orderRepo.save(order);             // ① DB write
kafkaTemplate.send("order-placed") // ② Kafka write
                                   // ① succeeds, ② fails → event LOST → inconsistency
```

### Components You Should Know First
- `outbox_events` table = durable list of events waiting to be published.
- `OutboxEvent` = row model for one event (`eventType`, `payload`, `status`).
- `outboxRepo` = repository for storing/fetching outbox rows.
- `poller` = background publisher that sends pending rows to Kafka.

```java
// SOLUTION: Write event to outbox table in SAME DB transaction
@Transactional
public Order placeOrder(CreateOrderRequest req) {
    Order order = orderRepo.save(new Order(req));

    // Same transaction as order save — atomically committed or rolled back together
    outboxRepo.save(new OutboxEvent(
        "order-placed",
        objectMapper.writeValueAsString(new OrderPlacedEvent(order))
    ));
    return order;
    // Separate poller (or Debezium CDC) reads outbox → publishes to Kafka
    // Event is NEVER lost — DB is the source of truth
}

// Outbox poller — runs independently
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishPendingEvents() {
    List<OutboxEvent> pending = outboxRepo.findByPublishedFalse();
    pending.forEach(event -> {
        kafkaTemplate.send(event.getTopic(), event.getPayload());
        event.setPublished(true);
    });
    outboxRepo.saveAll(pending);
}
```

### Code Explained (Outbox)
- `@Transactional` on `placeOrder(...)`
  - **What:** order save + outbox save happen in one DB transaction.
  - **Why:** prevents “order saved but event lost” inconsistency.
- `outboxRepo.save(new OutboxEvent(...))`
  - **What:** stores event intent in DB.
  - **Why:** even if Kafka is down now, event can be published later.
- `@Scheduled publishPendingEvents()`
  - **What:** reads unsent outbox rows and publishes to Kafka.
  - **Why:** retries asynchronously without blocking user request path.

### Real Usage
- `order-service` publishes `OrderPlaced` once and returns quickly.
- `payment`, `inventory`, `notification`, `analytics` consume independently.
- Kafka downtime does not lose events because outbox rows remain pending.

### Interview Answer
> "I use EDA when one business event must trigger multiple downstream actions independently. For reliability, I avoid direct DB+Kafka dual writes and use Outbox: write business row and outbox row in one transaction, then publish asynchronously.
>
> This gives both performance and consistency: user path stays fast, and events are never lost even if Kafka is temporarily unavailable. Consumers are idempotent so retries are safe.
>
> **Follow-up:** *How do you keep order of events?*
> — Use stable partition key like `orderId`, so events for the same order stay in-order in one partition."

---

## 9. CQRS (Command Query Responsibility Segregation)

### Concept
CQRS separates commands (writes) and queries (reads) into completely independent data models, allowing each to be optimized for its specific workload.

### Simple Explanation
Two bank windows: **Teller** (Command) — deposit, withdraw — full validation, paperwork, official ledger. **Balance screen** (Query) — just read, must be instant. Completely different concerns — CQRS separates them.

### How It Works
1. Command path: validate business rules, persist to normalized write DB, emit event.
2. Event published to Kafka.
3. Read model listener receives event and updates denormalized query table.
4. Client reads from the optimized flattened query table (no joins needed).

### The Split

```
Write Side (Command)                Read Side (Query)
────────────────────                ─────────────────
Full domain validation              Pre-joined flat tables
Normalised DB (PostgreSQL)          Denormalised read store
Domain model, strict rules          No business logic
Publishes events via Kafka ──────►  Updated asynchronously
Slower writes OK                    Millisecond reads
```

```java
// Command side — rich domain model
@Transactional
public OrderId placeOrder(PlaceOrderCommand cmd) {
    Order order = Order.create(cmd);   // all business rules applied
    orderRepo.save(order);
    eventBus.publish(new OrderPlacedEvent(order.toSnapshot())); // update read side
    return order.getId();
}

// Query side — zero business logic, pure data access
public OrderSummaryDto getOrderSummary(Long id) {
    return orderSummaryRepo.findById(id);  // flat pre-joined table, <15ms
}

// Read model updater — separate bean, listens to events
@EventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    orderSummaryRepo.save(new OrderSummary(
        event.getOrderId(),
        event.getUserName(),   // denormalised — joined at write time
        event.getTotal(),
        event.getStatus()
    ));
}
```

### Code Explained (CQRS)
- `Order.create(cmd)` on write side
  - **What:** full domain model with business rules.
  - **Why:** ensures only valid state enters the system.
- `orderSummaryRepo.findById(id)` on read side
  - **What:** flat denormalized table, no joins.
  - **Why:** queries become simple and fast.
- `@EventListener` feeding read model
  - **What:** separate process updates read table when write happens.
  - **Why:** reads and writes scale independently.

### Real Usage
- Order management: write-heavy validation vs read-heavy dashboards.
- Complex reporting queries (100+ JOINs) replaced with single-table denormalized reads.
- Analytics updated asynchronously — dashboards stay responsive.

### When to Use
- ✅ Read load >> write load — scale independently
- ✅ Complex reporting killing write DB performance
- ✅ Dashboard queries very different from write operations
- ❌ Simple CRUD — doubles complexity for no gain
- ❌ Financial balances — eventual consistency not acceptable

### Interview Answer
> "I used CQRS for an order management system with a heavy-traffic dashboard. The command side handles validation and maintains the source of truth in a normalized schema. Events flow to a read model that pre-joins order + customer + product data into a flat denormalized table.
>
> Result: order creation stays transactional and strict. Dashboards query a flat table in ~15ms instead of 800ms with multiple joins.
>
> Trade-off: read side is eventually consistent — typically updates within 100ms but may lag. Acceptable for dashboards. I'd never use CQRS for account balances.
>
> **Follow-up:** *Does CQRS require event sourcing?*
> — No. They pair well but are independent. CQRS = separate read/write models. Event sourcing = store state as sequence of events."

---

## 10. Saga Pattern

### Concept
Saga pattern implements distributed transactions by orchestrating a sequence of local transactions across services, with automatic rollback (compensating transactions) if any step fails.

### Simple Explanation
Booking a holiday: flight → hotel → car rental. Three separate systems, no shared DB. Car rental unavailable? Cancel hotel + cancel flight. That reversal sequence = **compensating transactions**. The whole sequence = **Saga**.

### How It Works
1. Saga initiator triggers first step (create order).
2. Each successful step publishes event or triggers next step.
3. If any step fails, compensating transactions execute in reverse order.
4. System reaches eventual consistency without distributed locks.

### Two Types

**Choreography** — services react to each other's events:
```
Order Service ──OrderPlaced──► Payment Service ──PaymentSuccess──► Inventory
                                     │
                               PaymentFailed
                                     │
                            ◄──OrderCancelled── Order Service
```
✅ Decoupled, no central coordinator
❌ Hard to trace, hard to debug complex flows

**Orchestration** — one central coordinator drives all steps:
```java
@Component
public class OrderSagaOrchestrator {
    public void execute(CreateOrderCommand cmd) {
        SagaContext ctx = new SagaContext();
        try {
            ctx.setOrderId(orderService.createOrder(cmd));         // step 1
            ctx.setPaymentId(paymentService.charge(cmd.getAmount())); // step 2
            inventoryService.deductStock(cmd.getItems());           // step 3 — may fail
            orderService.confirmOrder(ctx.getOrderId());
        } catch (Exception e) {
            compensate(ctx); // rollback in reverse order
            throw new SagaFailedException(e);
        }
    }

    private void compensate(SagaContext ctx) {
        // Always idempotent — called twice = same result
        if (ctx.getPaymentId() != null)
            paymentService.refund(ctx.getPaymentId());  // compensating tx
        if (ctx.getOrderId() != null)
            orderService.cancelOrder(ctx.getOrderId()); // compensating tx
    }
}
```

### Code Explained (Saga Orchestrator)
- `SagaContext ctx = new SagaContext()`
  - **What:** stores completed step IDs/state.
  - **Why:** compensation logic needs this state to roll back safely.
- `try { ... } catch { compensate(ctx); }`
  - **What:** execute local transactions, compensate on failure.
  - **Why:** replaces distributed ACID transaction with controlled eventual consistency.
- `paymentService.refund(...)` / `orderService.cancelOrder(...)`
  - **What:** compensating actions in reverse direction.
  - **Why:** restores business consistency after partial success.
✅ Clear flow, easy to debug, state in one place
❌ Orchestrator couples to all services

### Critical Points

| Point | Detail |
|---|---|
| **Idempotency is MANDATORY** | Compensating transaction called twice = same result |
| **Saga ≠ Atomic** | Other services see intermediate state during execution |
| **Saga vs 2PC** | 2PC locks resources across services (deadlocks, perf). Saga = local txs + events, no distributed lock |
| **Use Orchestration for** | Complex 3+ step flows, need visibility/debugging |
| **Use Choreography for** | Simple 2-step flows, maximum decoupling needed |

### Interview Answer
> "Saga solves distributed transactions. Order, Payment, Inventory — each owns its DB, no shared transaction. Saga: local transaction in each service + event to signal completion. Failure triggers compensating transactions in reverse.
>
> I prefer orchestration — one Saga Orchestrator drives all steps, knows current state, triggers compensations. Easier to debug than choreography where flow is implicit in event chains.
>
> Critical: every compensating transaction must be idempotent. If refund is called twice due to a retry, customer still only refunded once. I check if refund exists before processing.
>
> **Follow-up:** *Saga vs 2PC?*
> — 2PC locks all resources during the transaction — distributed deadlocks, massive performance hit. Saga uses local transactions, no distributed lock, eventual consistency — temporary inconsistency window but scales and performs."

---

## 11. Circuit Breaker

### Concept
Circuit breaker pattern prevents cascading failures by stopping requests to failing services and allowing them time to recover.

### Simple Explanation
House circuit breaker: short circuit → breaker trips → stops electricity rather than burning the house. After fixing: flip it back. Same concept: stop calling broken service, let it recover, cautiously resume.

### How It Works
1. **CLOSED**: Monitors call success/failure rate.
2. **OPEN**: Failure rate exceeds threshold → immediately reject calls without network attempt.
3. **HALF-OPEN**: After timeout, allow limited test calls. Success → back to CLOSED. Failure → back to OPEN.

### 3 States

```
              failure rate > threshold
CLOSED ─────────────────────────────────► OPEN
(normal, all calls go through)           (ALL calls fail immediately, fallback returned)
  ▲                                            │
  │                                            │ after waitDurationInOpenState (30s)
  │                                            ▼
  │                                      HALF-OPEN
  │                                   (N test calls allowed)
  │   test calls succeed                       │
  └────────────────────────────────────────────┘
     if test calls fail → back to OPEN
```

- **CLOSED** — tracks failure rate of last N calls
- **OPEN** — no network call made. Fallback returned in microseconds. Thread freed immediately.
- **HALF-OPEN** — 3–5 test calls. Success → CLOSED. Failure → OPEN.

### Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10          # evaluate last 10 calls
        failureRateThreshold: 50       # open if 50% fail
        slowCallRateThreshold: 60      # open if 60% take > slowCallDurationThreshold
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 30s   # stay open 30s before HALF-OPEN
        permittedNumberOfCallsInHalfOpenState: 3  # test calls
        minimumNumberOfCalls: 5        # don't evaluate until 5 calls made

        # ⚠️ MOST MISSED CONFIG — exception classification
        ignoreExceptions:
          - com.app.exception.CardDeclinedException     # business failure, NOT infra
          - com.app.exception.NotFoundException         # 404 = bad request, not outage
          - com.app.exception.ValidationException
        recordExceptions:
          - java.net.ConnectException                   # can't reach service
          - java.util.concurrent.TimeoutException       # service too slow
          - feign.FeignException$ServiceUnavailable
```

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult processPayment(PaymentRequest req) {
    return stripeClient.charge(req);
}

public PaymentResult paymentFallback(PaymentRequest req, Exception ex) {
    if (ex instanceof CallNotPermittedException) {
        // Circuit is OPEN — known service downtime
        paymentRetryQueue.enqueue(req);
        return PaymentResult.pending("Payment queued — processing shortly");
    }
    log.error("Payment failed: {}", ex.getMessage());
    return PaymentResult.failed("Payment temporarily unavailable");
}
```

### Code Explained (Circuit Breaker)
- `@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")`
  - **What:** wraps external call with resilience4j state machine.
  - **Why:** automatically handles state transitions, avoids manual retry logic.
- `if (ex instanceof CallNotPermittedException)`
  - **What:** circuit is OPEN, call is blocked immediately.
  - **Why:** fail fast (microseconds) instead of waiting for timeout.
- `paymentRetryQueue.enqueue(req)`
  - **What:** queues work for later async retry.
  - **Why:** preserves business intent without blocking user.

### Real Usage
- Payment integrations: when processor is down, queue requests for later batch processing.
- Inventory checks: fall back to stale cache during Inventory service outage.
- Alerts: open circuit generates alert for on-call engineer to investigate.

### Why `ignoreExceptions` Is Critical
If card-declined exceptions count as failures → run of bad user inputs opens the circuit → nobody can pay → your business logic caused an outage. **Only infrastructure failures** (timeouts, connection refused) should trip the breaker.

### Monitor Circuit State
```
GET /actuator/circuitbreakers
{
  "paymentService": { "state": "OPEN", "failureRate": "60.0%" }
}
```
Alert on OPEN state in production — early warning of service degradation.

---

## 12. Fault Tolerance — Bulkhead, Retry, Timeout

### Concept
Fault tolerance uses four independent techniques (bulkhead, retry, timeout, circuit breaker) layered together to handle failures gracefully.

### Simple Explanation
A ship has multiple systems: engine isolated from helm, ballast compartments sealed (bulkheads), automatic backup generators (retry with fallback), and distress signal if critical systems fail (circuit breaker). Each handles failure independently.

### How It Works
1. Request arrives.
2. Bulkhead allocates a thread from a dedicated pool.
3. If call fails transiently, retry with exponential backoff.
4. If retry takes too long, timeout terminates it.
5. If failure rate becomes persistent, circuit breaker opens to prevent further attempts.

### All Techniques Combined

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
@Retry(name = "inventory")
@TimeLimiter(name = "inventory")   // timeout
@Bulkhead(name = "inventory")      // dedicated thread pool
public CompletableFuture<Boolean> isAvailable(Long id, int qty) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.check(id, qty));
}
```

### Code Explained (Retry + Timeout + Bulkhead)
- `@Retry`
  - **What:** retries transient failures.
  - **Why:** many network errors are temporary and recover quickly.
- `@TimeLimiter`
  - **What:** caps max wait time.
  - **Why:** prevents blocked threads and high tail latency.
- `@Bulkhead`
  - **What:** isolates concurrency per dependency.
  - **Why:** one slow dependency should not consume all application resources.

```yaml
resilience4j:
  retry:
    instances:
      inventory:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2    # 500ms → 1s → 2s
        retryExceptions:
          - java.net.ConnectException      # only retry infra failures
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.app.exception.BusinessException  # don't retry business failures

  timelimiter:
    instances:
      inventory:
        timeoutDuration: 2s               # fail fast after 2s

  bulkhead:
    instances:
      inventory:
        maxConcurrentCalls: 20            # max 20 threads for inventory calls
        maxWaitDuration: 100ms            # wait max 100ms for a thread slot
```

### Real Usage
- All three (retry + timeout + bulkhead) wrap every external call in production.
- Circuit breaker monitors health and opens if failures become systematic.
- Public SLA depends on this layered fault tolerance — not just good network.

### Bulkhead Pattern — Why It Matters

```
Without Bulkhead:
  App has 200 threads. Inventory is slow.
  100 requests → 100 threads blocked waiting on inventory
  Other services (orders, payments) only have 100 threads left
  If payments also gets slow → everything queues → app-wide degradation

With Bulkhead:
  Inventory: max 20 dedicated threads
  Payments: max 30 dedicated threads
  Orders: remaining threads
  Inventory slow → only 20 threads blocked → payments, orders unaffected
```

Think of a ship's bulkheads — watertight compartments. One floods, others stay dry.

### Retry Anti-Patterns
```
❌ Retry immediately — hammers an already struggling service
❌ Retry non-transient failures (business validation, auth errors) — pointless
❌ Retry without circuit breaker — keeps retrying even when service is down for 30 min
❌ Retry storms — 1000 clients all retry at same time → "thundering herd"

✅ Exponential backoff: 500ms → 1s → 2s → give up
✅ Jitter: add random delay to spread retries: 500ms ± 100ms random
✅ Only retry idempotent operations (GET, PUT) — retrying POST may create duplicates
✅ Circuit breaker wraps retry — opens after N failures, stops retry attempts
```

### Interview Answer
> "I use all four techniques together: circuit breaker to stop calling a clearly failing service, retry with exponential backoff for transient network blips, timeout to fail fast and free threads, and bulkhead to give each downstream dependency its own thread pool.
>
> Bulkhead is the one most teams skip. Without it, one slow service can monopolise all 200 threads — app-wide degradation from one dependency. With it, inventory can be completely dead and only its 20 threads are blocked — payments and orders keep running normally."

---

## 13. Aggregator (API Composition) Pattern

### Concept
Aggregator pattern collects and merges data from multiple downstream services into one response, typically for dashboard or UI scenarios.

### Simple Explanation
Banking dashboard: balance, transactions, credit score, pending payments — four systems. You call ONE endpoint. Aggregator calls all 4 **in parallel**, waits, merges, returns one response.

### How It Works
1. Incoming request arrives at aggregator.
2. Aggregator spawns parallel calls to all required downstream services.
3. Each call wraps in timeout and fallback.
4. Aggregator waits for all (with combined timeout).
5. Merges results and returns combined response.

```java
@Service
public class DashboardAggregatorService {

    public DashboardDto getDashboard(Long userId) {
        // Fire ALL calls in parallel
        CompletableFuture<UserProfile> userF =
            CompletableFuture.supplyAsync(() -> userClient.getProfile(userId), pool)
                .exceptionally(ex -> UserProfile.empty());     // per-service fallback

        CompletableFuture<List<Order>> ordersF =
            CompletableFuture.supplyAsync(() -> orderClient.getOrders(userId), pool)
                .exceptionally(ex -> Collections.emptyList());

        CompletableFuture<PaymentSummary> paymentF =
            CompletableFuture.supplyAsync(() -> paymentClient.getSummary(userId), pool)
                .exceptionally(ex -> PaymentSummary.empty());

        // Wait for ALL — with combined timeout
        try {
            CompletableFuture.allOf(userF, ordersF, paymentF)
                .get(3, TimeUnit.SECONDS);  // max 3s total — partial response if timeout
        } catch (TimeoutException e) {
            log.warn("Dashboard aggregation partial — timeout exceeded");
        }

        return DashboardDto.builder()
            .user(userF.join())
            .orders(ordersF.join())
            .payment(paymentF.join())
            .build();
        // Total time = max(individual times) NOT sum
        // 3 calls × 200ms = ~200ms (parallel), not 600ms (sequential)
    }
}
```

### Code Explained (Aggregator)
- `CompletableFuture.supplyAsync(...)` per downstream call
  - **What:** executes calls in parallel threads.
  - **Why:** total latency becomes near max(call times), not sum(call times).
- `.exceptionally(...)`
  - **What:** fallback per dependency.
  - **Why:** partial response is better than full failure for dashboards.
- `CompletableFuture.allOf(...).get(3, TimeUnit.SECONDS)`
  - **What:** global timeout for fan-out call set.
  - **Why:** protects API response SLA and prevents indefinite waiting.

### Real Usage
- Dashboard endpoint that needs user profile, order history, payment methods, recommendations.
- Admin console that aggregates metrics from multiple monitoring systems.
- Partial responses acceptable when one service slow — better UX than full timeout.

**BFF Pattern (Backend For Frontend):**
Instead of one aggregator, one per client type:
- Mobile BFF: lightweight data, fewer fields, small payloads
- Web BFF: rich data, charts, full detail
- Partner BFF: partner-specific contract

Each BFF knows exactly what its consumer needs — zero overfetching or underfetching.

---

