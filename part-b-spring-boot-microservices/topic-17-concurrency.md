# Topic 17: Concurrency and Async Processing

## Q121. Callable vs Runnable

### Concept
Both represent tasks for execution in a thread. **Runnable** fires-and-forgets (no result, no checked exception). **Callable** can return a result and throw checked exceptions.

### Simple Explanation
**Runnable** = Fire a rocket and walk away. You can't track it.  
**Callable** = Order from a restaurant and get a **ticket** (Future). You can check back, wait for your meal, or cancel the order.

### Side-by-Side Comparison

| | Runnable | Callable |
|---|---|---|
| **Method** | `run()` | `call()` |
| **Return type** | `void` | Generic `V` |
| **Checked exceptions** | ❌ Cannot throw | ✅ Can throw |
| **Result retrieval** | Not possible | Via `Future<V>` |
| **Introduced** | Java 1.0 | Java 5 (with Executors) |
| **Usage** | Fire-and-forget tasks | Tasks requiring result or exception handling |

### Code

```java
// 1. Runnable — no result, no checked exceptions
Runnable runnable = () -> {
    System.out.println("Running on: " + Thread.currentThread().getName());
    // File I/O, DB write, sending email — fire and forget
};

ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(runnable);  // Returns Future<?>, but result is null

// 2. Callable — returns result, can throw checked exception
Callable<UserDto> callable = () -> {
    return userService.fetchUserFromRemote(123L);  // Can throw IOException
};

Future<UserDto> future = executor.submit(callable);

// Do other work while task runs...

UserDto user = future.get();          // Blocks until result ready
// future.get(5, TimeUnit.SECONDS);   // Blocks max 5 seconds
// future.cancel(true);               // Cancel task
// future.isDone();                   // Check without blocking
```

### Real Use Cases

```java
// Runnable use case: Async notification (no result needed)
@Service
public class NotificationService {
    @Autowired
    private ExecutorService executor;
    
    public void sendOrderConfirmation(Order order) {
        Runnable notification = () -> {
            emailService.send(order.getUserEmail(), "Order Confirmed", buildBody(order));
            smsService.send(order.getUserPhone(), "Your order is confirmed: " + order.getId());
        };
        executor.submit(notification);  // Don't need result
    }
}

// Callable use case: Parallel external API calls
@Service
public class ProductAggregatorService {
    @Autowired
    private ExecutorService executor;
    
    public ProductAggregateDto getProductDetails(Long productId) throws Exception {
        // Launch 3 external calls in parallel
        Callable<ProductInfo> infoTask = () -> inventoryService.getInfo(productId);
        Callable<PriceDto> priceTask = () -> pricingService.getPrice(productId);
        Callable<ReviewSummary> reviewTask = () -> reviewService.getSummary(productId);
        
        Future<ProductInfo> infoFuture = executor.submit(infoTask);
        Future<PriceDto> priceFuture = executor.submit(priceTask);
        Future<ReviewSummary> reviewFuture = executor.submit(reviewTask);
        
        // All 3 run in parallel, total time = slowest call (not sum)
        ProductInfo info = infoFuture.get(5, TimeUnit.SECONDS);
        PriceDto price = priceFuture.get(5, TimeUnit.SECONDS);
        ReviewSummary reviews = reviewFuture.get(5, TimeUnit.SECONDS);
        
        return new ProductAggregateDto(info, price, reviews);
    }
}
```

### invokeAll — Run Multiple Callables

```java
List<Callable<UserDto>> tasks = userIds.stream()
        .map(id -> (Callable<UserDto>) () -> userService.fetchUser(id))
        .collect(Collectors.toList());

// Executes all, blocks until all complete
List<Future<UserDto>> futures = executor.invokeAll(tasks, 30, TimeUnit.SECONDS);

List<UserDto> users = futures.stream()
        .map(f -> {
            try {
                return f.get();
            } catch (Exception e) {
                log.error("Task failed", e);
                return null;
            }
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
```

### Modern Alternative: CompletableFuture

```java
// CompletableFuture replaces Callable+Future in most modern code
CompletableFuture<UserDto> future = CompletableFuture.supplyAsync(() ->
        userService.fetchUser(123L),
        executor
);

// Chain operations
future
    .thenApply(user -> userMapper.toDto(user))  // Transform result
    .thenAccept(dto -> log.info("User: {}", dto)) // Consume result
    .exceptionally(ex -> {                       // Handle error
        log.error("Failed", ex);
        return null;
    });
```

### Interview Answer
> "Runnable is for fire-and-forget tasks: no return, no checked exceptions — notifications, async logging, background writes. Callable is for tasks where you need the result or need to handle checked exceptions — external API calls, DB lookups, file processing.
>
> Both submitted via ExecutorService. Callable returns `Future<T>` — call `future.get()` to block-and-retrieve, or `future.get(timeout)` to timeout. In modern Spring, CompletableFuture replaces Callable for most use cases — supports chaining, combining, and non-blocking callbacks.
>
> Follow-up: What's the difference between `future.get()` and `future.get(timeout)` — the timeout version throws TimeoutException instead of blocking forever, critical in production to prevent thread starvation."

---

## Q122. ThreadPoolExecutor Sizing Strategy & Resource Exhaustion

### Concept
`ThreadPoolExecutor` manages a pool of reusable threads for executing tasks, avoiding the overhead of creating/destroying threads. Sizing controls how many threads exist and what happens when they're exhausted.

### Simple Explanation
**Thread pool = taxi fleet.** 
- Core threads = always-on taxis (idle, waiting for fare)
- Max threads = extra taxis called in when all core taxis are busy
- Queue = waiting room at the taxi rank
- Task rejected = rank is full AND all taxis busy — passenger turned away

### ThreadPoolExecutor Parameters

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                              // corePoolSize: Always active threads
    20,                             // maximumPoolSize: Max threads when queue full
    60L,                            // keepAliveTime: Idle extra thread lifetime
    TimeUnit.SECONDS,               //
    new LinkedBlockingQueue<>(100), // workQueue: Task buffer
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy() // rejectionHandler
);
```

### Execution Flow

```
Task arrives → Is corePool full?
    No  → Create/use a core thread → execute
    Yes → Is queue full?
        No  → Add task to queue
        Yes → Is maxPool full?
            No  → Create extra thread → execute
            Yes → REJECTION → RejectionHandler
```

### Sizing Strategy

#### CPU-Bound Tasks
```java
// CPU-bound: No blocking, just computation (sorting, hashing, encryption)
// Optimal = CPU cores (no benefit from more threads — just context switching)
int cpuCores = Runtime.getRuntime().availableProcessors();

ThreadPoolExecutor cpuExecutor = new ThreadPoolExecutor(
    cpuCores,       // corePoolSize
    cpuCores,       // maximumPoolSize (no benefit going higher)
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000)
);

// Example: 8-core machine → pool of 8
// More than 8 threads: idle threads just context-switch overhead
```

#### I/O-Bound Tasks
```java
// I/O-bound: Mostly waiting (DB queries, HTTP calls, file reads)
// Threads blocked while waiting → can support more threads than CPU count
// Rule of thumb: N_threads = N_cores × (1 + wait_time / compute_time)
// DB query: 95% wait, 5% compute → multiplier = 1 + 19 = 20

int cpuCores = Runtime.getRuntime().availableProcessors();
int threadMultiplier = 20;  // Higher for longer waits

ThreadPoolExecutor ioExecutor = new ThreadPoolExecutor(
    cpuCores * threadMultiplier,    // corePoolSize: e.g., 160 for 8-core
    cpuCores * threadMultiplier * 2, // maximumPoolSize: burst capacity
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500)
);
```

### Resource Exhaustion Risks

```java
// Risk 1: Thread starvation — pool too small
// All threads blocked on slow DB, no threads for new requests
// Symptom: Request timeout, growing queue depth

// Risk 2: Memory exhaustion — unbounded queue
new LinkedBlockingQueue<>()  // ❌ Unbounded! Queue grows to millions of tasks
// Solution: Always set queue capacity
new LinkedBlockingQueue<>(1000)  // ✅ Bounded

// Risk 3: Thread explosion — pool too large
// Thousands of threads competing for CPU
// Symptom: context switching overhead, slow per-task execution

// Risk 4: OOM — ThreadLocal not cleaned
// Thread reused, old ThreadLocal value persists to next task
ThreadLocal<Map<String, Object>> requestContext = new ThreadLocal<>();
// After task: requestContext.remove(); ← MUST do this in finally block

executor.submit(() -> {
    try {
        requestContext.set(Map.of("requestId", UUID.randomUUID().toString()));
        // business logic
    } finally {
        requestContext.remove(); // mandatory cleanup on pooled threads
    }
});
```

### Rejection Policies

```java
// 1. AbortPolicy (default) — throws RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy()
// Caller must catch and handle

// 2. CallerRunsPolicy — caller thread executes the task
new ThreadPoolExecutor.CallerRunsPolicy()
// Slows down caller naturally (backpressure) — good for queues that can't drop

// 3. DiscardPolicy — silently drops task
new ThreadPoolExecutor.DiscardPolicy()
// Good for non-critical tasks (metrics, audit logs that can be lost)

// 4. DiscardOldestPolicy — drops oldest queued task, adds new
new ThreadPoolExecutor.DiscardOldestPolicy()

// 5. Custom — log, retry, or send to DLQ
executor.setRejectedExecutionHandler((task, exec) -> {
    log.error("Task rejected: {}", task.toString());
    deadLetterQueue.add(task);
    metrics.increment("task.rejected");
});
```

### Spring @Async Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method '{}' threw exception: {}", method.getName(), ex.getMessage());
        };
    }
}

@Service
public class EmailService {
    @Async
    public CompletableFuture<Void> sendEmail(String to, String subject) {
        // Runs in async thread pool
        emailClient.send(to, subject);
        return CompletableFuture.completedFuture(null);
    }
}
```

### Monitoring Pool Health

```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) asyncExecutor;

// Key metrics to monitor
log.info("Pool size: {}, Active: {}, Queue: {}, Completed: {}",
        executor.getPoolSize(),
        executor.getActiveCount(),          // Currently executing
        executor.getQueue().size(),         // Waiting tasks
        executor.getCompletedTaskCount()); // Total completed

// Alert thresholds
if (executor.getQueue().size() > executor.getQueue().remainingCapacity() * 0.8) {
    log.warn("Thread pool queue at 80% capacity!");
    // Page on-call, scale up
}
```

### Interview Answer
> "ThreadPoolExecutor has corePoolSize (always-on threads), maximumPoolSize (burst capacity), queue (buffer), and rejection handler. Sizing depends on task type: CPU-bound = number of CPU cores (no benefit going higher — context switching overhead). I/O-bound = cores × (1 + wait/compute ratio) — often 10-20× cores for DB-heavy workloads.
>
> Exhaustion risks: thread starvation (pool too small, all blocked on slow DB), memory exhaustion (unbounded queue — millions of tasks), OOM from ThreadLocal not cleaned between tasks. Always use bounded queues. Use CallerRunsPolicy for natural backpressure, or custom handler to log and DLQ.
>
> In Spring, use ThreadPoolTaskExecutor via @EnableAsync. Monitor queue depth, active count, completed count. At 80% queue capacity: alert and scale."

---

## Q123. Parallel Stream vs Sequential Stream

### Concept
**Sequential stream** processes elements one-by-one in the calling thread. **Parallel stream** splits work across multiple threads using the ForkJoinPool.

### Simple Explanation
**Sequential** = One checkout lane — customers served one after another.  
**Parallel** = Multiple checkout lanes — customers served simultaneously (but opening all lanes for 3 customers costs more than it saves).

### How Parallel Stream Works Internally

```
Source list: [1, 2, 3, 4, 5, 6, 7, 8]

ForkJoinPool (common pool, default: N-1 CPU cores)
         |
    Fork-Join Framework
    /              \
[1,2,3,4]        [5,6,7,8]
 /    \            /    \
[1,2] [3,4]    [5,6]  [7,8]
 /\    /\       /\    /\
1  2  3  4    5  6   7  8

→ Process each leaf in parallel thread
→ Merge results up the tree
→ Return combined result
```

### Code Examples

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Sequential
long sum = numbers.stream()
        .filter(n -> n % 2 == 0)
        .mapToLong(Integer::longValue)
        .sum();

// Parallel
long sumParallel = numbers.parallelStream()  // Just add .parallelStream()
        .filter(n -> n % 2 == 0)
        .mapToLong(Integer::longValue)
        .sum();

// Custom thread pool (avoid blocking common ForkJoinPool)
ForkJoinPool customPool = new ForkJoinPool(8);
long result = customPool.submit(() ->
        numbers.parallelStream()
               .mapToLong(this::expensiveComputation)
               .sum()
).get();
customPool.shutdown();
```

### When Parallel IS Faster

```java
// ✅ WINS: CPU-bound, large dataset, independent operations
List<String> hashes = largeUserList.parallelStream()
        .map(user -> hashService.bcrypt(user.getPassword()))  // CPU-intensive
        .collect(Collectors.toList());

// ✅ WINS: Bulk data processing (100K+ elements)
List<ReportRow> report = millionRecords.parallelStream()
        .filter(r -> r.getDate().isAfter(startDate))
        .map(this::transformToReportRow)
        .collect(Collectors.toList());
```

### When Parallel IS SLOWER or DANGEROUS

```java
// ❌ Small datasets — thread overhead > computation
List<Integer> small = List.of(1, 2, 3);
small.parallelStream().sum();  // Slower than sequential — ForkJoin overhead

// ❌ I/O-bound operations — threads block, compete for I/O
List<UserDto> users = ids.parallelStream()
        .map(id -> userRepo.findById(id).orElseThrow())  // ❌ DB I/O blocks threads
        .collect(Collectors.toList());
// Uses common ForkJoinPool → blocks all parallel streams in JVM!
// Use CompletableFuture with custom executor instead

// ❌ Thread-unsafe operations
List<Integer> results = new ArrayList<>();     // ❌ Not thread-safe!
numbers.parallelStream().forEach(n -> results.add(n * 2));  // Race condition!

// ✅ Thread-safe alternative
List<Integer> results = numbers.parallelStream()
        .map(n -> n * 2)
        .collect(Collectors.toList());  // Collector is thread-safe

// ❌ Order-sensitive operations
numbers.parallelStream()
        .forEach(System.out::println);  // Random order (use forEachOrdered for order)

// ❌ Stateful lambdas
int[] count = {0};
numbers.parallelStream()
        .filter(n -> ++count[0] < 5)  // ❌ Shared mutable state — race condition
        .collect(Collectors.toList());
```

### Performance Comparison

```java
// Benchmark: Process 1M elements with 10ms computation each

// Sequential: 1,000,000 × 10ms = 10,000 seconds (doesn't use other cores)

// Parallel (8 cores): 1,000,000 × 10ms / 8 ≈ 1,250 seconds

// BUT for small lists:
// Sequential: 100 × 1ms = 100ms
// Parallel: 100 × 1ms / 8 + overhead = 30ms + 50ms overhead = 80ms (barely better)
// For 10 elements: sequential often faster

// Rule: Benefit > 10K elements + CPU-bound operation
```

### Common Pitfalls

```java
// Pitfall 1: Using common ForkJoinPool for blocking calls
// Common pool used by ALL parallel streams in JVM
// Blocking one starves all others

// ✅ Fix: custom pool for I/O
ForkJoinPool customPool = new ForkJoinPool(32);  // Large pool for I/O

// Pitfall 2: Collecting into concurrent structure
Map<String, List<User>> grouped = users.parallelStream()
        .collect(Collectors.groupingBy(User::getDepartment));  // ✅ Thread-safe

// HashMap NOT thread-safe — don't collect manually:
Map<String, List<User>> map = new HashMap<>();
users.parallelStream()
        .forEach(u -> map.computeIfAbsent(u.getDept(), k -> new ArrayList<>()).add(u));  // ❌ Race condition

// Pitfall 3: Reduce with non-associative operation
// Parallel reduce requires associative + stateless combiner
int result = numbers.parallelStream()
        .reduce(0, Integer::sum);  // ✅ Sum is associative
// Subtraction is NOT associative — different results in parallel

// Pitfall 4: findFirst vs findAny
list.parallelStream().findFirst();  // Preserves order, slower in parallel
list.parallelStream().findAny();    // Returns any match, faster in parallel
```

### Interview Answer
> "Parallel stream splits work across ForkJoinPool threads (default: N-1 cores). Wins for CPU-bound operations on large datasets (100K+ elements) — bcrypt hashing, data transformation, report generation. Automatically merges results.
>
> Dangerous for: I/O-bound operations (blocks common ForkJoinPool, starves all parallel streams in JVM — use CompletableFuture with custom executor instead), thread-unsafe shared state (use stateless lambdas and thread-safe collectors), small datasets (overhead exceeds benefit).
>
> Always use stateless lambdas in parallel streams. Prefer `collect(Collectors.toList())` over manual `forEach` + ArrayList. For I/O in parallel, create a custom ForkJoinPool. In my project, I use parallel streams for batch report generation — transformed 500K records in 3s vs 20s sequential. But never for any operation involving DB calls."

---

## Q124. synchronized vs ReentrantLock

### Concept
Both protect shared state from concurrent modification. `synchronized` is simple and implicit. `ReentrantLock` is explicit with advanced features (timeouts, tryLock, fairness, multiple conditions).

### Simple Explanation
**synchronized** = Single bathroom key hanging on a hook — only one person can take it, whoever gets it goes in, returns key when done. Simple but limited.  
**ReentrantLock** = Electronic key card system — you can try to swipe (tryLock), have multiple doors (Condition), choose who goes next (fairness), and set a time limit for waiting.

### Code Comparison

```java
// 1. synchronized — implicit lock/unlock
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;  // Lock auto-acquired on method entry, released on exit
    }
    
    public synchronized int get() {
        return count;
    }
}

// 2. ReentrantLock — explicit lock/unlock
public class BetterCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();  // Explicit lock
        try {
            count++;
        } finally {
            lock.unlock();  // ALWAYS in finally — prevent leaked lock
        }
    }
    
    public int get() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### ReentrantLock Advanced Features

#### 1. tryLock (Non-Blocking)
```java
// Don't wait indefinitely — try and give up if busy
if (lock.tryLock()) {
    try {
        // Got the lock — proceed
        processOrder(order);
    } finally {
        lock.unlock();
    }
} else {
    // Could not acquire lock — do something else
    log.warn("Resource busy, skipping order: {}", order.getId());
    queue.add(order);  // Retry later
}

// With timeout
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        processOrder(order);
    } finally {
        lock.unlock();
    }
} else {
    throw new ServiceUnavailableException("Resource locked, try again later");
}
```

#### 2. Fairness (Queue-Based Lock)
```java
// Fair lock: Threads acquire in order they requested (FIFO)
ReentrantLock fairLock = new ReentrantLock(true);  // true = fair

// Unfair (default): Any waiting thread can acquire — better throughput
ReentrantLock unfairLock = new ReentrantLock(false);

// Fair = prevents thread starvation (long-waiting thread always gets turn)
// Unfair = higher throughput (allows "barging" — thread acquires before waiting ones)
```

#### 3. Interruptible Lock Wait
```java
try {
    lock.lockInterruptibly();  // Can be interrupted while waiting
    try {
        processData();
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    log.warn("Thread interrupted while waiting for lock");
}
```

#### 4. Condition Variables (Multiple Wait Queues)
```java
// Condition: Like wait()/notify() but per-condition
// Use Case: Bounded buffer (producer-consumer)

public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> buffer;
    private final int capacity;
    
    public void produce(T item) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == capacity) {
                notFull.await();  // Wait until buffer has space
            }
            buffer.add(item);
            notEmpty.signal();  // Signal consumer that item is available
        } finally {
            lock.unlock();
        }
    }
    
    public T consume() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                notEmpty.await();  // Wait until item available
            }
            T item = buffer.poll();
            notFull.signal();  // Signal producer that space is available
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### When to Use Which

| Scenario | Use |
|---|---|
| Simple critical section (increment, add to list) | `synchronized` |
| Timeout-based lock acquisition | `ReentrantLock.tryLock(timeout)` |
| Non-blocking lock attempt | `ReentrantLock.tryLock()` |
| Prevent thread starvation | `ReentrantLock(fair=true)` |
| Multiple wait conditions | `ReentrantLock.newCondition()` |
| Interruptible wait | `ReentrantLock.lockInterruptibly()` |
| Method or block should be locked | `synchronized` — simpler, compiler-enforced |

### ReadWriteLock (Optimization for Read-Heavy)
```java
// Multiple readers OR one writer at a time
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

public double getPrice(String symbol) {
    readLock.lock();     // Multiple threads can read simultaneously
    try {
        return prices.get(symbol);
    } finally {
        readLock.unlock();
    }
}

public void updatePrice(String symbol, double price) {
    writeLock.lock();    // Exclusive — blocks all readers/writers
    try {
        prices.put(symbol, price);
    } finally {
        writeLock.unlock();
    }
}
// If reads >> writes (e.g., price feed): 10× faster than synchronized
```

### Modern Alternative: java.util.concurrent

```java
// Instead of manual locking, prefer these:
AtomicInteger counter = new AtomicInteger();  // Lock-free atomic ops
counter.incrementAndGet();  // CAS-based, no lock needed

ConcurrentHashMap<K,V> map = new ConcurrentHashMap<>();  // Segmented locking

CopyOnWriteArrayList<T> list = new CopyOnWriteArrayList<>();  // Read-heavy lists

Semaphore semaphore = new Semaphore(10);  // Limit concurrent access (e.g., DB connections)
semaphore.acquire(); 
try { ... } finally { semaphore.release(); }
```

### Interview Answer
> "Both protect shared state. `synchronized` is simpler — method or block level, lock auto-released on exit, handled by JVM. `ReentrantLock` is explicit — lock(), unlock() in finally, but gives you: tryLock() to avoid indefinite wait, timeout, fairness (FIFO to prevent starvation), Condition variables (multiple wait sets, better than wait/notify), and interruptible waiting.
>
> Preference: `synchronized` for simple critical sections — less code, no risk of forgetting unlock, JIT can optimize well. `ReentrantLock` when I need tryLock() (timeout-based resource acquisition) or multiple conditions (producer-consumer bounded buffer).
>
> Modern code: prefer AtomicInteger/ConcurrentHashMap over manual locking — lock-free CAS operations. ReadWriteLock for read-heavy data structures where shared reads don't need mutual exclusion."

---

## Q125. Implementing & Reasoning About Parallel Processing Safely

### Concept
Splitting work across multiple threads to use available CPU cores, while ensuring correctness (no race conditions, no deadlocks, no data corruption).

### The Three Core Rules of Safe Parallel Processing
1. **Never share mutable state** across threads without synchronization
2. **Prefer immutable data** and pure functions
3. **Confine state** to individual threads when possible

### Pattern 1: Divide-and-Conquer with CompletableFuture

```java
@Service
public class BatchReportService {
    
    @Autowired
    private ThreadPoolTaskExecutor executor;
    
    public ReportResult generateReport(List<Long> orderIds) {
        int batchSize = 500;
        
        // Split into batches
        List<List<Long>> batches = partition(orderIds, batchSize);
        
        // Process each batch in parallel
        List<CompletableFuture<BatchResult>> futures = batches.stream()
                .map(batch -> CompletableFuture.supplyAsync(
                        () -> processBatch(batch),  // Pure function — no shared state
                        executor.getThreadPoolExecutor()
                ))
                .collect(Collectors.toList());
        
        // Wait for all and merge
        List<BatchResult> results = futures.stream()
                .map(CompletableFuture::join)  // Blocks until all complete
                .collect(Collectors.toList());
        
        return merge(results);  // Single-threaded merge — safe
    }
    
    private BatchResult processBatch(List<Long> batchIds) {
        // Each batch independent — no shared state between batches
        List<Order> orders = orderRepo.findAllById(batchIds);
        return new BatchResult(orders.stream()
                .mapToDouble(Order::getTotal)
                .sum(), orders.size());
    }
}
```

### Pattern 2: Thread Confinement

```java
// Each thread works on its own data — no sharing
public void processOrders(List<Order> allOrders) {
    int cores = Runtime.getRuntime().availableProcessors();
    int partitionSize = allOrders.size() / cores;
    
    ExecutorService executor = Executors.newFixedThreadPool(cores);
    List<Future<Integer>> futures = new ArrayList<>();
    
    // Partition list — each thread gets non-overlapping slice
    for (int i = 0; i < cores; i++) {
        final int start = i * partitionSize;
        final int end = (i == cores - 1) ? allOrders.size() : start + partitionSize;
        final List<Order> partition = allOrders.subList(start, end);  // Thread-confined
        
        futures.add(executor.submit(() -> {
            // This thread only touches its own partition — safe
            return partition.stream()
                    .filter(o -> o.getStatus() == OrderStatus.PENDING)
                    .mapToInt(Order::getQuantity)
                    .sum();
        }));
    }
    
    int total = futures.stream()
            .mapToInt(f -> { try { return f.get(); } catch (Exception e) { return 0; } })
            .sum();
}
```

### Pattern 3: Immutable Data Exchange

```java
// Pass immutable objects between threads — no locking needed
@Value  // Lombok: generates constructor + getters, no setters
public final class OrderSummary {  // final class = can't be subclassed
    private final Long orderId;
    private final Double total;
    private final Instant createdAt;
    private final List<OrderItem> items;  // Should be Collections.unmodifiableList()
}

// Records (Java 16+) — immutable by design
public record OrderSummary(Long orderId, Double total, Instant createdAt) {
    // All fields final, auto-generated equals/hashCode/toString
}

// Thread 1 creates, Thread 2 consumes — no race condition
CompletableFuture.supplyAsync(() -> new OrderSummary(1L, 99.99, Instant.now()))
                 .thenAcceptAsync(summary -> reportService.add(summary), reportExecutor);
```

### Pattern 4: Atomic Operations (Lock-Free)

```java
@Service
public class MetricsCollector {
    private final AtomicLong requestCount = new AtomicLong(0);
    private final AtomicLong totalLatency = new AtomicLong(0);
    private final ConcurrentHashMap<String, AtomicLong> errorCounts = new ConcurrentHashMap<>();
    
    // CAS-based — no lock, no contention
    public void recordRequest(String endpoint, long latencyMs, boolean isError) {
        requestCount.incrementAndGet();               // Atomic increment
        totalLatency.addAndGet(latencyMs);            // Atomic add
        
        if (isError) {
            errorCounts
                .computeIfAbsent(endpoint, k -> new AtomicLong(0))
                .incrementAndGet();
        }
    }
    
    public MetricsSnapshot snapshot() {
        return new MetricsSnapshot(requestCount.get(), totalLatency.get());
    }
}
```

### Pattern 5: Fork/Join for Recursive Tasks

```java
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;  // Minimum partition size
    private final List<Long> numbers;
    private final int start;
    private final int end;
    
    @Override
    protected Long compute() {
        int length = end - start;
        
        if (length <= THRESHOLD) {
            // Base case: sequential
            return numbers.subList(start, end).stream().mapToLong(Long::longValue).sum();
        }
        
        // Recursive case: split and conquer
        int mid = start + length / 2;
        SumTask left = new SumTask(numbers, start, mid);
        SumTask right = new SumTask(numbers, mid, end);
        
        left.fork();                    // Async: submit left to ForkJoinPool
        long rightResult = right.compute();  // Sync: compute right in current thread
        long leftResult = left.join();        // Wait for left result
        
        return leftResult + rightResult;
    }
}

// Usage
ForkJoinPool pool = ForkJoinPool.commonPool();
Long total = pool.invoke(new SumTask(largeList, 0, largeList.size()));
```

### Pattern 6: Producer-Consumer with BlockingQueue

```java
// Thread-safe handoff between producers and consumers
@Service
public class EventProcessor {
    private final BlockingQueue<Event> queue = new LinkedBlockingQueue<>(1000);
    private final ExecutorService producers = Executors.newFixedThreadPool(4);
    private final ExecutorService consumers = Executors.newFixedThreadPool(8);
    
    public void start() {
        // 8 consumer threads
        for (int i = 0; i < 8; i++) {
            consumers.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Event event = queue.take();  // Blocks if empty
                        eventService.process(event);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }
    
    public void enqueue(Event event) throws InterruptedException {
        queue.put(event);  // Blocks if queue full (backpressure)
    }
}
```

### Deadlock Prevention

```java
// Deadlock: Thread 1 holds A, waits for B
//           Thread 2 holds B, waits for A

// ❌ Deadlock-prone
void transferMoney(Account from, Account to, BigDecimal amount) {
    synchronized (from) {      // Thread 1: locks Account A
        synchronized (to) {    // Thread 2: locks Account B first (opposite order)
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// ✅ Fix: Consistent lock ordering
void transferMoney(Account from, Account to, BigDecimal amount) {
    Account first = from.getId() < to.getId() ? from : to;   // Always lock lower ID first
    Account second = from.getId() < to.getId() ? to : from;
    
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// ✅ Or: Use tryLock() with timeout to detect deadlock
boolean success = false;
while (!success) {
    if (lock1.tryLock(1, TimeUnit.SECONDS)) {
        if (lock2.tryLock(1, TimeUnit.SECONDS)) {
            try {
                // Do work
                success = true;
            } finally {
                lock2.unlock();
            }
        }
        lock1.unlock();
    }
}
```

### Thread Safety Checklist

```
Before parallelizing, ask:
1. Does this operation have shared mutable state?
   → No:  ✅ Safe to parallelize
   → Yes: Use synchronized, Atomic, or eliminate sharing

2. Is the operation idempotent?
   → Yes: ✅ Safe to retry on thread failure
   → No:  Ensure exactly-once execution

3. Are lambdas stateless?
   → No shared variables captured by reference in lambdas

4. Is the collector thread-safe?
   → Collectors.toList(), groupingBy, etc.: ✅ Safe
   → Manual forEach to non-concurrent collection: ❌ Unsafe

5. Does parallel processing actually help?
   → Dataset < 10K elements + trivial operations: probably sequential is faster
   → Profile first, parallelize after
```

### Interview Answer
> "Safe parallel processing has three rules: no shared mutable state, prefer immutable data, confine state per thread. 
>
> My patterns: Divide-and-Conquer with CompletableFuture (split list into batches, each batch independent, merge results), Thread Confinement (partition data so each thread owns non-overlapping slice), Immutable Data Exchange (Records/Value Objects between threads — no locking needed).
>
> Lock-free with AtomicInteger/ConcurrentHashMap where possible (CAS-based, no contention). BlockingQueue for producer-consumer (natural backpressure when queue full). Deadlock prevention: consistent lock ordering or tryLock with timeout.
>
> Before parallelizing: check if lambdas are stateless, check dataset size (< 10K elements often slower in parallel). Profile before and after — I used parallel CompletableFuture processing to reduce batch report generation from 45s to 8s on a 200K record dataset."

---

## Quick Revision Card — Section 13

| Topic | Key Point |
|---|---|
| **Runnable** | `run()`, void, no checked exception, fire-and-forget |
| **Callable** | `call()`, returns `T`, can throw, retrieval via `Future.get()` |
| **CompletableFuture** | Modern replacement — chainable, combinable, non-blocking callbacks |
| **ThreadPoolExecutor** | coreSize, maxSize, queue, rejectionHandler. CPU-bound = cores, I/O-bound = cores × (1 + wait/compute) |
| **Rejection Policies** | Abort (throw), CallerRuns (backpressure), Discard (drop), DiscardOldest |
| **Queue** | Always bounded. Unbounded = OOM risk. 80% full = alert |
| **Parallel Stream** | CPU-bound + large data (10K+) = wins. I/O-bound = blocks ForkJoinPool. Stateless lambdas required. |
| **Sequential Stream** | Small datasets, I/O operations, order-sensitive |
| **synchronized** | Simple, implicit, method/block level. JIT optimized. |
| **ReentrantLock** | tryLock (timeout), fairness, Condition (multiple wait sets), interruptible |
| **ReadWriteLock** | Multiple concurrent readers OR one writer. Read-heavy optimization. |
| **AtomicInteger** | Lock-free CAS for counters/flags. No contention. |
| **Safe Parallelism** | No shared mutable state, immutable data, thread confinement, stateless lambdas |
| **Deadlock Prevention** | Consistent lock ordering, tryLock with timeout |
| **BlockingQueue** | Thread-safe producer-consumer, `put()` = backpressure, `take()` = blocks until item |

---

**End of Section 13**

---

## Additional Deep-Dive (Q121-Q125)

### Senior-Level Concurrency Guardrails

- Prefer immutability and message passing over lock-heavy shared mutable state.
- Use bounded queues to enforce backpressure; unbounded queues hide overload until memory collapse.
- Measure contention (`blocked/waiting` thread states) before introducing complex lock strategies.

### Real Project Usage

- In async batch pipelines, bounded thread pools plus queue limits prevented downstream DB collapse during traffic bursts.
- Replacing naive parallel streams with explicit `ExecutorService` controls improved predictability under load.

### Java 21 Virtual Threads Note

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> result = executor.submit(() -> "ok");
    System.out.println(result.get());
}
```

Interview point: virtual threads are strong for I/O-bound workloads with high concurrency; CPU-bound work still needs controlled parallelism by core count.

---

## Q125. wait(), notify(), and notifyAll() — Coordinating Threads with Manual Synchronization

### Concept
`wait()`, `notify()`, and `notifyAll()` are low-level mechanisms for threads to communicate: one thread pauses waiting for a condition, while another thread modifies state and wakes the waiter. This is the foundation for thread-safe producer-consumer queues, condition variables, and manual synchronization. Modern code prefers `Lock` + `Condition` or queue-based messaging, but understanding `wait/notify` is essential for interviews and debugging legacy code.

### Simple Explanation
Imagine a restaurant kitchen:
- **Cook** (producer): Prepares a dish. Notifies the waiter: "Order ready!"
- **Waiter** (consumer): Waits at the counter. When the cook notifies, the waiter picks up the dish and delivers it.

`wait()` = Waiter sleeps at counter, giving up the counter lock.  
`notify()` = Cook wakes one waiter (arbitrarily, if multiple are waiting).  
`notifyAll()` = Cook wakes ALL waiters. Safe default; let them compete for the lock.

Critical: You must hold the lock when calling `wait()`, `notify()`, or `notifyAll()`. They work *inside* a synchronized block.

### How It Works — The State Machine

```
┌──────────────┐
│   Running    │  Thread holds lock, executing code
│              │
└──────┬───────┘
       │ wait()
       ▼
┌──────────────────────┐
│  Waiting/Blocked     │  Lock released, thread suspended
│  (on condition var)  │  Woken by notify() or timeout
└┬───────────────────┬─┘
 │                   │
 │ notify()          │ interrupt/timeout
 │ another thread    │
 │ releases lock     │
 ▼                   ▼
┌──────────────┐    (exception)
│   Running    │
└──────────────┘
```

### Code — Producer-Consumer Pattern

#### ❌ Problem: Spin-wait (Wasteful CPU, prone to missed signals)

```java
public class BoundedQueue<T> {
    private final List<T> items = new ArrayList<>();
    private final int capacity;
    
    public BoundedQueue(int capacity) { this.capacity = capacity; }
    
    // ❌ WRONG: Consumer spin-waits (wastes CPU)
    public T take() throws InterruptedException {
        while (items.isEmpty()) {
            Thread.sleep(100);  // Spin, polling
        }
        return items.remove(0);
    }
    
    public void put(T item) throws InterruptedException {
        while (items.size() >= capacity) {
            Thread.sleep(100);  // Producer also spins
        }
        items.add(item);
    }
}

// Problems:
// 1. CPU waste: threads sleep 100ms repeatedly even if queue is empty for hours
// 2. Latency: consumer doesn't wake immediately; waits up to 100ms after item added
// 3. Thundering herd: if many consumers, they all wake and compete
```

#### ✅ Solution: wait/notify Pattern

```java
public class BoundedQueue<T> {
    private final List<T> items = new ArrayList<>();
    private final int capacity;
    private final Object lock = new Object();
    
    public BoundedQueue(int capacity) { this.capacity = capacity; }
    
    public T take() throws InterruptedException {
        synchronized (lock) {
            while (items.isEmpty()) {  // while, not if (spurious wakeup protection)
                lock.wait();  // Release lock, sleep until notified
            }
            T item = items.remove(0);
            lock.notifyAll();  // Wake producers blocked on put()
            return item;
        }
    }
    
    public void put(T item) throws InterruptedException {
        synchronized (lock) {
            while (items.size() >= capacity) {
                lock.wait();  // Release lock, sleep until notified
            }
            items.add(item);
            lock.notifyAll();  // Wake consumers blocked on take()
        }
    }
}

// Behavior:
// 1. Consumer calls take() -> queue empty -> wait() suspends, releases lock
// 2. Producer calls put() -> adds item -> notifyAll() wakes all waiters
// 3. Consumer wakes, re-acquires lock, returns item
// 4. Zero CPU waste, immediate wakeup
```

#### ⚠️ Spurious Wakeups — The Gotcha

```java
// ❌ WRONG: Use if instead of while
public T take() throws InterruptedException {
    synchronized (lock) {
        if (items.isEmpty()) {  // ❌ Dangerous
            lock.wait();  // Woken by notify()
        }
        return items.remove(0);  // "Assume" items is not empty!
    }
}

// Java spec: wait() can wake spuriously, even without notify() call.
// If queue is still empty after spurious wakeup, items.remove(0) throws IndexOutOfBoundsException.

// ✅ CORRECT: Use while
public T take() throws InterruptedException {
    synchronized (lock) {
        while (items.isEmpty()) {  // Loop, re-check condition
            lock.wait();  // Even if spurious wakeup, loop re-checks
        }
        return items.remove(0);  // Now guaranteed items is not empty
    }
}
```

### wait() Timeout — Preventing Deadlock

```java
public T take(long timeoutMs) throws InterruptedException, TimeoutException {
    synchronized (lock) {
        long deadline = System.currentTimeMillis() + timeoutMs;
        while (items.isEmpty()) {
            long remaining = deadline - System.currentTimeMillis();
            if (remaining <= 0) {
                throw new TimeoutException("Queue empty for " + timeoutMs + "ms");
            }
            lock.wait(remaining);  // Wait max 'remaining' milliseconds
        }
        T item = items.remove(0);
        lock.notifyAll();
        return item;
    }
}

// Timeout prevents indefinite blocking if producer fails.
```

### notify() vs notifyAll() — When to Use Which

```java
// notify() wakes one arbitrary waiting thread
public void put(T item) throws InterruptedException {
    synchronized (lock) {
        while (items.size() >= capacity) {
            lock.wait();
        }
        items.add(item);
        lock.notify();  // ❌ Only wakes one consumer (or one producer)
    }
}

// Problem: If multiple consumers waiting and you notify(),
// only ONE wakes. Others stay asleep. Latency unpredictable.
// Or if multiple producers and consumer wakes a producer (wrong waiter type), inefficient.

// ✅ notifyAll() wakes everyone, let scheduler decide
public void put(T item) throws InterruptedException {
    synchronized (lock) {
        while (items.size() >= capacity) {
            lock.wait();
        }
        items.add(item);
        lock.notifyAll();  // ✅ Wakes all waiter threads
    }
}

// All consumers and producers wake, re-test condition (while loop),
// only threads whose condition is satisfied proceed.
```

### Real Project Usage — Throttling Task Queue

```java
public class ThrottledTaskQueue implements Runnable {
    private final Queue<Runnable> tasks = new LinkedList<>();
    private final int maxConcurrent = 10;
    private int activeCount = 0;
    private final Object lock = new Object();
    private volatile boolean shutdown = false;
    
    public void submit(Runnable task) throws InterruptedException {
        synchronized (lock) {
            while (activeCount >= maxConcurrent) {
                lock.wait();  // Throttle: don't pile up too many tasks
            }
            tasks.offer(task);
        }
    }
    
    @Override
    public void run() {
        while (!shutdown) {
            Runnable task = null;
            synchronized (lock) {
                if (!tasks.isEmpty()) {
                    task = tasks.poll();
                    activeCount++;
                }
            }
            
            if (task == null) {
                Thread.sleep(10);  // Small idle loop
            } else {
                try {
                    task.run();
                } finally {
                    synchronized (lock) {
                        activeCount--;
                        lock.notifyAll();  // Wake submit() threads if activeCount dropped below limit
                    }
                }
            }
        }
    }
}

// Use case: Database connection pool handler.
// Only 10 connections active at once. submit() blocks if limit reached, unblocks when a task finishes.
```

### Production Pitfalls

```java
// ❌ Pitfall 1: Forgetting to check condition after wait()
while (items.isEmpty()) {  // Correct
    lock.wait();
}
// If you use if, spurious wakeup or wrong notification type leads to bugs.

// ❌ Pitfall 2: Using notify() in multi-consumer scenario
// Only one thread wakes. If it's a producer (wrong type), it waits again.
// Dead: all other consumers stay asleep.

// ✅ Always use notifyAll() unless you're absolutely sure only one thread is waiting.

// ❌ Pitfall 3: Holding the lock too long
synchronized (lock) {
    // SLOW I/O here while holding lock
    http.get("https://api.example.com");  // Blocks all other threads!
    lock.notifyAll();
}

// ✅ Release lock before slow operations
Runnable task;
synchronized (lock) {
    task = tasks.poll();
    // ... minimal work, release lock
}
// SLOW OPERATION HERE (not holding lock)
if (task != null) task.run();
```

### Modern Alternative: Lock + Condition

```java
// Java 5+: Explicit lock + condition (preferred)
public class BoundedQueueModern<T> {
    private final List<T> items = new ArrayList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (items.isEmpty()) {
                notEmpty.await();  // Wait for not-empty condition
            }
            T item = items.remove(0);
            notFull.signal();  // Signal not-full condition
            return item;
        } finally {
            lock.unlock();
        }
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (items.size() >= capacity) {
                notFull.await();
            }
            items.add(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
}

// Advantages:
// 1. Named conditions (notEmpty, notFull) explicit
// 2. signal() vs signalAll() clearer intent
// 3. Can use interruptibly (with timeout)
// 4. Better performance in high-contention scenarios
```

### Interview Answer

> "wait() and notify() are the foundation of thread-safe queues and condition variables. They're synchronization primitives.
>
> Here's the pattern: You hold a lock inside synchronized block. Check if your condition is not met (while loop, not if). If not met, call wait() — this releases the lock and suspends the thread. Another thread modifies state and calls notifyAll(). This wakes all waiting threads. They re-acquire the lock and re-check the condition (while loop handles spurious wakeups).
>
> Critical point: Always use while, not if, when checking condition after wait(). Java spec allows spurious wakeups — the thread can wake without anyone calling notify(). If you use if, you assume the condition is true after wakeup, but it might not be.
>
> notify() wakes one arbitrary thread. notifyAll() wakes all. In production, I always use notifyAll() unless I *know* only one type of thread is waiting. notify() leads to subtle bugs where you wake the wrong thread type and deadlock.
>
> Real example: We had a task queue with producers and consumers. Producers called notify() after adding a task. Sometimes a producer woke another producer (wrong!), that producer waited again, and all consumers stayed asleep. Task backed up. We switched to notifyAll(), problem solved.
>
> Modern Java prefers Lock + Condition API (explicit conditions, named signals) over wait/notify, especially in complex scenarios. But every Java developer must understand wait/notify for interviews, reading legacy code, and diagnosing thread coordination bugs."

**Follow-up likely:** "What's a spurious wakeup and why does it happen?" → Thread can wake without notify(). OS signal, hardware interrupt, or JVM implementation detail. That's why while loops are essential.

---

## Quick Revision Card — wait/notify

| Concept | Usage | Gotcha |
|---------|-------|--------|
| **wait()** | Suspend thread; release lock | Must hold synchronized lock first |
| **while (condition)** | Re-check after wake | if causes spurious wakeup vulnerability |
| **notify()** | Wake one waiting thread | Random choice; wrong thread might wake |
| **notifyAll()** | Wake all waiting threads | Safe default; let scheduler decide |
| **Timeout** | wait(ms) | Prevents indefinite deadlock |
| **Lock + Condition** | Modern alternative (Java 5+) | signal() vs signal(All), explicit condition names |
| **Producer-Consumer** | Coordinated put/take with wait/notify | Prototype for all thread coordination |
| **Throttling** | Limit concurrent tasks | Uses wait() to pause submissions when over-capacity |

---
