# Topic 12: Spring Ecosystem

## Q57. Spring Actuator — Purpose, Benefits, Endpoints

### Concept
Spring Boot Actuator exposes production-ready endpoints to monitor and manage your running application — health, metrics, environment, thread dumps — without writing any monitoring code yourself.

### Simple Explanation
Your car dashboard shows speed, fuel, engine temp — you don't wire those sensors yourself, the car ships with them. Actuator is your Spring Boot dashboard — it ships pre-wired and tells you exactly what's happening inside the running app.

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, env, loggers, threaddump, httptrace
        # include: "*"  — expose all (only in internal/monitored environments)
  endpoint:
    health:
      show-details: always   # show component-level health (DB, cache, disk)
  metrics:
    export:
      prometheus:
        enabled: true        # expose /actuator/prometheus for Prometheus scraping
```

### Key Endpoints

| Endpoint | URL | What it shows |
|---|---|---|
| `/actuator/health` | GET | UP/DOWN status + component health (DB, Redis, disk) |
| `/actuator/info` | GET | App version, build info, git commit |
| `/actuator/metrics` | GET | All available metric names |
| `/actuator/metrics/{name}` | GET | Specific metric value (e.g. JVM heap, HTTP latency) |
| `/actuator/env` | GET | All config properties and their sources |
| `/actuator/loggers` | GET/POST | View and change log levels at runtime |
| `/actuator/threaddump` | GET | All thread states — useful for deadlock diagnosis |
| `/actuator/heapdump` | GET | Download heap dump for memory analysis |
| `/actuator/prometheus` | GET | Metrics in Prometheus format for scraping |
| `/actuator/refresh` | POST | Reload `@RefreshScope` beans (Spring Cloud) |
| `/actuator/shutdown` | POST | Graceful shutdown (disabled by default) |

### Custom Health Indicator
```java
// Add your own component to the /health endpoint
@Component
public class PaymentServiceHealthIndicator implements HealthIndicator {

    @Autowired private PaymentGatewayClient client;

    @Override
    public Health health() {
        try {
            boolean reachable = client.ping();
            if (reachable) {
                return Health.up()
                    .withDetail("gateway", "Stripe")
                    .withDetail("latency", "12ms")
                    .build();
            }
            return Health.down().withDetail("reason", "Stripe ping failed").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
// Result in /actuator/health:
// "paymentService": { "status": "UP", "details": { "gateway": "Stripe", "latency": "12ms" } }
```

### Custom Metrics
```java
// Register custom business metrics
@Service
public class OrderService {

    private final Counter ordersPlaced;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry registry) {
        this.ordersPlaced = registry.counter("orders.placed", "env", "prod");
        this.orderProcessingTime = registry.timer("orders.processing.time");
    }

    public void placeOrder(Order order) {
        orderProcessingTime.record(() -> {
            processOrder(order);
            ordersPlaced.increment();
        });
    }
}
// Accessible at: /actuator/metrics/orders.placed
```

### Securing Actuator Endpoints
```java
// Never expose actuator publicly — secure it
@Configuration
public class ActuatorSecurityConfig {
    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        return http
            .requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .anyRequest().hasRole("ACTUATOR_ADMIN"))
            .httpBasic(withDefaults())
            .build();
    }
}
```

### Interview Answer
> "Actuator gives you production visibility without writing any monitoring code. I expose `/health` (used by Kubernetes liveness/readiness probes), `/metrics` (scraped by Prometheus, visualised in Grafana), and `/loggers` (lets me change log level to DEBUG on a running prod instance without restart — invaluable for diagnosing live issues).
>
> In production I only expose what's needed and always secure actuator endpoints behind a role — `/health` and `/info` can be public, everything else requires auth. I also write custom `HealthIndicator` beans for critical external dependencies like payment gateways and message brokers so our ops dashboard shows exactly which component is degraded.
>
> **Follow-up:** *How does Kubernetes use Actuator?*
> — Liveness probe hits `/actuator/health/liveness` — if DOWN, Kubernetes restarts the pod. Readiness probe hits `/actuator/health/readiness` — if DOWN, Kubernetes stops sending traffic to that pod but doesn't restart it."

---

## Q58. Spring Batch — Purpose and Real Scenario

### Concept
Spring Batch is a framework for processing large volumes of data in **chunks** — read, process, write — with built-in restart/retry, skip logic, and job tracking.

### Simple Explanation
You have 10 million customer records to migrate from old DB to new DB. You can't load all 10M into memory. You can't fail halfway and not know where you stopped. Spring Batch solves both: it reads 1000 at a time (chunks), processes them, writes them, records progress — if it crashes at record 500,000, it restarts at 500,001, not from the beginning.

### Core Concepts

```
Job
  └─ Step (one or more)
       ├─ ItemReader     — reads data (DB, file, CSV, API)
       ├─ ItemProcessor  — transforms/validates each item (optional)
       └─ ItemWriter     — writes results (DB, file, API)
```

```
Chunk-oriented processing:
  Read 1000 → Process 1000 → Write 1000 → COMMIT (checkpoint)
  Read 1000 → Process 1000 → Write 1000 → COMMIT (checkpoint)
  ...repeat until done
  
  If crash at chunk 50 → restart from chunk 51, not chunk 1
```

### Code — Real Scenario: Monthly Salary Processing
```java
@Configuration
@EnableBatchProcessing
public class SalaryBatchConfig {

    // 1. ItemReader — read employees from DB in chunks
    @Bean
    public JpaPagingItemReader<Employee> employeeReader(EntityManagerFactory emf) {
        return new JpaPagingItemReaderBuilder<Employee>()
            .name("employeeReader")
            .entityManagerFactory(emf)
            .queryString("SELECT e FROM Employee e WHERE e.active = true")
            .pageSize(500)   // read 500 at a time
            .build();
    }

    // 2. ItemProcessor — calculate salary, apply tax
    @Bean
    public ItemProcessor<Employee, SalaryRecord> salaryProcessor() {
        return employee -> {
            if (employee.getSalary() == null) return null; // null = skip this item
            BigDecimal tax = employee.getSalary().multiply(BigDecimal.valueOf(0.2));
            return new SalaryRecord(employee.getId(),
                employee.getSalary().subtract(tax), LocalDate.now());
        };
    }

    // 3. ItemWriter — write salary records to DB
    @Bean
    public JpaItemWriter<SalaryRecord> salaryWriter(EntityManagerFactory emf) {
        JpaItemWriter<SalaryRecord> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(emf);
        return writer;
    }

    // 4. Step — wire reader, processor, writer with chunk size
    @Bean
    public Step salaryStep(JobRepository jobRepository,
                           PlatformTransactionManager txManager) {
        return new StepBuilder("salaryStep", jobRepository)
            .<Employee, SalaryRecord>chunk(500, txManager)  // commit every 500
            .reader(employeeReader(null))
            .processor(salaryProcessor())
            .writer(salaryWriter(null))
            .faultTolerant()
            .skip(DataIntegrityViolationException.class)  // skip bad records
            .skipLimit(10)                                 // but max 10 skips
            .retry(TransientDataAccessException.class)    // retry transient DB errors
            .retryLimit(3)
            .build();
    }

    // 5. Job
    @Bean
    public Job salaryProcessingJob(JobRepository jobRepository, Step salaryStep) {
        return new JobBuilder("salaryProcessingJob", jobRepository)
            .start(salaryStep)
            .listener(new JobExecutionListener() {
                @Override
                public void afterJob(JobExecution jobExecution) {
                    log.info("Job status: {}, records processed: {}",
                        jobExecution.getStatus(),
                        jobExecution.getStepExecutions().iterator().next().getWriteCount());
                }
            })
            .build();
    }
}

// 6. Trigger job (via scheduler or API)
@Service
public class BatchTriggerService {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job salaryProcessingJob;

    public void triggerSalaryJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLocalDateTime("runTime", LocalDateTime.now()) // unique params = new run
            .toJobParameters();
        jobLauncher.run(salaryProcessingJob, params);
    }
}
```

### Most Asked Checkpoints
- **JobRepository** — Spring Batch persists job execution state to DB; enables restart from failure point
- **Chunk vs Tasklet** — chunk for large data sets (read-process-write); `Tasklet` for single tasks (send email, delete temp files)
- **Skip vs Retry** — skip: permanently bad records (bad data); retry: transient failures (DB timeout, network blip)
- **Partitioning** — split data into partitions, process in parallel threads or multiple nodes (for very large datasets)

### Interview Answer
> "I've used Spring Batch for two main scenarios: large data migrations and monthly financial processing. The key value is chunk-oriented processing with checkpointing — process 500 records, commit, record progress. If the job fails at record 2 million, it restarts from the last committed checkpoint, not from zero.
>
> The built-in skip/retry is critical for production. Skip handles permanently bad records — log them, move on. Retry handles transient failures — DB connection timeout, retry 3 times before failing. Without this you'd need to write all that error handling yourself.
>
> Spring Batch tracks every job execution in its `JobRepository` tables — you get full audit history of what ran, when, how many records processed, and the exact failure point.
>
> **Follow-up:** *How would you run a batch job faster?*
> — Partitioning: split the dataset by range (IDs 1–1M to partition A, 1M–2M to partition B) and process partitions in parallel threads or across multiple nodes."

---

## Q59. Spring Scheduler — Use Cases

### Concept
Spring Scheduler runs methods on a **fixed schedule** (cron, fixed rate, fixed delay) within the same JVM — no external job scheduler needed for simple periodic tasks.

### Simple Explanation
It's like setting a recurring alarm. "Every day at 2 AM, run this method." Spring wakes it up, runs it, goes back to sleep.

### Setup & Patterns
```java
@Configuration
@EnableScheduling
public class SchedulerConfig { }

@Component
public class ScheduledTasks {

    // Fixed rate — every 5 seconds regardless of execution time
    // (next run starts 5s after previous START)
    @Scheduled(fixedRate = 5000)
    public void pollExternalApi() {
        // check for new orders from partner API
    }

    // Fixed delay — 5 seconds AFTER previous execution ENDS
    // safer for tasks that might take variable time
    @Scheduled(fixedDelay = 5000)
    public void cleanupTempFiles() { }

    // Cron — full cron expression
    @Scheduled(cron = "0 0 2 * * ?")   // 2 AM every day
    public void generateDailyReport() { }

    @Scheduled(cron = "0 0 0 1 * ?")   // midnight on 1st of every month
    public void processMonthlyBilling() { }

    // With initial delay — wait 10s after startup before first run
    @Scheduled(fixedRate = 60000, initialDelay = 10000)
    public void syncWithCRM() { }
}
```

### Real Use Cases
| Use Case | Schedule | Why |
|---|---|---|
| Cache refresh | Every 15 min | Keep local cache warm with latest reference data |
| Expired token cleanup | Every hour | Delete expired JWTs/sessions from DB |
| Retry failed notifications | Every 5 min | Re-attempt failed email/SMS sends |
| Daily report generation | 1 AM cron | Reports ready when users log in at 9 AM |
| Health check on partner APIs | Every 30s | Alert before customers notice |
| DB archive old records | Weekly Sunday 3 AM | Move old orders to archive table |

### Important: Scheduler in Clustered Environments
```
Problem: App deployed on 3 nodes → same @Scheduled job runs 3x simultaneously
         → 3 duplicate report emails, 3x DB writes

Solutions:
1. ShedLock — distributed lock in DB/Redis, only one node runs at a time
2. Spring Batch (for heavy jobs) — built-in single-execution guarantee
3. Quartz Scheduler — cluster-aware job scheduler with DB-backed coordination
```

```java
// ShedLock — simplest fix for clustered schedulers
@Scheduled(cron = "0 0 2 * * ?")
@SchedulerLock(name = "dailyReport", lockAtMostFor = "30m", lockAtLeastFor = "1m")
public void generateDailyReport() {
    // Only one node acquires the lock — others skip
}
```

### Interview Answer
> "Spring Scheduler is my go-to for simple recurring tasks within a single application — cache warming, cleanup jobs, periodic syncs. I use fixed delay for tasks with variable execution time (so they don't overlap) and cron for time-specific windows.
>
> The critical production concern is clustering: if you deploy 3 instances and all have `@Scheduled`, all 3 run the job simultaneously — duplicate emails, duplicate writes. I solve this with ShedLock, which uses a DB/Redis lock so exactly one node runs the job. For heavy batch workloads, I move it to Spring Batch which has proper cluster-safe execution.
>
> **Follow-up:** *Difference between `fixedRate` and `fixedDelay`?*
> — `fixedRate`: next run starts X ms after previous run STARTED — can overlap if execution takes longer than the rate. `fixedDelay`: next run starts X ms after previous run ENDED — guaranteed gap between executions. For tasks that hit a DB or external API, `fixedDelay` is safer."

---

## Q60. Caching — Application Level vs Redis, Eviction & Refresh

### Concept
Caching stores frequently accessed data in fast memory so repeated reads skip expensive DB/API calls — trading memory for speed.

### Simple Explanation
You're an accountant asked the same question 100 times a day: "What's the company tax rate?" Instead of looking it up in the filing cabinet (DB) every time, you write it on a sticky note on your desk (cache). Much faster. But if the tax rate changes, you must update or throw away the sticky note (eviction/refresh).

### Application-Level Cache vs Redis

| | Application Cache (Caffeine) | Redis |
|---|---|---|
| Location | Inside JVM heap | External server |
| Speed | Fastest (nanoseconds) | Fast (network: ~1ms) |
| Shared across instances? | ❌ No — each node has its own | ✅ Yes — all nodes share one cache |
| Survives restart? | ❌ No | ✅ Yes (if persistence configured) |
| Use for | Single-instance or read-heavy reference data | Multi-instance / session / rate limiting |
| Size limit | JVM heap constrained | Configurable, much larger |

### Spring Cache Abstraction
```java
// 1. Enable caching
@SpringBootApplication
@EnableCaching
public class App { }

// 2. Configure Caffeine (application-level, in-process)
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)               // max 1000 entries per cache
        .expireAfterWrite(15, MINUTES)   // TTL: expire 15 min after write
        .recordStats());                 // enable hit/miss metrics
    return manager;
}

// 3. Configure Redis (distributed, cross-instance)
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(30))           // 30 min TTL
        .disableCachingNullValues()                  // don't cache null responses
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    return RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .build();
}
```

### Cache Annotations
```java
@Service
public class ProductService {

    // @Cacheable — check cache first; call method only on cache miss
    // key built from method params: "products::ELECTRONICS"
    @Cacheable(value = "products", key = "#category",
               condition = "#category != null",      // only cache if category not null
               unless = "#result.isEmpty()")         // don't cache empty results
    public List<Product> getByCategory(String category) {
        return productRepo.findByCategory(category); // DB hit only on cache miss
    }

    // @CachePut — ALWAYS calls method AND updates cache
    // use for write operations to keep cache in sync
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepo.save(product);
    }

    // @CacheEvict — remove from cache
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(Long productId) {
        productRepo.deleteById(productId);
    }

    // @CacheEvict — clear entire cache (e.g. bulk import)
    @CacheEvict(value = "products", allEntries = true)
    public void bulkImport(List<Product> products) {
        productRepo.saveAll(products);
    }

    // @Caching — multiple cache operations on one method
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#product.id"),
        @CacheEvict(value = "categories", key = "#product.category")
    })
    public void updateProductWithCategory(Product product) {
        productRepo.save(product);
    }
}
```

### Cache Refresh Strategies
```java
// Strategy 1: TTL-based expiry (simplest)
// Entry expires after X minutes → next read triggers DB call + re-caches
.expireAfterWrite(15, MINUTES)

// Strategy 2: Event-driven eviction (most accurate)
@TransactionalEventListener  // fires AFTER the DB transaction commits
public void onProductUpdated(ProductUpdatedEvent event) {
    cacheManager.getCache("products").evict(event.getProductId());
}

// Strategy 3: Scheduled refresh (for reference data that rarely changes)
@Scheduled(cron = "0 0 */4 * * ?")  // every 4 hours
@CacheEvict(value = "taxRates", allEntries = true)
public void refreshTaxRateCache() {
    log.info("Tax rate cache cleared — will reload on next access");
}

// Strategy 4: Cache-aside with manual reload
@Scheduled(fixedRate = 300_000)  // every 5 minutes
public void warmUpCache() {
    // proactively reload before TTL expires — prevents cache stampede
    List<String> popularCategories = List.of("ELECTRONICS", "CLOTHING", "FOOD");
    popularCategories.forEach(cat -> getByCategory(cat)); // triggers cache reload
}
```

### Cache Stampede Prevention
```java
// Problem: 1000 threads all get cache miss simultaneously → 1000 DB queries
// Solution 1: Caffeine's refreshAfterWrite (async refresh, serves stale while refreshing)
Caffeine.newBuilder()
    .expireAfterWrite(10, MINUTES)
    .refreshAfterWrite(8, MINUTES)  // async refresh at 8 min, stale served till 10 min
    .build();

// Solution 2: Redis with probabilistic early expiration (complex, for very high traffic)
```

### Interview Answer
> "I use two levels of caching based on the scenario. Application-level cache (Caffeine) for within-JVM, single-node fast lookups — reference data like product categories or tax rates that all requests on that node need. Redis for anything that must be consistent across multiple instances — user sessions, rate limiting counters, shared lookup tables.
>
> For eviction I choose based on how stale data impacts the user. For financial data — event-driven eviction the moment the DB record changes. For product catalogue — TTL of 15 minutes is acceptable; minor staleness doesn't hurt. For reference data that never changes in prod (country codes, currencies) — I warm up on startup and use a 24h TTL.
>
> One real incident: we cached search results without a size limit. Memory pressure caused GC pauses that degraded the entire service. Now I always set `maximumSize` in Caffeine and monitor hit/miss ratio via Actuator metrics.
>
> **Follow-up:** *What is a cache stampede and how do you prevent it?*
> — When a popular cache entry expires and hundreds of threads simultaneously get a cache miss, all hitting the DB at once. Fix: `refreshAfterWrite` (Caffeine serves stale while async refreshing), or a distributed lock so only one thread refreshes while others wait for the new value."

---

## Q61. What Data Should and Should NOT Be Cached — Real Wrong-Caching Example

### What TO Cache
| Type | Example | Why |
|---|---|---|
| Expensive, rarely changing reads | Product catalogue, exchange rates | High DB cost, low change frequency |
| Reference / lookup data | Country codes, tax rates, roles | Read 1000x, write once |
| Computed aggregations | Dashboard totals, report summaries | Expensive to recompute each time |
| External API responses | Currency conversion, geolocation | Rate limited or slow external calls |
| Session data | Auth tokens, user preferences | Avoid DB lookup per request |

### What NOT to Cache
| Type | Why |
|---|---|
| Frequently changing data | Account balance, stock quantity — stale within seconds |
| User-specific sensitive data carelessly | Risk of cross-user data leakage if key is wrong |
| Large payloads per user | Memory exhaustion — 10k users × 500KB each = 5GB cache |
| Write-heavy data | Cache invalidation cost > cache hit benefit |
| Data with regulatory freshness requirements | Financial/medical data that must always be real-time |

### Real Wrong-Caching Incident
```java
// ❌ Real incident: caching account balance
@Cacheable(value = "accountBalance", key = "#accountId")
public BigDecimal getBalance(Long accountId) {
    return accountRepo.findById(accountId).getBalance();
}

// What happened:
// 1. User A checked balance: ₹10,000 → cached
// 2. User A transferred ₹8,000 out → DB updated to ₹2,000
// 3. Cache NOT evicted (eviction was on @CacheEvict in transfer() — but that method
//    was called via this.transfer() inside the same bean — SELF INVOCATION!
//    Proxy bypassed → @CacheEvict never fired)
// 4. User A checked balance again → cache returned ₹10,000 → STALE
// 5. User A transferred again → bank showed wrong available balance

// ✅ Fix:
// 1. Never cache account balances — always read from DB
// 2. Or: use event-driven eviction via @TransactionalEventListener (separate bean)
// 3. Or: add readOnly=false query with a DB row lock for balance checks
```

### Interview Answer
> "The rule I follow: cache data that is read far more than it's written, and where serving slightly stale data is acceptable. Product catalog, reference data, computed reports — yes. Account balances, inventory stock levels, anything financial or transactional — no.
>
> A real incident from my project: we cached account balances to reduce DB load. The eviction logic used `@CacheEvict` inside the same service class — self-invocation bypassed the proxy and the eviction never fired. Users saw stale balances immediately after transfers. We fixed it by moving the eviction to a separate `@TransactionalEventListener` bean, and ultimately decided account balances should never be cached — the business risk is too high.
>
> **Follow-up:** *How do you detect wrong caching in production?*
> — Monitor cache hit ratio (should be high) and compare DB read patterns. If cache hit is 40% but cache miss causes a write-heavy query, the caching strategy is wrong. Also correlate customer complaints about stale data with cache TTL windows."

---

## Q62. Hazelcast in Spring — What It Is and How It Works

### Concept
Hazelcast is an **in-memory distributed data grid** — a cluster of JVMs that share memory, giving you a distributed cache/data store that is replicated and highly available.

### Simple Explanation
Redis is a separate server your apps talk to over the network. Hazelcast lives **inside your JVM** AND connects to other JVMs directly — all app nodes form one shared memory grid. No separate server to manage. Each node holds a shard of the data, and every other node knows where everything is.

### Redis vs Hazelcast

| | Redis | Hazelcast |
|---|---|---|
| Architecture | Separate server process | Embedded in your JVM (or separate mode) |
| Network hop | Yes (TCP to Redis server) | Minimal (peer-to-peer between nodes) |
| Setup | External infra needed | Just add dependency + auto-discovery |
| Data structures | Rich (streams, sorted sets, etc.) | Map, Queue, List, Topic, Lock, AtomicLong |
| Best for | General caching, session, pub/sub | Distributed computing, grid processing |
| Spring Boot support | `spring-boot-starter-data-redis` | `hazelcast-spring` |

### How It Works Internally

```
Node 1 (JVM)          Node 2 (JVM)          Node 3 (JVM)
┌────────────┐        ┌────────────┐        ┌────────────┐
│ Hazelcast  │◄──────►│ Hazelcast  │◄──────►│ Hazelcast  │
│ Member     │        │ Member     │        │ Member     │
│            │        │            │        │            │
│ Partition  │        │ Partition  │        │ Partition  │
│ 1-85       │        │ 86-170     │        │ 171-271    │
│            │        │            │        │            │
│ Backup of  │        │ Backup of  │        │ Backup of  │
│ 86-170     │        │ 171-271    │        │ 1-85       │
└────────────┘        └────────────┘        └────────────┘

- Data split into 271 partitions distributed evenly
- Each partition has a primary owner + backup on another node
- Any node can receive a request → routes to correct partition owner
- Node leaves → backups are promoted, repartitioning happens automatically
```

### Spring Boot Integration
```xml
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-spring</artifactId>
</dependency>
```

```java
// 1. Configure Hazelcast instance
@Bean
public Config hazelcastConfig() {
    Config config = new Config();
    config.setInstanceName("myapp-hazelcast");

    // Auto-discover other nodes (Kubernetes, multicast, TCP/IP)
    JoinConfig join = config.getNetworkConfig().getJoin();
    join.getMulticastConfig().setEnabled(false);
    join.getTcpIpConfig()
        .setEnabled(true)
        .addMember("10.0.0.1")
        .addMember("10.0.0.2");

    // Configure a distributed map (cache)
    MapConfig productCache = new MapConfig("products")
        .setTimeToLiveSeconds(900)          // 15 min TTL
        .setMaxSizeConfig(new MaxSizeConfig(10000, PER_NODE))
        .setEvictionPolicy(EvictionPolicy.LRU);

    config.addMapConfig(productCache);
    return config;
}

// 2. Use as Spring Cache provider
@Bean
public CacheManager cacheManager(HazelcastInstance instance) {
    return new HazelcastCacheManager(instance);
}

// 3. Use distributed Map directly for more control
@Service
public class SessionService {
    @Autowired private HazelcastInstance hazelcast;

    public void storeSession(String sessionId, UserSession session) {
        IMap<String, UserSession> sessions = hazelcast.getMap("userSessions");
        sessions.put(sessionId, session, 30, MINUTES);  // auto-expire in 30 min
    }

    public UserSession getSession(String sessionId) {
        return hazelcast.getMap("userSessions").get(sessionId);
        // hazelcast routes to the correct partition owner automatically
    }
}

// 4. Distributed Lock — coordinate across nodes
public void processOrderOnce(String orderId) {
    ILock lock = hazelcast.getLock("order-" + orderId);
    if (lock.tryLock(5, SECONDS, 30, SECONDS)) {
        try {
            processOrder(orderId); // only one node processes this order
        } finally {
            lock.unlock();
        }
    }
}
```

### Use Cases Where Hazelcast Shines
- **Distributed session management** — user login on Node 1, next request hits Node 2 — session found
- **Distributed caching** — same as Redis but lower latency, no separate infra
- **Distributed locking** — prevent two nodes processing same order simultaneously
- **Event/Message distribution** — `ITopic` for pub/sub across cluster nodes
- **Cluster coordination** — leader election, cluster state management

### Interview Answer
> "Hazelcast is an in-memory data grid that runs embedded in your JVM and forms a peer-to-peer cluster with other nodes — no separate server required. Data is partitioned across all 271 internal partitions and distributed evenly; each partition has a primary and a backup on a different node for fault tolerance.
>
> I've used it for distributed session storage in a horizontally scaled Spring Boot app — a user logs in, their session stored in Hazelcast; next request can hit any node and still find the session because the grid is shared.
>
> The key advantage over Redis for this use case: no network hop to a separate server — Hazelcast routes to the correct partition owner peer-to-peer, which is faster. The trade-off: Hazelcast consumes your app's JVM heap for data — size it carefully or use client-server mode where Hazelcast runs separately but still as a cluster.
>
> **Follow-up:** *When would you choose Redis over Hazelcast?*
> — When you need advanced data structures (Redis Streams, Sorted Sets), when you have polyglot services (non-Java services also need the cache), or when ops team prefers managing a known Redis cluster. Hazelcast is more compelling when you want zero extra infrastructure and your stack is Java-homogeneous."

---

## Quick Revision Card

| Topic | Key Points |
|---|---|
| Actuator | Production dashboard — health, metrics, loggers, threaddump. Secure it. Custom HealthIndicator for external deps. K8s uses liveness/readiness probes. |
| Spring Batch | Chunk-oriented: read-process-write in chunks. Checkpointing = restart from failure. Skip bad records, retry transient errors. Partition for parallel processing. |
| Spring Scheduler | @Scheduled for periodic tasks. fixedRate vs fixedDelay. MUST add ShedLock/Quartz in clustered deployments — else job runs N times. |
| Caching | @Cacheable check-then-cache. @CachePut always update. @CacheEvict remove. TTL vs event-driven eviction. Never cache account balances or write-heavy data. |
| Wrong caching | Self-invocation bypassed @CacheEvict → stale balance shown. Fix: separate bean for eviction, or @TransactionalEventListener. |
| Hazelcast | Distributed in-memory grid embedded in JVM. Partitioned + replicated. Use for distributed session, lock, cache. Redis = external server; Hazelcast = embedded peer cluster. |

---

## Additional Deep-Dive (Q57-Q62)

### Production Readiness Checklist

- Actuator endpoints are network-restricted and role-protected.
- Liveness/readiness probes are distinct and reflect real dependency states.
- Batch jobs have idempotency and restartability validated with failure simulation.
- Scheduled jobs are cluster-safe (ShedLock/Quartz), not single-node assumptions.
- Cache strategy includes explicit invalidation ownership and stale-data tolerance definition.

### Senior Interview Edge Cases

- Why `@Scheduled` can fail silently in distributed deployments:
Without cluster coordination, every pod executes the same job, creating duplicate processing.

- Why cache improves latency but can hurt correctness:
If eviction design is weak, stale data leaks into user flows. Correctness-sensitive domains should prioritize consistency over hit ratio.

- Why Actuator health should be contextual:
"UP" with DB down is misleading. Health should represent serving ability, not merely process liveliness.

### Real Project Usage

- Mature teams maintain separate dashboards for technical metrics (CPU, heap, pool) and business metrics (order success rate, payment failure rate) to avoid blind spots.
- Batch + scheduler pipelines usually include dead-letter handling and replay tooling so operations can recover data safely after partial failures.

### Spring Data Repository and Query Depth

```java
public interface OrderRepository extends JpaRepository<OrderEntity, Long> {
    List<OrderEntity> findByStatusOrderByCreatedAtDesc(String status);

    @Query("select o from OrderEntity o where o.total >= :minTotal")
    List<OrderEntity> findHighValueOrders(@Param("minTotal") BigDecimal minTotal);
}
```

Interview point: derived query methods are fast to build; custom `@Query` is preferred when intent/performance needs explicit control.

---

## Distributed Locks — Production Patterns for Coordinating Across Services

### Concept
In distributed systems, multiple service instances need to coordinate: prevent duplicate order processing, ensure only one service backfills cache, or serialize access to a limited resource. Distributed locks (typically backed by Redis, database, or Zookeeper) provide mutual exclusion across processes. Production implementations balance safety (avoiding lost locks) with performance (minimal latency).

### Simple Explanation
Imagine a shared bathhouse:
- **Single-server app**: One bathroom door lock (synchronized block). Only one person enters. Simple.
- **Multi-server app**: 100 service instances sharing one bathroom (shared resource). Local locks don't work across instances. Need a distributed lock on the door so at most one service enters.

Redis SET with NX ("set if not exists") or database lock tables are common implementations.

### Problem 1: Naive Lock — Deadlock if Service Crashes

```java
// ❌ WRONG: Lock without expiry
@Service
public class OrderService {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    public void processOrder(String orderId) {
        String lockKey = "order:lock:" + orderId;
        
        // Try to acquire lock
        while (!redis.opsForValue().setIfAbsent(lockKey, "locked")) {
            Thread.sleep(100);  // Spin-wait
        }
        
        try {
            // Process order
            // ... business logic ...
        } finally {
            redis.delete(lockKey);  // Release lock
        }
    }
}

// Problem:
// Service 1 acquires lock on order:100
// Service 1 crashes before releasing
// Lock remains in Redis forever
// Service 2 waits indefinitely for lock
// ❌ Deadlock: order:100 never processed again
```

#### ✅ Solution 1: Lock with TTL (Time-To-Live)

```java
@Service
public class OrderService {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    private static final long LOCK_TTL_SECONDS = 30;
    
    public boolean tryAcquireLock(String orderId) {
        String lockKey = "order:lock:" + orderId;
        String lockValue = UUID.randomUUID().toString();  // Unique identifier
        
        // SET key value EX expiry NX (atomic operation in Redis)
        Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, lockValue,
                Duration.ofSeconds(LOCK_TTL_SECONDS));
        
        return acquired != null && acquired;
    }
    
    public void processOrder(String orderId) throws InterruptedException {
        String lockKey = "order:lock:" + orderId;
        
        for (int retries = 0; retries < 10; retries++) {
            if (tryAcquireLock(orderId)) {
                try {
                    // Business logic
                } finally {
                    redis.delete(lockKey);  // Release lock
                }
                return;  // Success
            }
            Thread.sleep(100 * (retries + 1));  // Exponential backoff
        }
        throw new LockAcquisitionException("Failed to acquire lock for " + orderId);
    }
}

// Behavior:
// 1. Service 1 acquires lock with 30s TTL
// 2. Service 1 crashes
// 3. After 30s, lock auto-expires in Redis
// 4. Service 2 acquires lock, processes order
// ✅ No deadlock; order eventually processed
```

### Problem 2: Wrong Unlock — Releasing Another Service's Lock

```java
// ❌ WRONG: Simple delete (no verification)
public void releaseLock(String orderId) {
    String lockKey = "order:lock:" + orderId;
    redis.delete(lockKey);  // Any service can delete any lock!
}

// Scenario:
// Service 1 acquires lock, sets value "S1-uuid-123"
// Service 1 processes (slow, 60 seconds)
// Lock TTL expires after 30s
// Service 2 acquires lock, sets value "S2-uuid-456"
// Service 1 finishes, calls releaseLock()
// ❌ Deletes lock without checking! Service 2's lock is released prematurely.
// Service 3 acquires lock while Service 2 is still processing
// ❌ Two services process same order concurrently. Data corruption.
```

#### ✅ Solution 2: Verify Lock Owner Before Release

```java
@Service
public class OrderService {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    private static final long LOCK_TTL_SECONDS = 30;
    
    private String acquireAndReturnLockValue(String orderId) {
        String lockKey = "order:lock:" + orderId;
        String lockValue = UUID.randomUUID().toString();
        
        // Retry until acquired
        while (true) {
            Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, lockValue,
                    Duration.ofSeconds(LOCK_TTL_SECONDS));
            if (acquired != null && acquired) {
                return lockValue;  // Return lock value for later verification
            }
            Thread.sleep(100);
        }
    }
    
    public void processOrder(String orderId) {
        String lockKey = "order:lock:" + orderId;
        String lockValue = acquireAndReturnLockValue(orderId);
        
        try {
            // Process order (may take time)
            orderRepo.save(new Order(orderId));
        } finally {
            releaseLockSafely(lockKey, lockValue);
        }
    }
    
    private void releaseLockSafely(String lockKey, String expectedValue) {
        // Check if lock value matches before deleting
        String currentValue = redis.opsForValue().get(lockKey);
        if (expectedValue.equals(currentValue)) {
            redis.delete(lockKey);  // ✅ Safe: verified ownership
        }
        // If values don't match, another service owns the lock. Don't delete.
    }
}

// ✅ Atomic verify-and-delete would be even better (Lua script)
```

### Problem 3: Lock Extended by Another Service (Expired But Held)

```java
// ❌ WRONG: Renew lock without checking ownership
public void renewLock(String orderId) {
    String lockKey = "order:lock:" + orderId;
    redis.expire(lockKey, Duration.ofSeconds(30));  // Extend TTL
}

// Scenario:
// Service 1 acquires lock, TTL=30s
// Service 1 slow (90s needed)
// After 30s, lock expires
// Service 2 acquires lock
// Service 1 calls renewLock() → extends Service 2's lock
// ❌ Service 1 extends someone else's lock!
```

#### ✅ Solution 3: Lua Script for Atomic Operations

```java
@Service
public class OrderService {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    private static final String RENEW_SCRIPT = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then" +
        "  return redis.call('expire', KEYS[1], ARGV[2])" +
        "else" +
        "  return 0" +
        "end";
    
    public boolean renewLockIfOwned(String orderId, String lockValue, long ttlSeconds) {
        String lockKey = "order:lock:" + orderId;
        
        // Execute Lua script atomically
        Boolean result = redis.execute(
            new DefaultRedisScript<>(RENEW_SCRIPT, Boolean.class),
            Collections.singletonList(lockKey),
            lockValue, String.valueOf(ttlSeconds)
        );
        
        return result != null && result;
    }
}

// ✅ Lua ensures: check value AND extend TTL in single atomic operation
// No gap where another service could acquire lock in between
```

### Real-World: Redlock Algorithm (Redis Cluster)

```java
// For mission-critical systems with multiple Redis nodes
@Service
public class ReliableOrderService {
    private final RedissonClient redisson;  // Redlock library
    
    public void processOrder(String orderId) throws InterruptedException {
        RLock lock = redisson.getLock("order:" + orderId);
        
        if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {  // Wait max 10s, keep 30s
            try {
                // Safety guarantee: lock held on majority of Redis nodes
                // Even single node failure doesn't break lock
            } finally {
                lock.unlock();
            }
        } else {
            throw new LockAcquisitionException("Could not acquire lock");
        }
    }
}

// Redlock acquires lock on 5 Redis nodes, needs 3 confirmations.
// Guarantees: even if 2 nodes crash, lock is still held reliably.
```

### Production Patterns

#### Pattern 1: Pessimistic Lock (Acquire first, process after)

```java
// Use case: Order inventory decrement (must prevent overselling)
public boolean decrementInventory(String productId, int quantity) throws InterruptedException {
    String lockKey = "inventory:" + productId;
    String lockValue = UUID.randomUUID().toString();
    
    // 1. Acquire lock
    if (!trySetLock(lockKey, lockValue, 30)) {
        return false;  // Could not acquire
    }
    
    try {
        // 2. Read inventory
        int current = inventoryRepo.findById(productId).getQuantity();
        
        // 3. Check and write
        if (current >= quantity) {
            inventoryRepo.updateQuantity(productId, current - quantity);
            return true;
        }
        return false;  // Not enough inventory
    } finally {
        releaseLock(lockKey, lockValue);
    }
}
```

#### Pattern 2: Optimistic Lock (Retry if conflict)

```java
// Use case: Cache backfill (multiple services try, one wins)
public void backfillCache(String key, String value) {
    String lockKey = "cache:backfill:" + key;
    
    // Try to acquire lock; if fail, someone else is backfilling
    String lockValue = UUID.randomUUID().toString();
    if (redis.opsForValue().setIfAbsent(lockKey, lockValue,
            Duration.ofSeconds(10))) {
        try {
            // We own lock, backfill
            cache.set(key, value);
        } finally {
            redis.delete(lockKey);
        }
    }
    // If we couldn't acquire, another service is backfilling. Skip.
}
```

### Diagnostics

```java
// Check lock status
@GetMapping("/admin/locks")
public Map<String, Object> viewLocks() {
    Set<String> lockKeys = redis.keys("*:lock:*");
    Map<String, Object> locks = new HashMap<>();
    
    for (String key : lockKeys) {
        String value = redis.opsForValue().get(key);
        Long ttl = redis.getExpire(key, TimeUnit.SECONDS);
        locks.put(key, Map.of(
            "holder", value,
            "expiresIn", ttl + "s"
        ));
    }
    return locks;  // Visibility into lock contention
}
```

### Interview Answer

> "Distributed locks are essential in multi-instance systems. Single-instance java.lang.Object synchronization doesn't work across processes.
>
> Key principle: Always use locks with TTL (time-to-live). Without expiry, if a service crashes holding the lock, the lock persists forever. Nothing else gets it. Redis SET NX with EX is the simplest starting point.
>
> Critical gotcha: You must verify lock ownership before releasing. Store a unique value (UUID) when acquiring. Before deleting, check if the value still matches. Otherwise, you delete another service's lock prematurely.
>
> Real scenario: We had an order processing service. Service A acquired lock, processed order (60 seconds). Lock TTL was 30 seconds. We expected A to renew the lock mid-processing, but the renewal was missing. Lock expired. Service B acquired the same lock. Both A and B processed the same order concurrently. Duplicate charges, data corruption. Fix: implement lock renewal with Lua script (atomic verify-and-extend), or use Redlock for multi-node guarantees.
>
> For mission-critical systems, Redlock (acquire lock from majority of Redis nodes) prevents false lock releases from minority node failures. Trade-off: slightly higher latency but much safer.
>
> Simpler rule: pessimistic lock (acquire, then process) for resources like inventory. Optimistic (win race to backfill cache) for non-critical coordination."

**Follow-up likely:** "How would you detect if a lock is stuck/held too long?" → Monitor lock keys with TTL, alert if lock persists across multiple TTL cycles without release.

---

## Quick Revision Card — Distributed Locks

| Pattern | Mechanism | Pros | Cons |
|---------|-----------|------|------|
| **Redis SETNX with TTL** | SET key value NX EX 30 | Simple, fast | Single point of failure |
| **Database Row Lock** | SELECT ... FOR UPDATE | Atomic with DB txn | Slower, locks DB |
| **Optimistic (Version field)** | Check version before update | No lock needed | Retry loops |
| **Pessimistic (Lock later)** | Acquire, then process | Fail-fast | Potential deadlock if slow |
| **Redlock (Multi-node)** | Majority quorum on Redis cluster | Partition-tolerant | Higher latency |
| **Zookeeper** | Sequential ephemeral nodes | Fair ordering, reliable | Operational overhead |
| **Lease-based** | Renew lock periodically | Handles crashes | Needs renewal logic |

---
