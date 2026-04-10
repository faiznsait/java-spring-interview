# Topic 14: Microservices Patterns and Resilience

## Question Bank Coverage Mapping (Q76-Q81)

| Question Bank Q# | Section In This File |
|---|---|
| Q76 | 14. CAP Theorem & Eventual Consistency |
| Q77 | 15. Idempotency in Distributed Systems |
| Q78 | 16. Database Per Service Pattern |
| Q79 | 17. Service Decomposition — Domain-Driven Design |
| Q80 | 18. API Versioning Strategies |
| Q81 | 19. Graceful Shutdown |

Additional sections in this file are advanced extensions retained from revision notes to enrich interview depth beyond minimum question-bank scope.

## 14. CAP Theorem & Eventual Consistency

### Concept
CAP theorem proves that in a distributed system you can only guarantee two of three properties. Understanding this determines your database and consistency strategy.

### Simple Explanation
Say you're running a bank account system across two datacenters. Best case: both have identical balance (Consistency). If network breaks between them, you must choose: refuse all updates (Consistency) or let both continue updating (Availability). You can't have both during a partition.

### How It Works
1. System operates normally with all three (C+A+P).
2. Network partition occurs between datacenters.
3. You're forced to choose: stop serving (CP) or serve stale data (AP).
4. System goes back normal after network heals.

### CAP Theorem
In a distributed system you can only guarantee 2 of 3:

| Property | What it means |
|---|---|
| **Consistency** | Every read returns the most recent write |
| **Availability** | Every request gets a response (even if not latest data) |
| **Partition Tolerance** | System keeps running despite network partition |

Network partitions happen in distributed systems — you MUST tolerate them. So the real choice is: **CP (Consistency + Partition) vs AP (Availability + Partition)**.
The C vs A trade-off is evaluated during a partition event; outside partitions, systems can provide both C and A for normal traffic.

```
CP Systems: Prefer to refuse requests rather than serve stale data
  - Zookeeper, HBase, MongoDB (in strict mode)
  - Use for: financial transactions, inventory (where stale = overselling)

AP Systems: Prefer to serve stale data rather than go down
  - Eureka, Cassandra, CouchDB
  - Use for: product catalog, user profiles, recommendations (stale is fine)
```

### Real Usage
- Financial systems: CP — refuse payments during partition rather than oversell balance.
- E-commerce catalog: AP — serve stale product prices during partition rather than 503 error.
- Eureka: explicitly AP — trades strong consistency for high availability.

### Code Explained (Choosing C vs A)
- CP is chosen when: `spring.datasource` with single-primary replication (waits for sync).
- AP is chosen when: Eureka for discovery, Cassandra each DC independent.

### Interview Answer
> "CAP theorem says in a distributed system you pick Consistency or Availability during a partition. I choose based on business impact: financial systems are CP (refuse requests rather than wrong balance). Product catalogs are AP (serve yesterday's prices rather than 503).
>
> In microservices, eventual consistency is the norm via events. Order placed → all services eventually receive and react. The lag (50-500ms) is acceptable for most use cases.
>
> **Follow-up:** *What if partition lasts 30 minutes?*
> — depends on choice: CP systems are DOWN for 30min. AP systems serve 30-min-old data. Pick based on your SLA.""

### Eventual Consistency
In microservices, full consistency across services is impossible without distributed locks (too expensive). Instead: each service is internally consistent; changes propagate via events — all services will **eventually** converge to a consistent state.

```
Order placed → Order DB updated immediately (the source of truth)
             → OrderPlacedEvent published to Kafka
             → Inventory DB updated 50ms later (eventually consistent)
             → Read model DB updated 80ms later (eventually consistent)

During those 80ms: inventory count is "stale" in the read model
After 80ms: all systems consistent again
```

### A common interview level question: Where to draw the boundary

```
Always consistent (synchronous, same DB):
  - Account balance during a transfer
  - Inventory during a flash sale checkout (overselling = real loss)

Eventually consistent (async, acceptable lag):
  - Order history/dashboard updates
  - Email sending after order
  - Analytics and reporting
  - Recommendation engine updates
```

---

## 15. Idempotency in Distributed Systems

### Concept
An operation is **idempotent** if calling it multiple times produces the same result as calling it once. Critical in distributed systems where retries are mandatory.

### Simple Explanation
Charging a credit card is NOT idempotent: charge five times = five charges. Updating status to "Confirmed" IS idempotent: set to "Confirmed" five times = still "Confirmed". With network failures triggering retries, make every operation idempotent or lose data.

### How It Works
1. Client sends request with unique idempotency key or deterministic ID.
2. Service checks if already processed — if yes, return stored result.
3. If not processed, execute operation and store result.
4. Future retries get cached result — no duplicate side effects.

### Why It Matters
In distributed systems, **retries are unavoidable** (network blips, timeouts, consumer restarts). If your operations aren't idempotent, retries = duplicate payments, duplicate orders, duplicate notifications.

```
Scenario without idempotency:
  → Payment service calls Bank API to charge ₹1000
  → Bank charges, sends response
  → Network timeout — payment service never receives response
  → Payment service RETRIES
  → Bank charges AGAIN — customer charged ₹2000
  → Bug in production
```

### Implementation Patterns

**Pattern 1: Idempotency Key**
```java
// Client sends unique idempotency key with every request
@PostMapping("/payments")
public PaymentResult processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest req) {

    // Check if we've already processed this key
    Optional<PaymentResult> existing = idempotencyStore.find(idempotencyKey);
    if (existing.isPresent()) {
        return existing.get();  // return same result — idempotent
    }

    // New request — process and store result
    PaymentResult result = bankClient.charge(req);
    idempotencyStore.save(idempotencyKey, result, Duration.ofHours(24));
    return result;
}
```

**Pattern 2: Database Unique Constraint**
```java
// Kafka consumer — may receive duplicate messages if consumer restarts
@KafkaListener(topics = "order-placed")
public void onOrderPlaced(OrderPlacedEvent event) {
    // Unique constraint on event_id prevents double-processing
    if (processedEventRepo.existsByEventId(event.getEventId())) {
        log.info("Duplicate event ignored: {}", event.getEventId());
        return;
    }
    try {
        processedEventRepo.save(new ProcessedEvent(event.getEventId()));
        warehouseService.prepareShipment(event.getOrderId());
    } catch (DataIntegrityViolationException e) {
        // Another instance processed it concurrently — ignore
        log.info("Concurrent duplicate event ignored: {}", event.getEventId());
    }
}
```

### Code Explained (Idempotent Handler)
- `idempotencyStore.find(idempotencyKey)` first
  - **What:** checks if we've seen this request before.
  - **Why:** prevent duplicate charge/payment on retry.
- `idempotencyStore.save(idempotencyKey, result, 24h)`
  - **What:** cache the result for 24 hours.
  - **Why:** future retries with same key get cached response without re-executing.
- `existsByEventId(...)` + unique constraint in Kafka consumer
  - **What:** deduplication at database level.
  - **Why:** handles concurrent replayed messages safely.
- `catch (DataIntegrityViolationException ...)`
  - **What:** handles race condition where two instances process same event.
  - **Why:** constraint violation is expected, safe to ignore.

**Pattern 3: Natural Idempotency (PUT)**
```java
// PUT is naturally idempotent — setting status to CONFIRMED 10 times = CONFIRMED
@PutMapping("/orders/{id}/status")
public void updateStatus(@PathVariable Long id, @RequestBody String status) {
    orderRepo.updateStatus(id, status);  // idempotent by nature
}
// vs POST which creates new records — NOT idempotent
```

### Real Usage
- Payment APIs: every POST adds `Idempotency-Key` header, stored 24h.
- Kafka consumers: `processedEventId` table tracks what's been processed.
- Idempotent PUT operations: status updates naturally idempotent.
- Frontend: retries payment automatically if network timeout, same key sent both times.

### Interview Answer
> "Idempotency is non-negotiable in distributed systems because retries are unavoidable. Every mutating endpoint accepts an `Idempotency-Key` header. If system crashes after charge but before response, next retry with same key returns cached result.
>
> For async paths like Kafka consumers, I track processed event IDs in the DB with a unique constraint. If consumer restarts and replays old messages, duplicates hit the unique constraint, exception is caught and ignored, consumer continues safely.
>
> The rule: if retry could cause duplicate side effects, implement idempotency first.""

---

## 16. Database Per Service Pattern

### Concept
Database Per Service means each microservice owns its own database schema. Nothing is shared. This ensures service independence but adds complexity to querying across services.

### Simple Explanation
Shared database: all teams have edit access to same tables. One team's schema change breaks everyone. Nightmare for autonomy. Database per service: each team owns their schema completely. They scale it, change it, technology independently. The trade-off: you can't join across services; must use aggregation or denormalization.

### How It Works
1. Order Service owns Orders DB schema entirely.
2. Payment Service owns Payments DB — knows nothing of Orders schema.
3. If Order needs product data, call Product Service (cross-service call) or denormalize product name into Orders.
4. Each DB scaled, backed up, deployed independently.

### Why Each Service Must Own Its DB

```
❌ SHARED DATABASE:
  Order Service ──┐
  Payment Service ─┤──► Shared DB
  Inventory Service ┘

Problems:
  - Schema changes by one team break all others
  - One slow query from one service degrades all services
  - No independent scaling of storage
  - All services must be deployed together for schema migrations
  - Defeats the purpose of microservices independence
```

```
✅ DATABASE PER SERVICE:
  Order Service ──► Orders DB (PostgreSQL)
  Payment Service ──► Payments DB (PostgreSQL + encrypted)
  Inventory Service ──► Inventory DB (PostgreSQL / Redis for stock counts)
  Product Service ──► Product DB (MongoDB — flexible schema for catalogue)
  Search Service ──► Elasticsearch
```

### Code Explained (Handling Cross-Service Data)
- Each service has minimal data: Orders table only has `orderId, productId, userId, amount`.
- No foreign key to Product table (lives in different DB).
- When need product.name, call Product Service API.

### Real Usage
- Order Service: PostgreSQL for transactional integrity.
- Product Service: MongoDB for flexible schema (catalog).
- Search Service: Elasticsearch for full-text search.
- All independent, no shared schema.

### How to Handle Cross-Service Queries

```
Problem: "Show me order details with product name and customer email"
  Order DB has: orderId, productId, userId, amount
  Product DB has: productId, name, price
  User DB has: userId, email, name

Option 1 — Aggregator Service (API Composition)
  → Call Order Service, Product Service, User Service in parallel
  → Merge in aggregator
  → Works for real-time small queries

Option 2 — Read Model / CQRS
  → Maintain a denormalised "order-summary" table in Orders service
  → Updated async via events when Product or User changes
  → Single fast query for reporting

Option 3 — Event-Driven Denormalisation
  → When user changes email, publish UserEmailUpdated event
  → Order service subscribes, copies email into its own Orders table
  → Orders table has everything it needs for fast local queries — zero calls to User service
```

### Interview Answer
> "Database per service is non-negotiable — shared DB creates a monolith you can't escape. Each team owns their schema, scales independently, can choose different technologies.
>
> Cross-service queries: I favour event-driven denormalization. When user email changes, that event flows to Order service which updates its local 'denormalized_email' column. Order queries stay fast and local. The trade-off is eventual consistency — email updates lag by 50-100ms, but that's acceptable.
>
> Aggregator pattern is fallback for real-time small queries where you must have current data across services.""

---

## 17. Service Decomposition — Domain-Driven Design

### How to Decide Service Boundaries

**Wrong approach:** Split by technical layer (all CRUD in one service, all validation in another)

**Right approach:** Split by **business capability / bounded context**

### Concept
Service decomposition using Domain-Driven Design (DDD) boundaries ensures services are organized around business capabilities, not technical layers. This drives independent development, scaling, and deployment.

### Simple Explanation
Wrong: "Create a Settings service that handles all configuration." Right: "Create an Order service that owns order entity, cart, pricing rules." Services are business capabilities, not technical utilities. Each service speaks its business language.

### How It Works
1. Map business domains (Order, Payment, Inventory, Catalog).
2. Within domain, define Bounded Context — what data/rules belong together.
3. Each Bounded Context becomes one service.
4. Services communicate via public APIs, not by sharing internal code.

### How to Decide Service Boundaries

**Wrong approach:** Split by technical layer (all CRUD in one service, all validation in another)

**Right approach:** Split by **business capability / bounded context**

```
E-commerce Domain:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Order Domain │  │Payment Domain│  │Catalog Domain│
  │  - Order     │  │ - Payment    │  │ - Product    │
  │  - OrderItem │  │ - Refund     │  │ - Category   │
  │  - Cart      │  │ - Invoice    │  │ - Review     │
  └──────────────┘  └──────────────┘  └──────────────┘
Each domain = one service, owns its data, speaks its own language

### Code Explained (Bounded Context)
- Order Service: owns Order, OrderItem, OrderStatus enums — all order state.
- Payment Service: owns Payment, PaymentStatus, RefundRequest — knows nothing of Orders.
- Each service defines its own `Customer` entity with only relevant fields.
- They synchronize via events, not shared domain model.

### Real Usage
- Order Service: customer shipping address, items, total.
- Payment Service: customer card, billing address, transaction ID.
- Same "customer" in two different models. They sync only via `customerId`.
```

### Bounded Context — Why It Matters
The word "Customer" means different things in different domains:
- **Order service:** Customer = { id, shippingAddress, orderId }
- **Payment service:** Customer = { id, creditCard, billingAddress }
- **Marketing service:** Customer = { id, email, preferences, segments }

Same word, different models. Bounded Context says: each service has its own definition. They share only an ID, not a schema.

### Warning Signs of Wrong Service Boundaries
- Services that always deploy together (tight coupling)
- Chatty communication — A calls B 10 times per request (should be one service)
- Shared database between two "separate" services (not actually separate)
- One team owns multiple services that always change together

### Interview Answer
> "Service boundaries come from business domains, not technical layers. Order service owns the complete order lifecycle: create, status updates, cancellation. Not: 'Orders service' (CRUD) + 'OrderWorkflow service' (state machine).
>
> Warning sign: if service A calls service B more than 3-5 times per request, they're probably the same service. If two services always deploy together, they're too tightly coupled.
>
> Bounded Context is key: 'Customer' in Order service is different from 'Customer' in Payment service. They share only the ID. Own the definition in your domain.""

---

## 18. API Versioning Strategies

```java
// Strategy 1: URL path versioning — most visible, most common
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { }  // breaking changes go in v2

// Strategy 2: Header versioning — cleaner URLs
@GetMapping(value = "/orders", headers = "X-API-Version=1")
public OrderV1Response getOrdersV1() { }

@GetMapping(value = "/orders", headers = "X-API-Version=2")
public OrderV2Response getOrdersV2() { }

// Strategy 3: Content negotiation
@GetMapping(value = "/orders",
    produces = "application/vnd.myapp.v1+json")
public OrderV1Response getV1() { }

@GetMapping(value = "/orders",
    produces = "application/vnd.myapp.v2+json")
public OrderV2Response getV2() { }
```

### Rules for Backward-Compatible Changes (Non-Breaking)
```java
// SAFE: Add new optional fields (existing clients ignore them)
// SAFE: Add new endpoints
// SAFE: Add new optional request params

// BREAKING — needs new version:
// - Remove a field
// - Rename a field
// - Change a field type (String → Integer)
// - Change URL structure
// - Change authentication mechanism

### Code Explained (Versioning Strategies)
- `@RequestMapping("/api/v1/orders")` — most explicit, in URL, routable in gateway.
- `headers = "X-API-Version=2"` — cleaner URLs if versions rarely queried together.
- `vnd.myapp.v2+json` — content negotiation, most RESTful but less common.

### Real Usage
- Path versioning in gateway for easy routing and monitoring.
- New clients use v2, old clients stay on v1.
- Deprecation header added 6 months before v1 sunset.
- Consumer-driven contract tests catch silent breaking changes.
```

### Interview Answer
> "URL path versioning (`/api/v1/`, `/api/v2/`) is my default — visible, easy to test, easy to route in the API Gateway. The rule is: backward-compatible changes (new optional fields, new endpoints) are free. Breaking changes (rename/remove/type change) always get a new version. I maintain N and N-1 — deprecate N-1 with 6 months notice and sunset header. Consumer-driven contract tests catch breaking changes automatically before deployment."
> "URL path versioning (`/api/v1/`, `/api/v2/`) is my default — visible in logs, easy to route in gateway, easy to test. The rule is: backward-compatible changes (add optional fields, add endpoints) stay in v1. Breaking changes (remove/rename/type change) require v2.
>
> I maintain N and N-1 in parallel. When launching v2, deprecation header is added to v1. Six months notice, then v1 shuts down. Clients have time to migrate.
>
> Consumer-driven contract tests catch silent breaking changes before they ship. If my response changes in a way that breaks Order Service's expectations, the test fails in my CI/CD — contract violations blocked from deployment.""

---

## 19. Graceful Shutdown

### Why It Matters
Without graceful shutdown:
- In-flight requests get `Connection reset` error
- Kafka consumers stop mid-processing → duplicate processing on restart
- DB transactions get rolled back unexpectedly
- Kubernetes kills pod → users see errors

### How Spring Boot Handles It

```yaml
# application.yml
server:
  shutdown: graceful                    # wait for in-flight requests to complete
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s    # max 30s to finish in-flight requests
```

```java
// Custom cleanup — runs before Spring shuts down
@Component
public class KafkaConsumerShutdownHandler implements DisposableBean {
    @Autowired private KafkaListenerEndpointRegistry registry;

    @Override
    public void destroy() {
        // Stop accepting new messages — finish processing current batch
        registry.getListenerContainers()
            .forEach(container -> container.stop(() ->
                log.info("Kafka listener stopped gracefully")));
    }
}

// For async thread pools — wait for running tasks to complete
@Bean(destroyMethod = "shutdown")
public ExecutorService taskExecutor() {
    return Executors.newFixedThreadPool(10);
    // Spring calls shutdown() on context close → pool rejects new tasks,
    // waits for in-flight tasks to complete
}
```

### Kubernetes Graceful Shutdown
```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60   # K8s waits 60s before SIGKILL
      containers:
        - name: order-service
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # allow load balancer to drain
```

### Shutdown Sequence (Spring Boot + K8s)
```
1. K8s sends SIGTERM → Spring starts graceful shutdown
2. Readiness probe returns DOWN → K8s stops routing new requests to this pod
3. Spring waits for in-flight requests to complete (up to 30s)
4. @PreDestroy methods run → close DB connections, stop Kafka consumers
5. Spring context closes, JVM exits
6. K8s receives clean exit → marks pod terminated
```

---

## 20. Contract Testing — Consumer-Driven Contracts

### The Problem
Service A consumes Service B's API. Service B team changes their response (removes a field). Neither team knows. Service A breaks in production.

### Consumer-Driven Contract Testing (Pact / Spring Cloud Contract)

```
Consumer (Order Service) defines the CONTRACT:
  "I expect Payment Service to return { paymentId, status, amount }"

Contract is shared with Producer (Payment Service).

Producer runs contract verification tests:
  "Does my actual API response match what Order Service expects?"

If Payment Service removes 'amount' from response:
  → Contract test FAILS in Payment Service CI/CD pipeline
  → Payment Service blocked from deploying
  → Bug caught BEFORE production, not after
```

```java
// Consumer side — define what you expect
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service")
class OrderServiceContractTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        return builder
            .given("payment 123 exists")
            .uponReceiving("get payment 123")
            .path("/api/payments/123")
            .method("GET")
            .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody()
                .stringType("paymentId", "pay-123")
                .stringType("status", "SUCCESS")
                .decimalType("amount", 500.00))  // ORDER SERVICE needs this field
            .toPact();
    }
}

// Producer side — verify your API matches all consumer contracts
// Runs automatically in CI — fails if contract is broken
```

### Why This Matters at Senior Level
> Removes the need for a shared test environment where all microservices deploy together. Each service can deploy independently and know — before production — that no contracts are broken. This is how you achieve true independent deployment in microservices.

---

## 21. Service Mesh — Istio/Sidecar

### Concept
A service mesh adds a **sidecar proxy** (Envoy) next to every service pod. All traffic goes through the sidecar — retry, mTLS, tracing, circuit breaking happen at the infrastructure level, not in application code.

### Simple Explanation
Without service mesh: your application code handles retries, mTLS, telemetry, circuit breaking — every service reimplements this.

With service mesh: a sidecar process runs alongside your app. All network traffic goes through it. It handles everything transparently — your app code stays simple, focused on business logic.

```
Without Mesh:                    With Mesh (Istio + Envoy sidecar):
  Service A (Java code):          Service A (Java code):
    - Retry logic                   - Just business logic
    - mTLS setup
    - Circuit breaker             Envoy Sidecar (infrastructure):
    - Metrics collection            - Retry / circuit breaking
    - Tracing headers               - mTLS encryption
                                    - Metrics / distributed traces
                                    - Rate limiting
                                    - Traffic shifting (canary)
```

### When to Use Service Mesh
- ✅ Polyglot teams (Java + Python + Go) — can't use Resilience4j everywhere
- ✅ Zero-trust security — mTLS between all services automatically
- ✅ Progressive delivery — route 5% of traffic to new version with zero code change
- ✅ Uniform observability — all services get tracing/metrics without code changes
- ❌ Small team / simple architecture — adds significant operational complexity
- ❌ When Resilience4j + Spring Cloud suffices

### Interview Answer
> "Service mesh offloads networking concerns into a sidecar proxy. In a polyglot environment where you can't standardise on Spring/Resilience4j, Istio gives you consistent retry, circuit breaking, and mTLS across Java, Python, and Go services with zero code changes. I've used it primarily for the security guarantee — automatic mTLS means every service-to-service call is encrypted and mutually authenticated without any application code. The operational cost is real though — Envoy sidecar resource overhead, complex debugging. Worth it at scale; overkill for small architectures."

---

## 22. Microservices Patterns Quick Reference

| Pattern | Problem Solved | Key Detail |
|---|---|---|
| **API Gateway** | Single entry point, cross-cutting | Auth, rate limit, routing, correlation ID. Spring Cloud Gateway. |
| **Service Discovery** | Dynamic service location | Eureka: heartbeat 30s, cache 30s, AP system. K8s: CoreDNS. |
| **Circuit Breaker** | Cascade failure prevention | CLOSED→OPEN→HALF-OPEN. ignoreExceptions for business errors. Monitor state. |
| **Bulkhead** | Thread pool isolation | Separate pool per downstream. Slow service ≠ app-wide degradation. |
| **Retry + Backoff** | Transient failure handling | Exponential + jitter. Only retry idempotent ops. |
| **Timeout** | Thread starvation prevention | Fail fast. Never wait forever. 2-3s for most services. |
| **Saga** | Distributed transactions | Local txs + compensations. Orchestration > Choreography for complex flows. Compensations must be idempotent. |
| **CQRS** | Read/write mismatch | Separate models. Read = pre-joined flat. Eventual consistency on reads. |
| **Outbox** | Dual-write atomicity | Write to outbox in same DB tx. Poller publishes to Kafka. |
| **Aggregator** | Parallel multi-service calls | `CompletableFuture.allOf()`. Per-service fallback. Combined timeout. |
| **Strangler Fig** | Monolith migration | Route new features to new service. Old in monolith until migrated. |
| **Sidecar/Service Mesh** | Cross-cutting at infra level | Envoy proxy. Best for polyglot. mTLS between services. |
| **BFF** | Client-specific aggregation | Mobile BFF lightweight, Web BFF rich. Shapes data per consumer. |

---

## 23. High-Level Architecture Template Answer

Use this structure when asked "describe your microservices architecture":

```
"In my project — e-commerce platform with 6 core services:
  User, Product, Order, Payment, Inventory, Notification

INGRESS:
  External traffic → AWS ALB (SSL termination, external LB)
  → Spring Cloud Gateway (JWT validation, rate limiting, routing, correlation IDs)

SERVICE DISCOVERY:
  All services register with Eureka on startup
  FeignClient uses Spring Cloud LB for client-side load balancing

COMMUNICATION:
  Synchronous: FeignClient for real-time queries (stock check, auth check)
  Asynchronous: Kafka for post-order events (payment, notification, analytics)
    - order-placed topic: Payment, Notification, Analytics all consume independently
    - Outbox pattern prevents dual-write inconsistency

RESILIENCE (every inter-service call):
  Circuit Breaker (Resilience4j) + Retry (exp backoff) + Timeout + Bulkhead + Fallback

DATA:
  Each service owns its DB — no shared schemas
  Order → PostgreSQL, Product → MongoDB, Inventory → Redis (fast stock counts)
  Cross-service queries: Aggregator pattern or CQRS read models

OBSERVABILITY:
  Micrometer Tracing → Zipkin (distributed traces, 10% sampling in prod)
  Prometheus + Grafana (metrics dashboards — latency, error rate, circuit state)
  ELK (centralised logs, searchable by trace ID)
  Actuator (health probes for K8s liveness/readiness)

DEPLOYMENT:
  Docker containers → Kubernetes (EKS)
  Helm charts for service config
  CI/CD: GitHub Actions → Docker build → Helm deploy
  Canary via Gateway weighted routing (90/10 split)
  Graceful shutdown: terminationGracePeriodSeconds=60

SECURITY (between services):
  JWT validated at Gateway — downstream services trust Gateway header
  Secrets in AWS Secrets Manager, injected via @RefreshScope"
```

---

## 24. HTTP Client Selection Playbook (2026)

### Simple Explanation
Think of this as choosing the right vehicle:
- `FeignClient` = company shuttle between office buildings (internal services)
- `RestClient` = personal car for external trips (third-party APIs)
- `WebClient` = metro system for massive parallel traffic (reactive)

### Practical Decision Matrix

| Scenario | Recommended Client | Why |
|---|---|---|
| Internal service call with discovery (`lb://`) | FeignClient | Declarative + service name + easy resilience wiring |
| External partner API (sync flow) | RestClient | Modern API, cleaner code, better than RestTemplate |
| High-concurrency non-blocking I/O | WebClient | Reactive and resource-efficient |
| Existing old module using RestTemplate | Keep short-term, migrate gradually | Minimize risky rewrites |

### Migration Guidance
- New code: never start with RestTemplate.
- Replace RestTemplate first in high-change modules.
- Keep interface contracts stable while migrating implementation.

---

## 25. Microservices Security End-to-End

### Request Flow (North-South + East-West)
```
Client → API Gateway (JWT validation, RBAC, rate limit)
      → Order Service (business authz checks)
      → Payment Service (service identity + scoped permissions)
```

### Core Practices
- Authenticate at gateway, authorize at both gateway and service boundaries.
- Propagate identity claims safely (`sub`, `roles`, `scope`) — never trust raw headers from outside.
- Use short-lived access tokens and rotate signing keys.
- Use mTLS for service-to-service traffic in zero-trust environments.
- Store secrets in vault manager (AWS Secrets Manager / HashiCorp Vault), never in git.

### Spring Security Resource Server (JWT)
```java
@Bean
SecurityFilterChain security(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt())
        .build();
}
```

### Interview Answer
> "I secure microservices in layers: gateway validates JWT and enforces coarse policies, each service still enforces domain-level authorization, and service-to-service traffic uses identity-based trust (mTLS/service identity). Tokens are short-lived, keys rotated, and secrets come from a secret manager. This prevents both perimeter and lateral-movement attacks."

---

## 26. Centralized Configuration (Spring Cloud Config)

### Why It Exists
Without centralized config, each service has duplicated environment config and risky manual updates.

### Flow
```
Config Repo (Git) → Config Server → Microservices (at startup / refresh)
```

### Minimal Setup
```yaml
# client service bootstrap/application config
spring:
  application:
    name: order-service
  config:
    import: "optional:configserver:http://config-server:8888"
```

### Runtime Refresh
- Use `/actuator/refresh` for selected dynamic properties (when appropriate).
- Prefer immutable config for critical properties (db url, topic names) and restart rollout for safety.

### Failure Strategy (Important)
- Fail-fast for critical config in production.
- Keep sensible defaults for non-critical feature flags.
- Version config with application release to avoid drift.

---

## 27. Schema Evolution & Zero-Downtime DB Migrations

### Expand → Migrate → Contract Pattern
1. **Expand**: Add new nullable column / new table (backward compatible)
2. **Migrate**: Deploy app writing old+new fields, backfill data
3. **Contract**: Remove old field only after all services switched

### Rules That Prevent Outages
- Never do destructive schema change first.
- Keep API backward compatible during migration window.
- Use feature flags to switch readers/writers gradually.
- Always have rollback-safe migrations.

### Tooling
- Use Flyway/Liquibase in CI/CD.
- Review slow queries with `EXPLAIN` before and after schema changes.

### Interview Answer
> "For zero-downtime schema changes I use expand-migrate-contract. First deploy additive schema, then dual-write/read, then remove legacy fields after traffic confirms full migration. This avoids breaking mixed-version deployments."

---

## 28. Observability Runbook — Metrics, Alerts, Incident Flow

### Golden Signals
- Latency (p50/p95/p99)
- Traffic (RPS)
- Errors (5xx rate)
- Saturation (CPU, memory, thread pools, queue lag)

### Minimum Alert Set
- p99 latency breach for 5+ min
- Error rate > threshold (e.g., 2-5%)
- Circuit breaker OPEN state sustained
- Kafka consumer lag growth beyond threshold
- Pod restart spike / readiness failures

### Incident Triage Flow
1. Check alert dashboard (which service and when)
2. Open trace for affected request IDs
3. Correlate logs by traceId/correlationId
4. Confirm dependency issue (db, downstream API, broker)
5. Mitigate (rollback, scale, disable feature flag), then fix root cause

### Interview Answer
> "I treat observability as an operations workflow, not just tools. Metrics detect, traces localize, logs explain. My runbook is standardized so on-call engineers can move from alert to mitigation quickly and consistently."

---

## Quick Interview Checklist

Before any microservices interview, make sure you can explain:

- [ ] Why you'd choose microservices (and when you wouldn't)
- [ ] How a request flows end-to-end (Gateway → Eureka → FeignClient → Service)
- [ ] How you'd handle a service going down (circuit breaker + fallback)
- [ ] How distributed transactions work (Saga + compensations)
- [ ] Why you can't just use a shared DB (and how you handle cross-service queries)
- [ ] How you'd debug an issue in production (trace ID + ELK + Zipkin)
- [ ] What happens when you deploy 3 instances and a Kafka consumer runs 3x (idempotency)
- [ ] How you ensure an event is never lost after DB save (outbox pattern)
- [ ] How you do a zero-downtime deploy (canary with weighted routing)
- [ ] High-level architecture of your actual project (use template above)
- [ ] Which HTTP client to choose now (FeignClient vs RestClient vs WebClient vs RestTemplate)
- [ ] End-to-end security flow (Gateway JWT, service authz, mTLS, secret management)
- [ ] Config strategy (Config Server, refresh boundaries, fail-fast policy)
- [ ] Zero-downtime schema evolution plan (expand-migrate-contract)
- [ ] Observability runbook (alerts, triage flow, mitigation path)
