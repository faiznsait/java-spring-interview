# Topic 5 — Multithreading, Concurrency & Async

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer → Gotchas & Related Concepts

---

## Q1. Process vs Thread — OS-Level Concepts

### Concept
A **process** is an isolated program instance with its own memory space. A **thread** is a lightweight execution unit within a process that shares memory with other threads in the same process.

### Simple Explanation
A process is like owning a separate house with its own kitchen, bedroom, storage. A thread is like multiple people living in the SAME house sharing the kitchen, bedroom, storage. If one person breaks something, everyone's affected. That's why threads need synchronization.

### Key Differences

| Aspect | Process | Thread |
|--------|---------|--------|
| **Memory** | Isolated, separate heap | Shared heap within process |
| **Creation** | Heavy (OS resource allocation) | Light (just stack allocation) |
| **IPC** (communication) | Complex (pipes, sockets, shared memory) | Simple (shared variables) |
| **Context switch** | Expensive (memory + registers) | Cheaper (registers only) |
| **Failure isolation** | Process crash doesn't affect others | Thread crash kills entire JVM |
| **Example** | Chrome tab, Firefox window | Worker threads in single app |

### Code — Process vs Thread

```java
// PROCESS: OS launches separate JVM instances
ProcessBuilder pb = new ProcessBuilder("java", "-jar", "worker.jar");
Process process = pb.start();  // Creates entirely new JVM
// Child process has SEPARATE heap, CLASS LOADER, EVERYTHING

// THREAD: Same JVM, shared memory
new Thread(() -> {
    System.out.println("Worker thread");
    // Shares class objects, static fields, heap with main thread
}).start();

// Gotcha: Static field modification visible across threads!
public class Counter {
    private static int count = 0;  // Shared by ALL threads
}

Thread t1 = new Thread(() -> Counter.count = 10);
Thread t2 = new Thread(() -> System.out.println(Counter.count));  // Might see 10!

// In separate processes:
Process p1 = Runtime.getRuntime().exec("java ProcessA");
Process p2 = Runtime.getRuntime().exec("java ProcessB");
// Each has OWN count variable — no shared state
```

---

## Q2. Thread Lifecycle — States of a Thread

### Concept
A thread transitions through 6 states from creation to death. Understanding these states is critical for debugging concurrency bugs.

### The 6 Thread States (with State Transitions)

```
1. NEW
   ↓ (thread.start() called)
2. RUNNABLE
   ↓ (JVM scheduler, waiting for CPU)
3. BLOCKED (on synchronized lock)
   ↓ (acquired lock)
   ↑ (released lock)
4. WAITING (indefinite wait for signal)
   ↓ (notified)
   ↑ (wait() or join())
5. TIMED_WAITING (wait with timeout)
   ↓ (timeout expires or notified)
   ↑ (Thread.sleep(), wait(timeout), join(timeout))
6. TERMINATED
```

### Code — Observing Thread States

```java
Thread worker = new Thread(() -> {
    try {
        System.out.println("In run, state: " + Thread.currentThread().getState());
        Thread.sleep(100);  // TIMED_WAITING
    } catch (InterruptedException e) {}
});

System.out.println("Created, state: " + worker.getState());  // NEW

worker.start();
System.out.println("Started, state: " + worker.getState());   // RUNNABLE (or might show start time variance)

try { Thread.sleep(10); } catch (Exception e) {}
System.out.println("During sleep, state: " + worker.getState());  // TIMED_WAITING

worker.join();  // WAITING for worker to finish
System.out.println("After join, state: " + worker.getState());  // TERMINATED

// Output:
// Created, state: NEW
// Started, state: RUNNABLE
// During sleep, state: TIMED_WAITING
// In run, state: RUNNABLE
// After join, state: TERMINATED

// BLOCKED state example:
Object lock = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock) {
        try { Thread.sleep(100); } catch (InterruptedException e) {}
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock) {  // ← Will block waiting for t1 to release
        System.out.println("t2 acquired lock");
    }
});

t1.start();
try { Thread.sleep(10); } catch (InterruptedException e) {}
t2.start();
try { Thread.sleep(5); } catch (InterruptedException e) {}

System.out.println("t2 state: " + t2.getState());  // BLOCKED (waiting for lock)
```

---

## Q3. wait() vs sleep() — Critical Difference

### Concept
- **sleep()**: Static method, pauses execution for fixed time, **keeps lock**
- **wait()**: Instance method, releases lock, waits for notification (indefinite)

### Simple Explanation
- **sleep()**: Like taking a nap while holding your car keys — no one else can start the car
- **wait()**: Like handing your car keys to a mechanic and sleeping — mechanic can use the car, and can wake you when done

### Code Comparison

```java
// ❌ WRONG: Using sleep() to wait for a condition
Object lock = new Object();

Thread task = new Thread(() -> {
    synchronized (lock) {
        boolean ready = false;
        while (!ready) {
            Thread.sleep(100);  // ← WRONG! Holding lock, wasting CPU
            ready = checkCondition();  // Re-check
        }
    }
});
// Problem: lock is held while sleeping, other threads can't enter synchronized block!

// ✅ CORRECT: Using wait() to wait for a condition
Thread task = new Thread(() -> {
    synchronized (lock) {
        while (!ready) {
            lock.wait();  // ← Releases lock, waits efficiently
        }
    }
});

// Meanwhile, other thread signals:
synchronized (lock) {
    ready = true;
    lock.notifyAll();  // Wakes 'task' thread
}
```

### Detailed Comparison Table

| Aspect | `sleep()` | `wait()` |
|--------|-----------|---------|
| **Type** | Static (Thread.sleep) | Instance (object.wait) |
| **Lock status** | Keeps lock | **Releases lock** ✓ |
| **Notification** | None (waits timeout) | Woken by notify() |
| **Exception** | InterruptedException | InterruptedException |
| **Wake up** | After time elapsed | Immediately on notify() |
| **Use case** | Delay before retry | Wait for condition |

### Real Production Pattern

```java
// ❌ WRONG: Busy loop with sleep
public class MessageQueue {
    private Queue<String> messages = new LinkedList<>();
    
    public String take() throws InterruptedException {
        while (messages.isEmpty()) {
            Thread.sleep(100);  // ← Wasteful spinning!
        }
        return messages.poll();
    }
}

// ✅ CORRECT: BlockingQueue (uses wait internally)
public class MessageQueue {
    private BlockingQueue<String> messages = new LinkedBlockingQueue<>();
    
    public String take() throws InterruptedException {
        return messages.take();  // Efficient waiting
    }
}
```

---

## Q4. wait(), notify(), notifyAll() — Full Deep Dive

### Concept
These methods coordinate between threads. `wait()` puts a thread to sleep, `notify()` wakes one, `notifyAll()` wakes all waiting on that object's monitor.

### Critical Rules

```java
// Rule 1: wait() MUST be called inside synchronized block
synchronized (lock) {
    lock.wait();  // ✓ Legal
}

lock.wait();  // ❌ IllegalMonitorStateException!

// Rule 2: Always check condition in while loop (spurious wakeup!)
synchronized (lock) {
    while (!condition) {  // while, NOT if!
        lock.wait();
    }
}

// Rule 3: Notify AFTER changing state (producer-consumer pattern)
synchronized (lock) {
    data = newValue;
    lock.notifyAll();  // Notify AFTER state change
}

// Rule 4: notifyAll() > notify() in most cases
synchronized (lock) {
    lock.notifyAll();  // Safe: wakes all, all re-check condition
}

synchronized (lock) {
    lock.notify();  // ⚠️ Risky: wakes only one, others might miss notification
}
```

### Producer-Consumer Pattern

```java
public class BoundedBuffer<T> {
    private final Queue<T> buffer = new LinkedList<>();
    private final int capacity;
    private final Object lock = new Object();
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    // Producer: adds to buffer
    public void put(T item) throws InterruptedException {
        synchronized (lock) {
            while (buffer.size() == capacity) {  // ← while, not if!
                lock.wait();  // Wait until consumer removes (buffer not full)
                // Woken by notifyAll() from take()
            }
            buffer.add(item);
            lock.notifyAll();  // Notify consumers (buffer not empty)
        }
    }
    
    // Consumer: takes from buffer
    public T take() throws InterruptedException {
        synchronized (lock) {
            while (buffer.isEmpty()) {  // ← while, not if!
                lock.wait();  // Wait until producer adds (buffer not empty)
                // Woken by notifyAll() from put()
            }
            T item = buffer.poll();
            lock.notifyAll();  // Notify producers (buffer not full)
            return item;
        }
    }
}

// Test:
BoundedBuffer<Integer> buffer = new BoundedBuffer<>(2);

Thread producer = new Thread(() -> {
    try {
        for (int i = 0; i < 5; i++) {
            buffer.put(i);
            System.out.println("Produced: " + i);
        }
    } catch (InterruptedException e) {}
});

Thread consumer = new Thread(() -> {
    try {
        for (int i = 0; i < 5; i++) {
            int val = buffer.take();
            System.out.println("Consumed: " + val);
            Thread.sleep(100);  // Slow consumer
        }
    } catch (InterruptedException e) {}
});

producer.start();
consumer.start();

// Output shows synchronization: producer waits when buffer full, consumer wakes it
```

### Spurious Wakeup — Why while Loop is Essential

```java
// Spurious wakeup: thread wakes even without explicit notification (rare but happens)
// JVM spec allows this, so we must guard against it

// ❌ DANGEROUS: Using if
synchronized (lock) {
    if (!condition) {
        lock.wait();
    }
    // ← Woken (might be spurious), proceeds WITHOUT checking condition again!
    useData();  // ← Might proceed when condition is still false!
}

// ✅ CORRECT: Using while
synchronized (lock) {
    while (!condition) {  // Re-checks condition on every wakeup
        lock.wait();
    }
    // Only proceeds if condition is actually true
    useData();
}
```

---

## Q5. CountDownLatch — Synchronization Barrier

### Concept
A **CountDownLatch** allows one thread to wait until a specific count of operations in other threads reaches zero. One-time use only.

### Simple Explanation
Like a race: Runners line up at the starting line (count=3, one per runner). Race official counts down: "3... 2... 1..." (threads call `countDown()`). When count reaches 0, the "GO!" signal triggers (other threads waiting on `await()` are released).

### Code — All Threads Wait for Setup

```java
// Pattern: Main thread waits for N worker threads to complete setup
int numWorkers = 3;
CountDownLatch startSignal = new CountDownLatch(numWorkers);

ExecutorService executor = Executors.newFixedThreadPool(numWorkers);

// Submit workers
for (int i = 0; i < numWorkers; i++) {
    final int workerId = i;
    executor.submit(() -> {
        try {
            System.out.println("Worker " + workerId + " starting setup...");
            Thread.sleep(100 * (workerId + 1));  // Simulate setup time
            System.out.println("Worker " + workerId + " setup complete!");
            startSignal.countDown();  // Signal that setup is done
        } catch (InterruptedException e) {}
    });
}

// Main thread waits for all workers to finish setup
System.out.println("Main: Waiting for workers to finish setup...");
startSignal.await();  // Blocks until count reaches 0
System.out.println("Main: All workers ready! Starting actual work.");

executor.shutdown();

// Output:
// Worker 0 starting setup...
// Worker 1 starting setup...
// Worker 2 starting setup...
// Main: Waiting for workers to finish setup...
// Worker 0 setup complete!
// Worker 1 setup complete!
// Worker 2 setup complete!
// Main: All workers ready! Starting actual work.
```

### CountDownLatch vs CyclicBarrier

```java
// CountDownLatch: One-time, one purpose
// ✓ M threads signal (countDown), 1 thread waits (await)
// ✓ One-way: when count reaches 0, no reset possible
// ✓ Use: Task completion synchronization

CountDownLatch latch = new CountDownLatch(3);
// 3 tasks complete → latch.countDown() three times → main thread woken from await()

// CyclicBarrier: Reusable, symmetric
// ✓ N threads ALL wait at barrier until all N arrive
// ✓ Upon successful release, resets automatically for next round
// ⚠️ If any waiter is interrupted/times out, barrier becomes BROKEN (manual reset needed)
// ✓ Use: Round-based synchronization (phases)
```

---

## Q6. CyclicBarrier — Reusable Synchronization Barrier

### Concept
A **CyclicBarrier** forces a fixed number of threads to wait for each other at a barrier point before proceeding. Reusable across multiple rounds.
If one waiting thread is interrupted or times out, the barrier enters a broken state and remaining waiters get `BrokenBarrierException`.

### Simple Explanation
Like a group hike: Each hiker must reach the checkpoint before anyone stops to rest. Once ALL hikers reach it, group rests together. Then they all continue to the NEXT checkpoint (reusable).

### Code — Multi-Phase Synchronization

```java
// Pattern: N threads must sync at each phase
int numThreads = 3;
CyclicBarrier barrier = new CyclicBarrier(
    numThreads,
    () -> System.out.println("  → All threads at barrier, proceeding to next phase...")
);

ExecutorService executor = Executors.newFixedThreadPool(numThreads);

for (int i = 0; i < numThreads; i++) {
    final int threadId = i;
    executor.submit(() -> {
        try {
            for (int phase = 1; phase <= 3; phase++) {
                System.out.println("Thread " + threadId + " completed phase " + phase);
                barrier.await();  // Wait for all threads to complete this phase
                // ↓ All threads released together
            }
            System.out.println("Thread " + threadId + " finished all phases!");
        } catch (InterruptedException | BrokenBarrierException e) {
            // In real code: log, stop this worker, and optionally barrier.reset() to recover
        }
    });
}

executor.shutdown();
executor.awaitTermination(10, TimeUnit.SECONDS);

// Output:
// Thread 0 completed phase 1
// Thread 1 completed phase 1
// Thread 2 completed phase 1
//   → All threads at barrier, proceeding to next phase...
// Thread 0 completed phase 2
// Thread 1 completed phase 2
// Thread 2 completed phase 2
//   → All threads at barrier, proceeding to next phase...
// (phase 3 similar)
//   → All threads at barrier, proceeding to next phase...
// Thread 0 finished all phases!
// Thread 1 finished all phases!
// Thread 2 finished all phases!
```

---

## Q7. Semaphore — Permit-Based Access Control

### Concept
A **Semaphore** maintains a set of **permits**. Threads call `acquire()` to get a permit (block if none available), and `release()` to return it. Useful for limiting concurrent access to a resource.

### Simple Explanation
Like a parking lot with 5 spaces: Each car needs a permit to park. When a car leaves, the space (permit) becomes available for another car. More than 5 cars must wait.

### Code — Resource Pool Limiting

```java
// Pattern: Limit concurrent access to a resource pool
Semaphore semaphore = new Semaphore(3);  // Max 3 concurrent accesses
ExecutorService executor = Executors.newFixedThreadPool(6);

for (int i = 0; i < 6; i++) {
    final int taskId = i;
    executor.submit(() -> {
        try {
            System.out.println("Task " + taskId + " waiting for permit...");
            semaphore.acquire();  // Block if no permits available
            System.out.println("Task " + taskId + " acquired permit, working...");
            Thread.sleep(500);  // Simulate work
            System.out.println("Task " + taskId + " finished");
            semaphore.release();  // Release permit for others
        } catch (InterruptedException e) {}
    });
}

executor.shutdown();
executor.awaitTermination(10, TimeUnit.SECONDS);

// Output:
// Task 0 waiting for permit...
// Task 0 acquired permit, working...
// Task 1 waiting for permit...
// Task 1 acquired permit, working...
// Task 2 waiting for permit...
// Task 2 acquired permit, working...
// Task 3 waiting for permit...
// Task 4 waiting for permit...
// Task 5 waiting for permit...
// Task 0 finished
// Task 3 acquired permit, working...
// (Tasks 4, 5 get permits as others release)
```

### Binary Semaphore (Mutex)

```java
// Semaphore with 1 permit acts like a Mutex (mutual exclusion lock)
Semaphore mutex = new Semaphore(1);  // Only 1 thread at a time

synchronized (mutex) {  // NOT synchronized! Using semaphore instead
    // Critical section: at most 1 thread here
}

// More flexible: can allow N concurrent threads instead of 1
Semaphore pool = new Semaphore(5);  // Max 5 concurrent
pool.acquire();
try {
    // Safe: up to 5 threads here simultaneously
} finally {
    pool.release();
}
```

---

## Q8. Java Memory Model (JMM) — Happens-Before Guarantees

### Concept
The **Java Memory Model** defines visibility and ordering rules across threads. **Happens-before** is a partial ordering that guarantees when one operation's effects are visible to another thread.

### Simple Explanation
Imagine two friends: Alice writes a note and puts it in a shared mailbox. The mailbox guarantees that Bob who opens it AFTER Alice closes it will see the note (visibility). The order is: Alice writes → Alice closes → Bob opens → Bob reads (happens-before order).

### Core Happens-Before Rules

```java
// Rule 1: Program Order — within same thread, statements execute in order
int x = 1;
int y = 2;
int z = x + y;  // Sees x=1, y=2 in same thread (guaranteed)

// Rule 2: Synchronization — lock acquisition/release
synchronized (lock) {
    x = 1;  // Write inside lock
}
// All writes before unlock happen-before all reads after lock acquisition

synchronized (lock) {
    int value = x;  // Sees x=1 (guaranteed visibility)
}

// Rule 3: volatile — visibility across threads (most important!)
volatile boolean stop = false;

Thread writer = new Thread(() -> {
    stop = true;  // Write to volatile
});

Thread reader = new Thread(() -> {
    while (!stop) {  // Read sees true (guaranteed visibility)
        doWork();
    }
});

// IMPORTANT: volatile gives visibility/ordering, NOT atomicity
volatile int counter = 0;
counter++;  // still a race (read-modify-write), not thread-safe
// For increments/counters across threads use AtomicInteger.incrementAndGet()

// Rule 4: Thread.start() → everything before start() happens-before thread's code
x = 1;
new Thread(() -> {
    int value = x;  // Guaranteed to see x=1
    doWork();
}).start();

// Rule 5: Thread.join() → thread's last statement happens-before code after join()
new Thread(() -> {
    x = 1;  // Last statement
}).start().join();
int value = x;  // Guaranteed to see x=1

// Rule 6: Happens-before is transitive: if A→B and B→C, then A→C
```

### Why JMM Matters

```java
// ❌ BROKEN: Without understanding JMM
class BrokenSharedState {
    int x = 0;
    
    void writer() {
        x = 1;  // Write to non-volatile field
    }
    
    void reader() {
        while (x == 0) {  // Might NEVER see x=1!
            // JIT optimizer can cache x in register
            // No happens-before guarantee
        }
    }
}

// ✅ FIXED: With volatile
class FixedSharedState {
    volatile int x = 0;  // Changes to volatile add memory barriers
    
    void writer() {
        x = 1;  // GUARANTEED visible to reader thread
    }
    
    void reader() {
        while (x == 0) {  // GUARANTEED to see x=1 eventually
            // Memory barrier prevents register caching
        }
    }
}
```

---

## Q9. Atomic Classes — AtomicInteger, CAS, and Lock-Free Programming

### Concept
**Atomic classes** provide lock-free, thread-safe operations for single variables using **Compare-And-Swap (CAS)** hardware instruction. Faster than synchronized/locks.

### Simple Explanation
Like a vending machine: Insert coin (read current state), check if it's valid (compare), dispense item (swap to new state) — all in one atomic action. No one else can interrupt.

### Code — AtomicInteger vs synchronized

```java
// ❌ Synchronized (slower, uses OS lock)
public class CounterSync {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int get() {
        return count;
    }
}

// ✅ AtomicInteger (faster, lock-free with CAS)
public class CounterAtomic {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();  // Lock-free
    }
    
    public int get() {
        return count.get();
    }
}

// Performance: 10M increments in single-threaded
// Sync: ~500ms
// Atomic: ~100ms (5× faster!)
// CAS: Faster even with contention (multi-threaded)
```

### CAS (Compare-And-Swap) Mechanics

```java
// CAS pseudocode:
// if (memory[address] == expectedValue) {
//     memory[address] = newValue;
//     return true;
// }
// return false;
// All atomic hardware operation!

// AtomicInteger uses CAS internally:
AtomicInteger counter = new AtomicInteger(0);

// incrementAndGet() with CAS:
// do {
//     int current = counter.get();
//     int next = current + 1;
// } while (!counter.compareAndSet(current, next));

// compareAndSet (CAS operation):
AtomicInteger value = new AtomicInteger(10);
boolean success = value.compareAndSet(10, 20);  // true — swapped
System.out.println(value.get());  // 20

success = value.compareAndSet(10, 30);  // false — 10 != 20, no swap
System.out.println(value.get());  // still 20

// getAndSet (atomic exchange):
AtomicInteger value = new AtomicInteger(5);
int old = value.getAndSet(10);  // Returns 5, sets to 10
System.out.println(old);  // 5
System.out.println(value.get());  // 10
```

### Common Atomic Classes

```java
// AtomicInteger, AtomicLong, AtomicBoolean, AtomicReference
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();      // ++count
count.getAndIncrement();      // count++
count.decrementAndGet();      // --count
count.addAndGet(5);           // count += 5
count.getAndSet(10);          // return count; count = 10

// AtomicReference (for objects)
AtomicReference<String> ref = new AtomicReference<>("initial");
String old = ref.getAndSet("new");  // Atomic swap

// AtomicIntegerArray, AtomicLongArray (array elements)
AtomicIntegerArray arr = new AtomicIntegerArray(5);
arr.incrementAndGet(0);  // ++arr[0] atomically
arr.getAndSet(1, 100);   // Atomic array element swap
```

---

## Q10. LongAdder — Better Than AtomicLong for High Contention

### Concept
**LongAdder** is optimized for scenarios where many threads frequently increment/add, reducing contention by distributing updates across multiple cells. Trade-off: reads are slower (must sum all cells).

### Contention Trade-Off

```java
// Very heavy contention (100+ threads adding):
// AtomicLong: All threads compete for SAME memory location
// → CAS failures, retries, spinning
// → ~50ns per operation (contention cost)

// LongAdder: Distributes updates across multiple cells (striping)
// → Each thread increments its OWN cell, less contention
// → ~10ns per operation (much faster!)

// Light contention (0-5 threads):
// AtomicLong: Fast, direct access
// → ~5ns

// LongAdder: Overhead of finding cell
// → ~50ns (slower than atomic!)

// So: Use AtomicLong for low contention, LongAdder for high
```

### Code Comparison

```java
// AtomicLong (good for low contention)
AtomicLong atomicCount = new AtomicLong(0);
atomicCount.incrementAndGet();
long value = atomicCount.get();  // Fast read

// LongAdder (good for high contention)
LongAdder adderCount = new LongAdder();
adderCount.increment();  // Very fast with many threads
long value = adderCount.sum();  // Slower read (sums all cells)

// Production decision: Use LongAdder in high-frequency scenarios
// (metrics collection, thread-safe counters in web servers)
public class MetricsCollector {
    private final LongAdder requestCount = new LongAdder();  // ✓ Right choice
    private final LongAdder errorCount = new LongAdder();
    
    public void recordRequest() {
        requestCount.increment();  // Fast, even with 1000s of threads
    }
    
    public long getTotalRequests() {
        return requestCount.sum();  // Sums internal cells
    }
}
```

---

## Q11. BlockingQueue Variants — Queue Types

### Concept
**BlockingQueue** extends Queue with blocking operations: `put()` blocks if full, `take()` blocks if empty. Different implementations suit different scenarios.

### BlockingQueue Implementations

```java
// 1. LinkedBlockingQueue (unbounded, default)
BlockingQueue<String> queue = new LinkedBlockingQueue<>();
// No capacity limit
// Good for producer-consumer when backpressure desired
// Risk: memory leak if producer faster than consumer

// 2. ArrayBlockingQueue (bounded, fixed capacity)
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
// Exactly 10 items max
// Good for known capacity, bounded memory
// takes(full) blocks, waits for put() space

// 3. PriorityBlockingQueue (priority-based)
BlockingQueue<Task> queue = new PriorityBlockingQueue<>();
// Tasks ordered by Comparable/Comparator (not FIFO!)
//Task highest priority taken first
public class Task implements Comparable<Task> {
    int priority;
    public int compareTo(Task other) {
        return Integer.compare(this.priority, other.priority);  // Lower = higher priority
    }
}

// 4. DelayQueue (delayed elements)
DelayQueue<DelayedTask> queue = new DelayQueue<>();
// Elements removed only after their delay has passed
public class DelayedTask implements Delayed {
    long delayMs;
    long createdTime = System.nanoTime();
    
    @Override
    public long getDelay(TimeUnit unit) {
        long elapsed = System.nanoTime() - createdTime;
        return unit.convert(delayMs * 1_000_000 - elapsed, TimeUnit.NANOSECONDS);
    }
}
queue.put(new DelayedTask(5000));  // Take only after 5s delay

// 5. SynchronousQueue (no capacity!)
BlockingQueue<String> queue = new SynchronousQueue<>();
// put() blocks until someone calls take()
// take() blocks until someone calls put()
// Used by ForkJoinPool, newCachedThreadPool()

// Producer-Consumer with blocking:
class Producer implements Runnable {
    BlockingQueue<String> queue;
    
    @Override
    public void run() {
        try {
            queue.put("item");  // Blocks if queue full
        } catch (InterruptedException e) {}
    }
}

class Consumer implements Runnable {
    BlockingQueue<String> queue;
    
    @Override
    public void run() {
        try {
            String item = queue.take();  // Blocks if queue empty
            process(item);
        } catch (InterruptedException e) {}
    }
}
```

---

## Q12. Coding Challenge — Odd-Even Printer Using wait/notify

### Concept
Two threads print numbers 1-10 alternately: Thread A prints odd (1,3,5...), Thread B prints even (2,4,6...). Must synchronize using wait/notify.

### Solution

```java
public class OddEvenPrinter {
    private int counter = 1;
    private final Object lock = new Object();
    private final int max = 10;
    
    public void printOdd() throws InterruptedException {
        synchronized (lock) {
            while (counter <= max) {
                if (counter % 2 == 0) {  // Not my turn (even)
                    lock.wait();
                } else {  // My turn (odd)
                    System.out.println("ODD: " + counter);
                    counter++;
                    lock.notifyAll();
                }
            }
        }
    }
    
    public void printEven() throws InterruptedException {
        synchronized (lock) {
            while (counter <= max) {
                if (counter % 2 != 0) {  // Not my turn (odd)
                    lock.wait();
                } else {  // My turn (even)
                    System.out.println("EVEN: " + counter);
                    counter++;
                    lock.notifyAll();
                }
            }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        OddEvenPrinter printer = new OddEvenPrinter();
        
        Thread oddThread = new Thread(() -> {
            try {
                printer.printOdd();
            } catch (InterruptedException e) {}
        });
        
        Thread evenThread = new Thread(() -> {
            try {
                printer.printEven();
            } catch (InterruptedException e) {}
        });
        
        oddThread.start();
        evenThread.start();
        
        oddThread.join();
        evenThread.join();
    }
}

// Output:
// ODD: 1
// EVEN: 2
// ODD: 3
// EVEN: 4
// ODD: 5
// EVEN: 6
// ...and so on until 10
```

---

## Q33/Q121. ExecutorService — Thread Pools, Callable vs Runnable

### Concept
A **thread pool** pre-allocates worker threads and reuses them across multiple tasks, eliminating the expensive overhead of creating and destroying OS threads per request. `ExecutorService` is Java's API for managing pools. `Runnable` = fire-and-forget (no result). `Callable` = task that returns a value.

### Simple Explanation
A restaurant kitchen keeps a fixed crew of 5 chefs, not hiring a new chef per order. Orders queue on a ticket rail — chefs grab tickets when free. That's a thread pool. A `Runnable` is firing a rocket and walking away (no tracking). A `Callable` is ordering from a restaurant and getting a ticket (Future) — you can check status, wait, or cancel.

### Why Not Create Threads Directly?
```java
// ❌ BAD: Create a new thread per task
ExecutorService badPool = Executors.newCachedThreadPool();
for (Request req : requests) {
    new Thread(() -> process(req)).start();
    // Each thread: ~1MB stack, OS scheduling overhead
    // 1000 requests = 1000 threads = likely OOM or severe context switching
}

// ✅ GOOD: Thread pool — reuse 10 fixed workers
ExecutorService pool = Executors.newFixedThreadPool(10);
for (Request req : requests) {
    pool.submit(() -> process(req));  // Task queued, one of 10 workers picks it up when free
}
pool.shutdown();  // No new tasks. Queued tasks complete. Workers finish then stop.
```

### Runnable vs Callable

| Aspect | `Runnable` | `Callable<V>` |
|--------|-----------|---------------|
| Method | `run()` → void | `call()` → V |
| Checked Exceptions | Cannot throw | Can throw checked exceptions |
| Return Value | None | Via `Future<V>` |
| Use Case | Fire-and-forget (logging, notifications) | Need result or exception handling |
| Introduction | Java 1.0 | Java 5 |

### Code — Runnable vs Callable

```java
// Runnable — fire and forget, no result
pool.execute(() -> System.out.println("Running"));  // No return, fire-and-forget

// Callable — returns a value
Future<Integer> future = pool.submit(() -> computeHeavyTask());  // Called at syntax

// Check multiple states
future.isDone();      // true if completed (success or exception)
future.isCancelled(); // true if cancel(true) was called before completion

// Get result (BLOCKS until task finishes or timeout)
try {
    Integer result = future.get(5, TimeUnit.SECONDS);  // Wait max 5 seconds
} catch (TimeoutException e) {
    future.cancel(true);  // Cancel if too slow (interrupts the task thread)
} catch (ExecutionException e) {
    // Task threw an exception — unwrap it
    log.error("Task failed", e.getCause());  // Original exception
}
```

### ExecutorService Types

```java
// 1. Fixed Thread Pool — predictable concurrency for CPU-bound work
ExecutorService fixed = Executors.newFixedThreadPool(4);
// 4 threads always alive. Tasks queue if all busy.
// ✓ Predictable resource usage, good for batch processing
// Sizing: CPU-bound = availableProcessors()

// 2. Cached Thread Pool — elastic for short-lived I/O tasks
ExecutorService cached = Executors.newCachedThreadPool();
// Creates threads on demand, reuses idle ones (60s timeout removes unused)
// ✓ Great for bursty I/O (web requests, DB queries)
// ❌ DANGER: no upper bound on thread count — can OOM under sustained load

// 3. Single Thread Executor — guaranteed sequential FIFO
ExecutorService single = Executors.newSingleThreadExecutor();
// Exactly 1 thread; all tasks run one-after-another in submission order
// ✓ Prevents race conditions for shared mutable state
// Use: audit log writers, rate limiters, stateful background tasks

// 4. Scheduled Thread Pool — periodic and delayed tasks
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
scheduler.scheduleAtFixedRate(() -> cleanCache(), 0, 1, TimeUnit.HOURS);
scheduler.schedule(() -> sendReminder(), 30, TimeUnit.MINUTES);

// 5. Virtual Thread Executor (Java 21) — scales to millions
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
// Creates a new virtual thread per task — cheap! (~few KB vs 1MB for OS thread)
// Perfect for high-concurrency I/O: 1M concurrent requests
// No pooling needed — JVM manages unmounting/mounting on blocking calls
```

### execute() vs submit() — Critical Difference

```java
// execute(Runnable) — fire and forget, NO return value or exception visibility
pool.execute(() -> System.out.println("Running"));
// If task throws exception: will be in default uncaught exception handler

// submit(Callable) → Future — can get result, check status, handle exceptions
Future<Integer> future = pool.submit(() -> {
    if (someCondition) throw new RuntimeException("Task failed");
    return 42;
});

// Get result (BLOCKS until task finishes or timeout)
try {
    Integer result = future.get(5, TimeUnit.SECONDS);
    log.info("Task succeeded, result: {}", result);
} catch (TimeoutException e) {
    future.cancel(true);  // Interrupt the task thread and give up
} catch (ExecutionException e) {
    log.error("Task threw: {}", e.getCause());  // Unwrap original exception
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  //Re-interrupt on interrupted
    throw new RuntimeException("Interrupted while waiting", e);
}

// submit(Runnable, T result) → Future<T> — returns a result after void task
String taskId = UUID.randomUUID().toString();
Future<String> future = pool.submit(() -> log.info("Task done"), taskId);
// If no exception, future.get() returns the provided result object

// invokeAll — wait for many Callables with timeout
List<Callable<UserDto>> tasks = userIds.stream()
    .map(id -> (Callable<UserDto>) () -> userService.fetchUser(id))
    .collect(Collectors.toList());

List<Future<UserDto>> futures = executor.invokeAll(tasks, 30, TimeUnit.SECONDS);
// Waits for all to complete or timeout expires
// Remaining incomplete tasks are cancelled
```

### Proper Shutdown — Critical Pattern

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// ... submit many tasks ...

// GRACEFUL SHUTDOWN (production pattern):
pool.shutdown();                                    // Rejects new tasks; queued tasks finish
boolean terminated = pool.awaitTermination(30, TimeUnit.SECONDS);
if (!terminated) {
    // Tasks still running after 30s — force stop
    List<Runnable> abandoned = pool.shutdownNow();  // Interrupts all running tasks
    log.warn("{} tasks were abandoned", abandoned.size());
}
```

### Real-World Pattern — Dashboard Parallel Service Calls

```java
@Service
public class DashboardService {
    private final UserService userService;
    private final OrderService orderService;
    private final ProductService productService;
    
    // Dedicated executor for I/O tasks (not blocking common pool)
    private final Executor ioExecutor = Executors.newFixedThreadPool(10);

    public DashboardDTO loadDashboard(Long userId) {
        // Fire off three independent service calls in parallel
        Future<User> userFuture = (Future<User>) ioExecutor.submit(() -> userService.getUser(userId));
        Future<List<Order>> ordersFuture = ioExecutor.submit(() -> orderService.getOrders(userId));
        Future<List<Product>> productsFuture = ioExecutor.submit(() -> productService.getRecommended(userId));

        try {
            User user = userFuture.get(3, TimeUnit.SECONDS);
            List<Order> orders = ordersFuture.get(3, TimeUnit.SECONDS);
            List<Product> products = productsFuture.get(3, TimeUnit.SECONDS);
            return new DashboardDTO(user, orders, products);
        } catch (TimeoutException e) {
            userFuture.cancel(true);
            ordersFuture.cancel(true);
            productsFuture.cancel(true);
            return DashboardDTO.empty();
        }
    }
}
```

### Interview Answer
> "Thread pools eliminate the ~1MB stack allocation and OS scheduling overhead of creating threads per task. `ExecutorService` is the API. For sizing: CPU-bound work = `availableProcessors()` threads. I/O-bound work = apply the formula `nCores × (1 + wait_time/compute_time)` — for DB-heavy workloads (95% wait), that's roughly `nCores × 20`.
>
> `execute()` is fire-and-forget for Runnable. `submit()` returns a `Future` — this is the critical difference. With `submit()` I can get the result, check `isDone()`, and critically, I can call `get(timeout)` with a timeout to avoid indefinite waits.
>
> `newFixedThreadPool` is predictable for CPU tasks. `newCachedThreadPool` elastically grows for spiky I/O but has no ceiling — dangerous under sustained load. For I/O-bound work in production I use a custom `newFixedThreadPool` with a bounded queue. `newSingleThreadExecutor` when I need guaranteed sequential ordering.
>
> Always use `shutdown()` then `awaitTermination()` to ensure cleanliness — prevents threads from running after the application should have stopped."
>
> *Likely follow-up: "How do you size a thread pool for I/O-bound vs CPU-bound tasks?"*
> (Answer: CPU = `nCores`. I/O = `nCores × (1 + wait_time/compute_time)`. DB-heavy = roughly `nCores × 20`.)

---

## Q122. ThreadPoolExecutor — Tuning, Rejection Policies & Spring @Async

### Concept
`ThreadPoolExecutor` is the underlying implementation behind all `Executors` factory methods. Understanding its core parameters lets you tune thread pools precisely for your workload.

### Core ThreadPoolExecutor Parameters

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                               // corePoolSize: always-on threads
    20,                              // maximumPoolSize: max threads during burst
    60L, TimeUnit.SECONDS,           // keepAliveTime: lifetime of extra threads
    new LinkedBlockingQueue<>(100),  // workQueue: bounded task buffer
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejectionHandler: what to do if queue full
);
```

### Execution Flow

```
Task arrives
  → Is corePoolSize threads running?
    No  → Create new thread, execute task
    Yes → Is queue full?
      No  → Add task to queue (thread picks up when free)
      Yes → Is maxPoolSize threads running?
        No  → Create new extra thread (temporary, expires after keepAliveTime)
        Yes → FULL → invoke rejectionHandler
```

### Rejection Policies — What Happens When Thread Pool is Saturated

```java
// Policy 1: AbortPolicy — DEFAULT, throws exception
new ThreadPoolExecutor.AbortPolicy()
// Throws RejectedExecutionException immediately
// ✓ Fast-fail, clear to caller pool is overloaded
// ❌ Exceptions cascade, callers must handle

// Policy 2: CallerRunsPolicy — BEST for backpressure
new ThreadPoolExecutor.CallerRunsPolicy()
// Submitting thread executes the task itself
// ✓ Natural backpressure — slows down producer automatically
// ❌ Temporarily blocks the web thread, but prevents OOM
// Use when: overload is temporary, producer can handle brief blocking

// Policy 3: DiscardPolicy — silently drops task
new ThreadPoolExecutor.DiscardPolicy()
// Task disappears silently
// ✓ For non-critical work (metrics, analytics, optional logging)
// ❌ Data loss, hard to debug

// Policy 4: DiscardOldestPolicy — drops oldest queued task
new ThreadPoolExecutor.DiscardOldestPolicy()
// Removes first task in queue, adds new one
// For priority systems where newer tasks matter more

// Custom rejection handler
executor.setRejectedExecutionHandler((task, exec) -> {
    log.error("Task rejected: {} from {}", task, exec);
    deadLetterQueue.add(task);           // Persist failed task
    metrics.increment("thread.pool.rejected");
});
```

### Queue Types (Critical for Behavior)

```java
// LinkedBlockingQueue — unbounded by default (DANGEROUS!)
new LinkedBlockingQueue<>()        // Can grow to Integer.MAX_VALUE → OOM
new LinkedBlockingQueue<>(100)     // ✓ BOUNDED — predictable memory, triggers rejection at 100 tasks

// ArrayBlockingQueue — bounded, fixed-size (more memory predictable)
new ArrayBlockingQueue<>(100)      // Fixed array, slightly better cache locality

// SynchronousQueue — no buffer!
new SynchronousQueue<>()           // Each task must be immediately handed to a thread
// If no thread available → immediately grow to maxPoolSize or reject
// Used by newCachedThreadPool() to avoid accumulating unbounded queue

// PriorityQueue — prioritize tasks
new PriorityQueue<>()              // Higher priority tasks executed first (but NOT FIFO)
// Useful: process critical requests before batch jobs
```

### Spring `@Async` Configuration (Production Pattern)

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);                           // Always 10 threads ready
        executor.setMaxPoolSize(50);                            // Burst up to 50 in traffic spikes
        executor.setQueueCapacity(200);                         // Buffer 200 tasks if all busy
        executor.setThreadNamePrefix("async-");                 // Thread name for jstack debugging
        executor.setRejectedExecutionHandler(
            new ThreadPoolTaskExecutor.CallerRunsPolicy()       // Producer thread executes if overload
        );
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method '{}' threw exception: {}",
                method.getName(), ex.getMessage(), ex);
        };
    }
}

@Service
public class NotificationService {
    
    // Fire-and-forget email sending
    @Async
    public CompletableFuture<Void> sendWelcomeEmail(String email) {
        emailClient.send(email, "Welcome!");
        return CompletableFuture.completedFuture(null);
    }

    // Async with return value
    @Async
    public CompletableFuture<String> generateReport(Long userId) {
        String report = reportEngine.build(userId);
        return CompletableFuture.completedFuture(report);
    }
}

// In controller — Spring handles async automatically
@PostMapping("/reports/{userId}")
public CompletableFuture<String> startReport(@PathVariable Long userId) {
    return notificationService.generateReport(userId);  // Spring waits for future completion
}
```

### Monitoring Thread Pool Health

```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) asyncExecutor;

// Log current state
log.info("Pool Size: {}, Active Threads: {}, Queued Tasks: {}, Completed: {}",
    executor.getPoolSize(),           // Current running threads
    executor.getActiveCount(),        // Busy threads right now
    executor.getQueue().size(),       // Tasks waiting in queue
    executor.getCompletedTaskCount()  // Historical completed count
);

// Alert when queue is near capacity
if (executor.getQueue().size() > (executor.getMaximumPoolSize() * 2)) {
    log.warn("Thread pool queue is saturated — consider scaling or optimizing");
    metrics.gauge("thread.pool.queue.depth", executor.getQueue().size());
}
```

### Interview Answer
> "ThreadPoolExecutor has three key queues: corePoolSize (always running), maximumPoolSize (burst capacity), and queue (buffer). Execution: fill core threads first, then queue, then extra threads, then reject. LinkedBlockingQueue must be bounded — unbounded queues hide overload problems until OOM. CallerRunsPolicy is my preferred rejection policy — it's natural backpressure, the submitting thread executes the task, slowing the producer automatically. In Spring I configure `@Async` via `AsyncConfigurer` with sensible defaults: cores = 10, max = 50, queue = 200, rejection = CallerRunsPolicy.
>
> Sizing heuristic: cores = `Runtime.getRuntime().availableProcessors()` for CPU-bound, or 2-3× cores for heavy I/O. Max should be high enough for burst but not astronomical — I typically use cores × 4 or 5. Monitor queue depth — above 80% full signals scale-up is needed."
>
> *Likely follow-up: "What's the difference between CallerRunsPolicy and DiscardPolicy?"*

---

## Q34. Deadlocks — Concepts, Detection & Prevention

### Concept
A **deadlock** is a situation where two or more threads are permanently blocked, each holding a lock that another thread needs, with no way to break the cycle. Multiple threads wait forever, making no progress.

### Simple Explanation
Thread A has a knife, wants a fork. Thread B has a fork, wants a knife. Neither will give up what they have until they get what they want. Both wait forever.

### The Four Coffman Conditions — ALL Must Hold for Deadlock

1. **Mutual Exclusion** — Resources cannot be shared simultaneously (only one thread can hold a lock)
2. **Hold and Wait** — A thread holds one resource while waiting for another
3. **No Preemption** — Locks cannot be forcibly taken from a thread; only the holder can release
4. **Circular Wait** — Thread A waits for Thread B's lock, Thread B waits for Thread A's lock (cycle)

To prevent deadlock, eliminate ANY ONE of these four conditions.

### Code — Classic Deadlock Scenario

```java
Object lockA = new Object();
Object lockB = new Object();

Thread thread1 = new Thread(() -> {
    synchronized (lockA) {
        System.out.println("Thread1: ✓ locked A");
        try { Thread.sleep(100); } catch (InterruptedException e) {}  // Give thread2 time to get B
        
        synchronized (lockB) {
            System.out.println("Thread1: ✓ locked both A and B");
        }
    }
}, "Thread-1");

Thread thread2 = new Thread(() -> {
    synchronized (lockB) {
        System.out.println("Thread2: ✓ locked B");
        
        synchronized (lockA) {
            System.out.println("Thread2: ✓ locked both A and B");
        }
    }
}, "Thread-2");

thread1.start();
thread2.start();

// OUTPUT: Thread1 locks A, Thread2 locks B, both wait forever → DEADLOCK!
```

### Prevention Strategy 1: Consistent Lock Ordering (Most Reliable)

```java
// ✅ FIX: BOTH threads always acquire locks in the SAME order
// If every thread always takes lockA BEFORE lockB, circular wait is impossible

Thread thread1 = new Thread(() -> {
    synchronized (lockA) {
        synchronized (lockB) {
            System.out.println("Thread1: ✓ got both locks in consistent order");
        }
    }
});

Thread thread2 = new Thread(() -> {
    synchronized (lockA) {   // Same order as thread1 — lockA FIRST, then lockB
        synchronized (lockB) {
            System.out.println("Thread2: ✓ got both locks in consistent order");
        }
    }
});

// Real-world example: Money transfer between accounts
// Always lock the lower account ID first to avoid circular waits
class Account {
    private int id;
    private int balance;

    public static void transfer(Account from, Account to, int amount) {
        // Order by account ID — prevents circular dependency
        Account first  = from.id < to.id ? from : to;
        Account second = from.id < to.id ? to : from;

        synchronized (first) {
            synchronized (second) {
                if (from.id == first.id) {
                    from.balance -= amount;
                    to.balance += amount;
                } else {
                    from.balance -= amount;
                    to.balance += amount;
                }
            }
        }
    }
}
```

### Prevention Strategy 2: TimeOut with tryLock (Non-blocking Acquisition)

```java
// ✅ Instead of indefinite wait, give up after timeout
ReentrantLock lockA2 = new ReentrantLock();
ReentrantLock lockB2 = new ReentrantLock();

boolean success = false;
try {
    if (lockA2.tryLock(100, TimeUnit.MILLISECONDS)) {
        try {
            if (lockB2.tryLock(100, TimeUnit.MILLISECONDS)) {
                try {
                    // Do critical section work
                    success = true;
                } finally {
                    lockB2.unlock();
                }
            }
        } finally {
            lockA2.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

if (!success) {
    log.warn("Failed to acquire both locks within timeout — will retry");
    // Retry or fail gracefully instead of waiting forever
}
```

### Prevention Strategy 3: Avoid Nested Locks

```java
// ❌ Problem: holding two locks simultaneously
public synchronized void methodA() { methodB(); }   // Holds methodA's lock, acquires methodB's
public synchronized void methodB() { methodC(); }   // Holds methodB's lock, requests methodA's

// ✅ Solution: acquire all locks upfront, not nested
public void transfer(...) {
    synchronized (lockA) {
        synchronized (lockB) {
            // All critical work inside, no nested method calls that need locks
        }
    }
}
```

### Prevention Strategy 4: Use High-Level Concurrency Utilities

```java
// ✅ ConcurrentHashMap, BlockingQueue, CompletableFuture handle locking internally
// No manual lock management = no risk of deadlock

// Instead of synchronized blocks:
ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();
cache.put(key, value);  // Thread-safe, no deadlock risk

// BlockingQueue for producer-consumer, handles all synchronization
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();
queue.put(task);           // Blocks if full
Task retrieved = queue.take();  // Blocks if empty — all coordinated internally
```

### Detection — Finding Deadlocks in Production

```java
// Programmatic detection using ThreadMXBean
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = bean.findDeadlockedThreads();

if (deadlockedThreads != null) {
    ThreadInfo[] info = bean.getThreadInfo(deadlockedThreads);
    for (ThreadInfo ti : info) {
        System.out.println("DEADLOCK DETECTED!");
        System.out.println("  Thread: " + ti.getThreadName());
        System.out.println("  Waiting on: " + ti.getLockName());
        System.out.println("  Lock held by: " + ti.getLockOwnerName());
        System.out.println("  State: " + ti.getThreadState());
        
        // Print full stack trace
        for (StackTraceElement ste : ti.getStackTrace()) {
            System.out.println("    at " + ste);
        }
    }
}

// From command line: jstack <pid>
// Outputs thread dump; look for DEADLOCK block at end
// Example jstack output:
// Found one Java-level deadlock:
// =============================
// "Thread-2" ...
//   waiting to lock monitor 0x...locked by "Thread-1"
// "Thread-1" ...
//   waiting to lock monitor 0x...locked by "Thread-2"
```

### Interview Answer
> "Deadlock requires all four Coffman conditions: mutual exclusion, hold-and-wait, no preemption, and circular wait. The classic example: Thread 1 holds lockA waiting for lockB, Thread 2 holds lockB waiting for lockA. Prevention: the most reliable fix is consistent lock ordering at the design level — if every thread always acquires lockA before lockB, circular wait is impossible. I used this for money transfers by always locking the lower account ID first.
>
> For more complex scenarios or where lock ordering isn't trivial, `ReentrantLock.tryLock(timeout)` lets threads back off gracefully rather than waiting forever. If acquisition fails within the timeout, I retry.
>
> In production, I diagnose deadlocks with `jstack <pid>` (thread dump) or programmatic detection using `ThreadMXBean.findDeadlockedThreads()`. Java explicitly labels BLOCKED threads and shows who holds the lock they're waiting on.
>
> Where possible, I prefer high-level utilities like `ConcurrentHashMap` or `BlockingQueue` — they hide the locks and eliminate the risk of deadlock entirely."
>
> *Likely follow-up: "What's the difference between deadlock and livelock?"*
> (Answer: Livelock — threads actively respond to each other but make no progress, like two people stepping aside in a corridor repeatedly in the same direction. Both are stuck, but livelocked threads are thrashing CPU.)

---

## Q35. The `volatile` Keyword — Memory Visibility vs Atomicity

### Concept
`volatile` guarantees that reads and writes to a variable are always from/to **main memory** (not a CPU core's local cache), ensuring **visibility** of changes across threads. Critical limitation: `volatile` does NOT guarantee **atomicity** — compound operations like `counter++` are still not thread-safe.

### Simple Explanation
Two workers share a whiteboard (main memory) but each has a personal notepad (CPU cache for speed). Without `volatile`, Worker B reads from their stale notepad and never sees Worker A's updates. With `volatile`, every read/write must go directly to the shared whiteboard — no private notepad.

### The Problem — Visibility Without volatile

```java
class StopTask implements Runnable {
    private boolean stopped = false;   // Not volatile — lives in CPU cache
    
    public void stop() {
        stopped = true;                // Written to main thread's CPU cache
    }                                  // Main memory may never be updated!
    
    @Override
    public void run() {
        while (!stopped) {             // Worker thread reads from ITS OWN cache
            doWork();                  // May NEVER see stopped = true
        }                              // Infinite loop! Main thread set it, but cache is stale
    }
}
```

### The Solution — With volatile

```java
class StopTask implements Runnable {
    private volatile boolean stopped = false;  // ← volatile keyword
    
    public void stop() {
        stopped = true;               // Forces IMMEDIATE write to main memory
    }
    
    @Override
    public void run() {
        while (!stopped) {            // Forces read from MAIN MEMORY every iteration
            doWork();                 // Will eventually see stopped = true ✓
        }
    }
}
```

### Critical Limitation — volatile Does NOT Provide Atomicity

```java
private volatile int counter = 0;

// ❌ This looks safe but is NOT atomic!
counter++;   // This is actually THREE operations:
             // (1) READ counter from memory
             // (2) INCREMENT the value
             // (3) WRITE counter back to memory
             // Between steps, another thread can read/modify/write → lost update!

// Two threads both read counter=5:
// Thread 1: reads 5, increments to 6, writes 6
// Thread 2: reads 5, increments to 6, writes 6  ← both set to 6, one increment lost!

// volatile visibility:
// Thread 1 ensures it writes 6 to main memory (not cache)
// Thread 2 sees 6 from main memory
// But compound operation (`++`) is not indivisible → lost updates possible

// ✅ CORRECT: Use AtomicInteger for atomic operations
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();       // ✓ Atomic — one indivisible operation
counter.getAndSet(5, 6);        // ✓ compare-and-set (CAS)
```

### When to Use volatile — Correct Use Cases

```java
// ✅ Use Case 1: Simple boolean flags
private volatile boolean isRunning = true;
private volatile boolean shutdown = false;

public void stop() { shutdown = true; }

@Override
public void run() {
    while (!shutdown) {
        doWork();
    }
}

// ✅ Use Case 2: Status updates from one writer, many readers
private volatile int currentConnections = 0;

public void onConnect() { currentConnections++; }      // ⚠ Not atomic, but acceptable if single writer!
public int getConnections() { return currentConnections; }

// ✅ Use Case 3: CRITICAL — Double-checked locking Singleton Pattern
public class Singleton {
    private static volatile Singleton instance;  // ← volatile IS REQUIRED HERE!

    public static Singleton getInstance() {
        if (instance == null) {                    // First check, no locking (fast path)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check inside lock
                    instance = new Singleton();    // Without volatile: STILL WRONG!
                }
            }
        }
        return instance;
    }
}
// WHY volatile is critical here:
// Object construction is THREE steps:
//   (1) Allocate memory for object
//   (2) Initialize fields (call constructor)
//   (3) Assign reference to variable
// Without volatile, JIT can REORDER to: (1) allocate, (3) assign, (2) init
// Another thread sees non-null ref but fields not yet initialized → NullPointerException!
// volatile prevents reordering (memory barrier), ensuring correct initialization order
```

### ❌ Incorrect Uses of volatile

```java
// ❌ WRONG: Using volatile for atomicity
private volatile int counter = 0;
counter++;  // Lost updates with concurrent threads

// Use instead:
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();

// ❌ WRONG: Check-then-act (TOCTOU — Time of Check, Time of Use)
if (!setContainsValue) {     // Check (even with volatile)
    set.add(value);          // Act (another thread may have added in between)
}
// Use instead:
if (set.add(value)) { /* was not present */ }  // Atomic check-and-add

// ❌ WRONG: Relying on visibility for complex invariants
private volatile int x = 0;
private volatile int y = 0;

public void setXY(int x, int y) { this.x = x; this.y = y; }
public int[] getXY() { return new int[]{x, y}; }
// Another thread may read x=0, y=1 (half-updated state)
// Use synchronized or atomics for multi-variable consistency
```

### Interview Answer
> "Volatile solves the CPU cache visibility problem. Each CPU core caches memory for speed — without volatile, a write on one thread sits in that core's cache and never reaches main memory, so other threads read stale values. Volatile adds a memory barrier: every write flushes to main memory immediately, every read fetches fresh from main memory. The critical limitation: volatile does NOT provide atomicity. `counter++` is three operations — read, increment, write — and even with volatile, two threads can both read 5, both increment to 6, and both write 6 → lost update. Use `AtomicInteger` when you need atomic operations.
>
> The most important correct use is the double-checked locking Singleton pattern. Without volatile, the JIT optimizer can reorder the three steps of object construction — allocate memory, initialize fields, assign reference. This can expose a partially-initialized object to other threads. Volatile prevents this reordering with a memory barrier, guaranteeing that initialization happens before the reference is assigned.
>
> The happens-before guarantee: a write to a volatile variable happens-before every subsequent read of that variable."
>
> *Likely follow-up: "Explain happens-before relationships in the Java Memory Model."*

---

## Q124. Synchronization Mechanisms — `synchronized` vs `ReentrantLock`

### Concept
Both `synchronized` (implicit, managed by JVM) and `ReentrantLock` (explicit, manual) protect critical sections from concurrent modification. `ReentrantLock` offers more control and advanced features.

### Simple Explanation
- **synchronized** = A bathroom key on a hook — grab it, use, return it. Simple, automatic.
- **ReentrantLock** = An electronic key card — try to swipe (tryLock), set time limits, choose fairness rules, signal specific waiting threads.

### Code — Basic Comparison

```java
// ❌ synchronized — implicit, JVM-managed
public class CounterSync {
    private int count = 0;
    
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

// ✅ ReentrantLock — explicit, fine control
public class CounterLock {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // MUST be in finally — prevents deadlock if exception occurs
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

```java
// Feature 1: Non-blocking tryLock with timeout
ReentrantLock lock = new ReentrantLock();

if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        // Execute critical section
        processOrder(order);
    } finally {
        lock.unlock();
    }
} else {
    // Timeout — lock not acquired
    throw new ServiceUnavailableException("Resource busy, try again later");
}

// Feature 2: Fairness (FIFO) — prevents thread starvation
ReentrantLock fairLock = new ReentrantLock(true);  // true = fair acquisition (FIFO)
// Threads acquire lock in the order they requested (no "barging")
// Trade-off: fairness reduces throughput (less barging overhead)

ReentrantLock unfairLock = new ReentrantLock(false);  // default = unfair
// Thread waiting to be unlocked can race with incoming threads
// Higher throughput but risk of starvation for some threads

// Feature 3: Condition variables — multiple wait sets
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();   // Condition for "buffer not full"
Condition notEmpty = lock.newCondition();  // Condition for "buffer not empty"

class BoundedBuffer<T> {
    private final Queue<T> buffer = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == capacity) {
                notFull.await();  // Wait until notified another thread took an item
            }
            buffer.add(item);
            notEmpty.signal();    // Wake ONE waiting consumer
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                notEmpty.await();  // Wait until notified producer added an item
            }
            T item = buffer.poll();
            notFull.signal();      // Wake ONE waiting producer
            return item;
        } finally {
            lock.unlock();
        }
    }
}

// Feature 4: ReadWriteLock — multiple readers OR one writer
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

class PriceCache {
    private final Map<String, Double> prices = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    public double getPrice(String symbol) {  // Multiple threads can read simultaneously
        rwLock.readLock().lock();
        try {
            return prices.getOrDefault(symbol, 0.0);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public void setPrice(String symbol, double price) {  // Only one writer at a time; blocks all readers
        rwLock.writeLock().lock();
        try {
            prices.put(symbol, price);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}

// When reads >> writes: ReadWriteLock = 5-10× faster than synchronized
```

### Comparison Table

| Aspect | `synchronized` | `ReentrantLock` |
|--------|---|---|
| Acquisition | Implicit | Manual (`lock()`) |
| Release | Automatic | Must be in `finally` |
| Timeout | ❌ No | ✅ `tryLock(timeout)` |
| Fairness | ❌ No | ✅ `ReentrantLock(fair=true)` |
| Multiple conditions | ❌ One wait set only | ✅ Multiple `Condition` variables |
| JIT optimization | ✅ (biased locking) | ❌ Less optimization |
| Interruptible locking | ❌ | ✅ `lockInterruptibly()` |

### When to Use Which

```java
// ✅ Use synchronized:
// - Simple critical sections
// - Low contention (rarely contested)
// - Let JVM optimize (biased locking, lock elision)

public synchronized void increment() { count++; }

// ✅ Use ReentrantLock:
// - Timeout-based acquisition needed
// - Fairness required (prevent starvation)
// - Multiple wait conditions (producer-consumer)
// - Read-heavy structures (ReadWriteLock)

// ✅ Prefer Lock-Free (atomic) for counter/flag:
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // No locking overhead

// ✅ Prefer high-level utilities:
ConcurrentHashMap<K, V> — thread-safe map
BlockingQueue<T> — producer-consumer
```

### Interview Answer
> "Both `synchronized` and `ReentrantLock` protect critical sections, but with different tradeoffs. `synchronized` is simpler — method or block level, lock auto-released on exit. The JVM can optimize it with biased locking (if only one thread accesses it) and lock elision (remove it entirely if memory analysis proves it's safe). Use it for simple critical sections.
>
> `ReentrantLock` is explicit — you call `lock()` and `unlock()` in a try-finally, but you get: `tryLock(timeout)` to avoid indefinite waits (critical for preventing deadlocks), fairness flag (FIFO acquisition prevents thread starvation), Condition variables (multiple wait sets — much cleaner than `wait()/notify()`), and `lockInterruptibly()` (let threads respond to interrupts).
>
> My modern preference: avoid manual locking where possible. Use `AtomicInteger` for simple counters (lock-free CAS). `ConcurrentHashMap` for thread-safe maps. `BlockingQueue` for producer-consumer (encapsulates all synchronization). `ReadWriteLock` for read-heavy structures where reads >> writes.
>
> For the double-checked locking pattern or complex scenarios like multiple wait conditions in a bounded buffer, I use `ReentrantLock`."
>
> *Likely follow-up: "What is `wait()/notify()` and how does it differ from Condition variables?"*

---

## Q36/Q123. Parallel Processing — Streams vs ForkJoinPool

### Concept
Java offers three levels of parallel processing: **parallel streams** (easiest), **CompletableFuture** (composition), and **ForkJoinPool** (divide-and-conquer). Choose based on workload.

### Parallel Streams — When They Win and Lose

```java
// ✅ Parallel WINS: CPU-bound, large dataset (100k+ elements), stateless operations
List<User> users = largeList;  // 1M users
List<String> hashes = users.parallelStream()
    .map(user -> bcrypt.hash(user.getPassword()))  // CPU-intensive hash computation
    .collect(Collectors.toList());
// 8 cores = ~8× speedup

// ❌ DANGER: I/O-bound work blocks ForkJoinPool
List<UserDTO> users = ids.parallelStream()
    .map(id -> userRepository.findById(id).orElseThrow())  // ❌ DB I/O blocks all workers!
    .collect(Collectors.toList());
// All 8 workers blocked waiting for database
// Other parallel streams in JVM also starved
// Use CompletableFuture with dedicated executor instead

// ❌ DANGER: Shared mutable state without thread safety
List<Integer> results = new ArrayList<>();  // ArrayList is NOT thread-safe
numbers.parallelStream()
    .forEach(n -> results.add(n * 2));  // Race condition! Data corruption
// Fix: use thread-safe collectors
List<Integer> safe = numbers.parallelStream()
    .map(n -> n * 2)
    .collect(Collectors.toList());  // Collectors handle thread safety

// ❌ Order doesn't matter: avoid ordered operations
numbers.parallelStream()
    .forEach(System.out::println);     // Random order
// If you need order:
numbers.parallelStream()
    .forEachOrdered(System.out::println);  // Ordered but negates speed benefit!

// ⚠ Sequential first, parallelize if slow:
List<String> data = files.stream()  // Start sequential
    .map(File::read)
    .filter(x -> x.length() > 100)
    .collect(Collectors.toList());

if (data.size() > 100000) {  // Only parallelize large datasets
    data = data.parallelStream().map(this::slowTransform).collect(Collectors.toList());
}
```

### Sequential vs Parallel Comparison

```java
// Sequential — predictable, single-threaded
long count = orders.stream()
    .filter(o -> o.getAmount() > 1000)
    .count();

// Parallel — uses ForkJoinPool.commonPool() (default: availableProcessors() - 1)
long count = orders.parallelStream()
    .filter(o -> o.getAmount() > 1000)
    .count();  // Splits list into chunks, processes in parallel, merges results

// Custom ForkJoinPool (control thread count for I/O-heavy parallel streams)
ForkJoinPool customPool = new ForkJoinPool(8);
long result = customPool.submit(() ->
    orders.parallelStream()
          .filter(o -> o.getAmount() > 1000)
          .count()
).get();

// Ordering:
Stream<T> sequential = data.stream().sorted();      // Single-pass sort, ordered
Stream<T> parallel = data.parallelStream().sorted();  // Multi-pass merge sort

// findFirst vs findAny
data.stream().findFirst();           // Return first in encounter order
data.parallelStream().findFirst();   // Still returns first, but slower (must respect order)
data.parallelStream().findAny();     // Return any element — faster, no order guarantee
```

### ForkJoinPool — Divide and Conquer for CPU-Bound Tasks

```java
// RecursiveTask: parallel computation that returns a result
class SumTask extends RecursiveTask<Long> {
    private final int[] array;
    private final int from, to;
    private static final int THRESHOLD = 1000;  // Split if > 1000 elements
    
    @Override
    protected Long compute() {
        if (to - from <= THRESHOLD) {
            // Base case: small enough to compute directly (no parallelism)
            long sum = 0;
            for (int i = from; i < to; i++) sum += array[i];
            return sum;
        }
        
        // Recursive case: split in half and process in parallel
        int mid = (from + to) / 2;
        SumTask left = new SumTask(array, from, mid);
        SumTask right = new SumTask(array, mid, to);
        
        left.fork();                    // Submit left task asynchronously
        long rightResult = right.compute();  // Compute right on current thread
        long leftResult = left.join();       // Wait for left task to finish
        
        return leftResult + rightResult;  // Combine results
    }
}

// Execute
ForkJoinPool pool = ForkJoinPool.commonPool();
long total = pool.invoke(new SumTask(bigArray, 0, bigArray.length));

// RecursiveAction: parallel task that doesn't return a value
class ForEachTask extends RecursiveAction {
    private final int[] array;
    private final int from, to;
    private final Consumer<Integer> action;
    private static final int THRESHOLD = 1000;
    
    @Override
    protected void compute() {
        if (to - from <= THRESHOLD) {
            for (int i = from; i < to; i++) action.accept(array[i]);
            return;
        }
        
        int mid = (from + to) / 2;
        ForEachTask left = new ForEachTask(array, from, mid, action);
        ForEachTask right = new ForEachTask(array, mid, to, action);
        
        invokeAll(left, right);  // Fork both, wait for both
    }
}
```

### Interview Answer
> "Parallel streams are easiest but require understanding when they work. They win for CPU-bound operations on large datasets (100k+ elements) — bulk hashing, data transformation, aggregation. They lose for I/O because all workers block on the same ForkJoinPool, starving other parallel streams in the JVM. For I/O I use CompletableFuture with a dedicated executor.
>
> Rules for parallel streams:
> 1. Stateless lambdas (no side effects, no shared mutable state)
> 2. Always use thread-safe collectors — not `forEach` + ArrayList
> 3. Dataset ≥ 10,000 elements (overhead shouldn't exceed speedup)
> 4. CPU-bound workload (not I/O)
> 5. Optimize to sequential first, parallelize only if profiling shows need
>
> For divide-and-conquer algorithms, I use `RecursiveTask` in `ForkJoinPool` — merge sort, tree traversal, large array aggregations. The framework automatically distributes work across cores and balances load."
>
> *Likely follow-up: "How do you choose THRESHOLD for RecursiveTask?"*

---

## Q37. CompletableFuture — Async Composition & Non-Blocking Pipelines

### Concept
`CompletableFuture<T>` is Java's promise abstraction for asynchronous result composition. It chains transformations, combines multiple async operations, and handles errors without blocking threads.

### Core Methods — runAsync vs supplyAsync

```java
// runAsync: void side effect (returns CompletableFuture<Void>)
CompletableFuture<Void> logTask = CompletableFuture.runAsync(() -> {
    auditLogger.log("User login");  // Side effect, no result
});

// supplyAsync: returns a value (returns CompletableFuture<T>)
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> {
    return userRepository.findById(id);  // Returns User
});

// Both default to ForkJoinPool.commonPool()
// ⚠ CRITICAL: custom executor for I/O to avoid blocking common pool
Executor ioExecutor = Executors.newFixedThreadPool(20);
CompletableFuture<User> userF = CompletableFuture.supplyAsync(
    () -> userRepository.findById(id),
    ioExecutor  // Use dedicated I/O thread pool, not common pool
);
```

### Chaining Transformations — thenApply, thenCompose, thenAccept

```java
// thenApply: synchronous transform (like map() on streams)
CompletableFuture<String> pipeline = userFuture
    .thenApply(user -> user.getName())           // User → String (sync)
    .thenApply(name -> name.toUpperCase());      // String → String (sync)

// thenApplyAsync: asynchronous transform (submits to executor)
CompletableFuture<String> pipeline2 = userFuture
    .thenApplyAsync(user -> expensiveComputation(user));  // Runs on new thread

// ⚠️ CRITICAL: thenApply vs thenCompose
// If transform returns a PLAIN value → thenApply
CompletableFuture<String> name = userFuture
    .thenApply(user -> user.getName());  // User → String, ✓ correct

// If transform returns a COMPLETABLE FUTURE → thenCompose (flat-map)
CompletableFuture<Order> latestOrder = userFuture
    .thenCompose(user -> orderService.getLatestOrderAsync(user.getId()));
    // Without thenCompose: CompletableFuture<CompletableFuture<Order>> — WRONG (nested)!

// thenAccept: consume result, no return
userFuture.thenAccept(user -> {
    log.info("Loaded user: {}", user.getName());  // Side effect, no return
});

// thenRun: no input, no return
CompletableFuture<Void> task = userFuture
    .thenRun(() -> log.info("All done"));
```

### Combining Multiple Futures

```java
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> getUser(id));
CompletableFuture<Account> accountF = CompletableFuture.supplyAsync(() -> getAccount(id));
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> getOrders(id));

// thenCombine: wait for TWO futures, combine when both complete
CompletableFuture<Dashboard> dashboard = userF
    .thenCombine(accountF, (user, account) ->
        new Dashboard(user, account)
    );

// allOf: wait for ALL futures
CompletableFuture<Void> allDone = CompletableFuture.allOf(userF, accountF, ordersF);
allDone.thenRun(() -> {
    User user = userF.join();
    Account account = accountF.join();
    List<Order> orders = ordersF.join();
    buildDashboard(user, account, orders);
});

// anyOf: proceed when FIRST completes
CompletableFuture<Object> fastest = CompletableFuture.anyOf(fastServer, slowServer);
fastest.thenAccept(result -> log.info("First result: {}", result));  // One of them

// Collecting multiple results
List<CompletableFuture<User>> userFutures = userIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> userService.getUser(id), ioPool))
    .collect(Collectors.toList());

CompletableFuture<Void> allUsers = CompletableFuture.allOf(
    userFutures.toArray(new CompletableFuture[0])
);

CompletableFuture<List<User>> allUsersList = allUsers.thenApply(v ->
    userFutures.stream().map(CompletableFuture::join).collect(Collectors.toList())
);
```

### Exception Handling

```java
CompletableFuture<User> safe = CompletableFuture
    .supplyAsync(() -> userRepository.findById(id))
    
    // exceptionally: runs ONLY on failure, provides fallback value
    .exceptionally(ex -> {
        log.error("Failed to fetch user", ex);
        return User.anonymous();  // Fallback
    })
    
    // handle: runs on BOTH success and failure (like try-finally)
    .handle((user, ex) -> {
        if (ex != null) {
            log.error("Failed", ex);
            return User.anonymous();
        }
        return user;
    })
    
    // whenComplete: side effect (logging, cleanup) — doesn't change result
    .whenComplete((user, ex) -> {
        if (ex != null) metrics.increment("user.fetch.failures");
        else            metrics.increment("user.fetch.successes");
    })
    
    // Recovery: convert CompletionException to something useful
    .exceptionally(ex -> {
        Throwable cause = ex instanceof CompletionException 
            ? ex.getCause() 
            : ex;
        if (cause instanceof EntityNotFoundException) {
            return User.anonymous();
        }
        throw new RuntimeException(cause);
    });
```

### get() vs join() — Blocking Extraction

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(2000);
    return "result";
});

// get() — synchronously blocks, throws checked exceptions
String result = future.get();              // Blocks indefinitely (DANGEROUS in production!)
String result = future.get(3, TimeUnit.SECONDS);  // Blocks max 3s → TimeoutException

// join() — synchronously blocks, throws unchecked CompletionException
String result = future.join();             // Cleaner for non-reactive code

// isDone() — non-blocking check
if (future.isDone()) {
    String result = future.join();  // Won't block since isDone() is true
}

// ⚠️ DANGER: never call get()/join() on EventLoop thread
// In Spring WebFlux (Netty event loop), blocking the thread kills all request processing
// Instead, return CompletableFuture directly; framework handles async

@PostMapping("/users/{id}")
public CompletableFuture<UserDTO> getUser(@PathVariable Long id) {
    return userService.fetchUserAsync(id);  // Return future, don't block
    // Spring handles waiting and response serialization
}
```

### Real-World Pattern — Parallel Downstream Service Calls

```java
@Service
public class DashboardService {
    private final UserService userService;
    private final OrderService orderService;
    private final ProductService productService;
    
    // Dedicated executor for I/O to avoid blocking common ForkJoinPool
    private final Executor ioPool = Executors.newFixedThreadPool(20);
    
    public CompletableFuture<DashboardDTO> loadDashboard(Long userId) {
        // Fire off three independent service calls in parallel
        var userFuture = CompletableFuture.supplyAsync(
            () -> userService.getUser(userId), 
            ioPool
        );
        var ordersFuture = CompletableFuture.supplyAsync(
            () -> orderService.getOrders(userId),
            ioPool
        );
        var productsFuture = CompletableFuture.supplyAsync(
            () -> productService.getRecommended(userId),
            ioPool
        );
        
        // Wait for all three, then combine
        return CompletableFuture.allOf(userFuture, ordersFuture, productsFuture)
            .thenApply(v -> new DashboardDTO(
                userFuture.join(),
                ordersFuture.join(),
                productsFuture.join()
            ))
            .exceptionally(ex -> {
                log.error("Dashboard load failed", ex);
                return DashboardDTO.empty();
            });
        
        // Sequential would take sum of all three latencies
        // Parallel takes MAX of all three latencies
        // Example: 300ms + 400ms + 250ms = 350ms (max), not 950ms (sum)
    }
}
```

### Interview Answer
> "CompletableFuture is Java's promise for async composition. `runAsync` is fire-and-forget (side effects), `supplyAsync` returns a value. Key transforms: `thenApply` for sync in-memory transforms (map), `thenCompose` for flat-map when your function returns another CompletableFuture — missing this creates nested futures which is wrong.
>
> Combining: `thenCombine` for two futures, `allOf` for many. String concatenations, `anyOf` for racing (first wins). `get()` blocks with checked exceptions — use sparingly with timeout. `join()` is cleaner for use inside chains, throws unchecked `CompletionException`.
>
> Critical production rule: always use a custom `Executor` for async I/O tasks — never let I/O jobs use the default `ForkJoinPool.commonPool()`. Blocking those threads starves all other parallel streams in the JVM.
>
> In my project I used `allOf` + individual `join()` to parallelize three downstream service calls for a dashboard page. Response time dropped from ~900ms sequential (sum of latencies) to ~350ms (max of latencies) — three independent services, only the slowest one matters in parallel."
>
> *Likely follow-up: "What's the difference between thenApply and thenApplyAsync?"*
> (Answer: `thenApply` runs on the thread that completed the previous stage — could be a worker thread if `supplyAsync` used a pool. `thenApplyAsync` always submits as a new task to the executor — useful if the callback is expensive and you don't want to block the worker.)

---

## Q125. ThreadLocal & Thread Context — Per-Thread Isolated Storage

### Concept
`ThreadLocal<T>` provides thread-isolated storage — each thread gets its own independent instance of a variable. Changes by one thread are invisible to others. Used for request context (user, transaction, correlation ID), connection pools, and avoiding parameter passing through deep call stacks.

### Simple Explanation
Each thread has its own locked drawer. Put something in your drawer — others can't see it, can't access it, can't modify it. When the thread exits, the drawer is cleaned. No one else's drawer is affected.

### The Problem — Without ThreadLocal

```java
// ❌ Problem: passing context through every method as parameter
public class UserService {
    public void processOrder(Order order) {
        Long userId = getCurrentUserId();  // How to get current user?
        // Option 1: Pass as parameter through 5 method calls
        validateOrder(order, userId);
        priceOrder(order, userId);
        saveOrder(order, userId);
    }
    
    private void validateOrder(Order order, Long userId) {
        checkInventory(order, userId);        // Pass down...
    }
    
    private void checkInventory(Order order, Long userId) {
        checkStock(order, userId);             // And down...
    }
    
    private void checkStock(Order order, Long userId) {
        // Finally use userId here — but had to thread it through 3 calls!
        log.info("Checking stock for user {}", userId);
    }
}

// ❌ Problem: static field (thread-unsafe if shared across threads)
public class RequestContext {
    public static String currentUserId;  // DANGER: every thread sees the same variable!
}

Thread thread1 = new Thread(() -> RequestContext.currentUserId = "user1");
Thread thread2 = new Thread(() -> RequestContext.currentUserId = "user2");
thread1.start();
thread2.start();
// Race condition: impossible to know which value other thread sees
```

### The Solution — ThreadLocal

```java
// ✅ ThreadLocal — each thread has isolated storage
public class RequestContext {
    private static final ThreadLocal<Long> userIdHolder = new ThreadLocal<>();
    private static final ThreadLocal<String> correlationIdHolder = new ThreadLocal<>();
    
    public static void setUserId(Long userId) { userIdHolder.set(userId); }
    public static Long getUserId() { return userIdHolder.get(); }
    
    public static void setCorrelationId(String id) { correlationIdHolder.set(id); }
    public static String getCorrelationId() { return correlationIdHolder.get(); }
    
    // CRITICAL: clean up when request completes
    public static void clear() {
        userIdHolder.remove();
        correlationIdHolder.remove();
    }
}

@Service
public class UserService {
    public void processOrder(Order order) {
        Long userId = RequestContext.getUserId();  // Get context without parameters
        validateOrder(order);                      // No need to pass userId
        priceOrder(order);
        saveOrder(order);
    }
    
    private void validateOrder(Order order) {
        checkInventory(order);
    }
    
    private void checkInventory(Order order) {
        checkStock(order);
    }
    
    private void checkStock(Order order) {
        Long userId = RequestContext.getUserId();  // Access from anywhere
        log.info("Checking stock for user {}", userId);
    }
}

// Spring Boot filter pattern — set context on request, clear on response
@Component
public class RequestContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // Extract user and correlation ID from request
        Long userId = (Long) httpRequest.getAttribute("userId");
        String correlationId = httpRequest.getHeader("X-Correlation-ID");
        
        try {
            // Set context for this thread (entire request processing)
            RequestContext.setUserId(userId);
            RequestContext.setCorrelationId(correlationId);
            
            chain.doFilter(request, response);  // All downstream services access context
        } finally {
            // CRITICAL: always clean up in finally
            RequestContext.clear();  // Prevent leaks to next request in thread pool
        }
    }
}

// Usage in service layer — context automatically available
@Service
public class OrderService {
    private final AuditLogger auditLogger;
    
    public void createOrder(CreateOrderRequest req) {
        Order order = new Order();
        order.setUserId(RequestContext.getUserId());  // From ThreadLocal
        
        // Later, in completely different service:
        auditLogger.logAction("ORDER_CREATED");    // Can access userId here too!
    }
}

@Service
public class AuditLogger {
    public void logAction(String action) {
        // No need to pass userId as parameter — fetch from context
        Long userId = RequestContext.getUserId();
        String correlationId = RequestContext.getCorrelationId();
        
        log.info("[{}] User {} performed {}", correlationId, userId, action);
    }
}
```

### ThreadLocal Pitfall — Memory Leaks in Thread Pools

```java
// ❌ CRITICAL DANGER: ThreadLocal in thread pools
ExecutorService pool = Executors.newFixedThreadPool(4);

for (int i = 0; i < 10; i++) {
    final int requestId = i;
    pool.submit(() -> {
        RequestContext.setUserId(12345L);     // Set in thread from pool
        // Process request...
        // FORGOT TO CALL RequestContext.clear()  ← MEMORY LEAK!
    });
}

// Thread 1 from pool processes request 1: userId = 12345
// Thread 1 is returned to pool
// Thread 1 processes request 2: userId is STILL 12345!  ← Wrong user context!
// This is a DATA LEAK and SECURITY ISSUE!

// The ThreadLocal map in Thread lives for the thread's lifetime
// If you don't remove entries, they persist until thread is recycled
// With 4 threads × 1MB per request context = 4MB lost after 1000 requests
// This causes OOM in high-traffic apps

// ✅ FIX: Always clean up in finally
ExecutorService pool = Executors.newFixedThreadPool(4);

for (int i = 0; i < 10; i++) {
    final int requestId = i;
    pool.submit(() -> {
        try {
            RequestContext.setUserId(12345L);
            // Process request...
        } finally {
            RequestContext.clear();  // ← MANDATORY cleanup
        }
    });
}
```

### InheritableThreadLocal — Parent → Child Thread Inheritance

```java
// ThreadLocal: child thread does NOT inherit parent's value
ThreadLocal<String> tl = new ThreadLocal<>();
tl.set("parent-value");

new Thread(() -> {
    String value = tl.get();  // Returns null — not inherited!
    System.out.println("Child sees: " + value);  // null
}).start();

// InheritableThreadLocal: child thread INHERITS parent's value
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
itl.set("parent-value");

new Thread(() -> {
    String value = itl.get();  // Returns "parent-value" — inherited!
    System.out.println("Child sees: " + value);  // parent-value
}).start();

// Use case: child threads spawned by parent should inherit request context
public class RequestContext {
    // Use InheritableThreadLocal so child threads (thread pool) inherit context
    private static final InheritableThreadLocal<Long> userIdHolder = new InheritableThreadLocal<>();
    
    public static void setUserId(Long userId) { userIdHolder.set(userId); }
    public static Long getUserId() { return userIdHolder.get(); }
}

// Parent thread (request thread) sets context
RequestContext.setUserId(123L);

// Child thread spawned by parent inherits it
Executors.newSingleThreadExecutor().submit(() -> {
    // Gets user 123 automatically from InheritableThreadLocal
    log.info("User: {}", RequestContext.getUserId());  // 123
});

// ⚠️ CAUTION: InheritableThreadLocal + thread pools can cause unintended inheritance
// If parent thread submits task with InheritableThreadLocal set, child threads inherit
// Then forgot to clear → context leaks to next request on that thread from pool
// Solution: manually copy context on submit, don't rely on inheritance in pools
```

### Real-World Pattern — Spring Logging with Correlation IDs

```java
// Request context holder
public class RequestContext {
    private static final ThreadLocal<String> correlationIdHolder = new ThreadLocal<>();
    private static final ThreadLocal<String> userHolder = new ThreadLocal<>();
    
    public static void setCorrelationId(String id) { correlationIdHolder.set(id); }
    public static String getCorrelationId() { return correlationIdHolder.get(); }
    
    public static void setUser(String user) { userHolder.set(user); }
    public static String getUser() { return userHolder.get(); }
    
    public static void clear() {
        correlationIdHolder.remove();
        userHolder.remove();
    }
}

// Filter to populate context from incoming request
@Component
public class CorrelationIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        HttpServletResponse httpResp = (HttpServletResponse) response;
        
        // Use existing correlation ID or generate new one
        String correlationId = httpReq.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        
        try {
            RequestContext.setCorrelationId(correlationId);
            RequestContext.setUser(httpReq.getUserPrincipal().getName());
            
            // Include correlation ID in response
            httpResp.setHeader("X-Correlation-ID", correlationId);
            chain.doFilter(request, response);
        } finally {
            RequestContext.clear();
        }
    }
}

// MDC (Mapped Diagnostic Context) in SLF4J uses ThreadLocal internally
@Service
public class OrderService {
    public void createOrder(CreateOrderRequest req) {
        // MDC stores values in ThreadLocal
        MDC.put("correlationId", RequestContext.getCorrelationId());
        MDC.put("userId", RequestContext.getUser());
        
        try {
            Order order = new Order();
            // All logging statements now include correlation ID automatically
            log.info("Creating order for items");  // [correlationId=abc-123] Creating order for items
            
            orderRepository.save(order);
            log.info("Order saved");  // [correlationId=abc-123] Order saved
        } finally {
            MDC.clear();  // Clean up MDC
        }
    }
}

// logback.xml configuration uses MDC values in pattern
// <pattern>%d{HH:mm:ss.SSS} [%X{correlationId}] [%thread] %-5level %msg%n</pattern>
// Output: 14:32:18.456 [abc-123] [pool-1-thread-1] INFO Creating order for items
```

### ThreadLocal Best Practices

```java
// ✅ Best Practice 1: Always declare ThreadLocal as static and final
public class BadContext {
    private ThreadLocal<String> context = new ThreadLocal<>();  // ❌ Instance field!
    // Creates new ThreadLocal for each instance — defeats purpose
}

public class GoodContext {
    private static final ThreadLocal<String> context = new ThreadLocal<>();  // ✅ Static + final
}

// ✅ Best Practice 2: Use try-finally for cleanup (or try-with-resources)
pool.submit(() -> {
    try {
        RequestContext.setRequest(req);
        processRequest();
    } finally {
        RequestContext.clear();  // ✅ Always clean up
    }
});

// ✅ Best Practice 3: Wrapper utility for automatic cleanup
public class ThreadLocalScope implements AutoCloseable {
    private final ThreadLocal<String> holder;
    private final String value;
    
    public ThreadLocalScope(ThreadLocal<String> holder, String value) {
        this.holder = holder;
        this.value = value;
        holder.set(value);
    }
    
    @Override
    public void close() {
        holder.remove();
    }
}

// Usage — auto-cleanup with try-with-resources
try (ThreadLocalScope scope = new ThreadLocalScope(RequestContext.contextHolder, "value")) {
    processRequest();  // Context available
}  // Auto-cleaned up

// ✅ Best Practice 4: Avoid ThreadLocal leaks — validate with WeakReference
public class ThreadLocalCache<T> {
    private final ThreadLocal<T> cache = ThreadLocal.withInitial(() -> null);
    
    public void set(T value) {
        cache.set(value);
    }
    
    public T get() {
        return cache.get();
    }
    
    public void clear() {
        cache.remove();  // ← Must be called
    }
}

// ❌ Memory leak detection — threads with non-null ThreadLocal values after request
Thread[] threads = new Thread[Thread.activeCount()];
Thread.enumerate(threads);
for (Thread t : threads) {
    if (t != null && RequestContext.getUserId() != null) {
        log.warn("Thread {} has request context set — possible leak", t.getName());
    }
}
```

### Interview Answer
> "ThreadLocal provides each thread isolated storage. Without it, you either pass context (user, transaction ID, correlation ID) as parameters through deep call stacks, or use static variables and risk thread interference. ThreadLocal solves this elegantly — each thread has its own namespace.
>
> Critical mistake: ThreadLocal in thread pools without cleanup causes security and memory bugs. A thread from the pool processes Request 1 (sets userId=123), then handles Request 2 but userId is still 123 from the previous request — you've leaked user context! Always clean up in finally blocks.
>
> In production I use ThreadLocal for request context in filters: set the correlation ID, user, request start time on entry, then clear in finally. All downstream services access context without parameters. SLF4J's MDC internally uses ThreadLocal for the same purpose — structured logging includes correlation IDs automatically.
>
> InheritableThreadLocal makes child threads inherit parent values, useful when spawning threads. But in pools it can cause unintended inheritance — prefer explicit context copying.
>
> Best practice: declare as static final, always clear in finally or try-with-resources, use wrapper utilities for automatic cleanup."
>
> *Likely follow-up: "What's the difference between ThreadLocal and InheritableThreadLocal?"*

---

## Real Project Usage — Topic 5 (Multithreading & Concurrency)

### Scenario 1: Deadlock in Account Transfer
> "Fund transfer service was deadlocked: Thread 1 locks Account A then B, Thread 2 locks B then A. Fix: always acquire locks in account ID order (consistent ordering). Added a comparator: `if (fromId > toId) { lock1=lockB; lock2=lockA; } else { lock1=lockA; lock2=lockB; }`. No more deadlocks in production."

### Scenario 2: Thread Pool Rejection in Bulk Processing
> "Bulk email service crashed with `RejectedExecutionException` when spiked. Using unbounded queue (wrong). Switched to bounded queue + `CallerRunsPolicy`: `new ThreadPoolExecutor(..., new LinkedBlockingQueue<>(5000), new ThreadPoolExecutor.CallerRunsPolicy())`. Spikes now back-pressure gradually instead of crashing."

### Scenario 3: ThreadLocal Leak in Servlet
> "Request context (userId, correlation ID) set in servlet filter was carried to the NEXT request in the connection pool. Attacker saw other users' data. Fix: always clear ThreadLocal in a try-finally in the filter. Also migrated to using `RequestContextHolder` from Spring (cleaner pattern)."

### Scenario 4: ForkJoinPool on I/O-Heavy Streams
> "Parallel streams were consistently slower than sequential for database queries. Expected: speedup with parallelism. Reality: ForkJoinPool had only 8 threads (CPU cores), all blocked on DB I/O. Millions of queries queued. Fix: kept queries sequential OR created custom ForkJoinPool with many threads for I/O (not default shared pool)."

---

## Quick Revision Card — Topic 5

| Question | Core Concept & Key Points |
|----------|---|
| **Q33/Q121** | Thread pool = reuse workers. `submit()` → `Future`. `shutdown()` + `awaitTermination()` for graceful stop. Sizing: CPU-bound = `nCores`, I/O-bound = `nCores × (1 + wait_time/compute_time)`. `Callable` returns value; `Runnable` returns void. |
| **Q122** | `ThreadPoolExecutor`: corePoolSize → queue → maxPoolSize → reject. Bounded queue is mandatory. `CallerRunsPolicy` = natural backpressure. Spring `@Async` via `AsyncConfigurer`. Monitor queue depth at 80% capacity threshold. |
| **Q34** | Deadlock: 4 Coffman conditions. Fix: consistent lock ordering (most reliable), `tryLock` with timeout, avoid nested locks, use high-level utilities. Detect with `jstack` or `ThreadMXBean.findDeadlockedThreads()`. |
| **Q35** | `volatile` = main memory visibility, NO atomicity. Use for flags. `counter++` still not atomic. Critical for double-checked locking Singleton (prevents JIT reordering of object construction). Happens-before: write to volatile happens-before subsequent read. |
| **Q124** | `synchronized` = simple, JIT-optimizable. `ReentrantLock` = explicit, timeout, fairness, Conditions, ReadWriteLock. Prefer: `AtomicInteger` for counters, `ConcurrentHashMap` for maps, `BlockingQueue` for producer-consumer, avoid manual locking. |
| **Q36/Q123** | Parallel streams win on CPU-bound, large datasets (100k+). Lose on I/O (blocks ForkJoinPool). Rules: stateless, thread-safe collectors, CPU-bound only. `RecursiveTask` for divide-and-conquer (threshold ~1000). Custom `ForkJoinPool` for I/O-heavy streams. |
| **Q37** | `supplyAsync` = async compute. `thenApply` = sync transform. `thenCompose` = flat-map (transform returns CompletableFuture). `allOf` = wait all. `anyOf` = first wins. Combine with `thenCombine`. Exception handle with `exceptionally` or `handle`. Custom `Executor` for I/O, never block on event loop. |
| **Q125** | `ThreadLocal` = per-thread isolated storage. Set/get context without parameters. CRITICAL: clean up in finally in thread pools (prevents leaks to next request). `InheritableThreadLocal` = child inherits parent value. Use for request context (user, correlation ID, transaction). Spring filters set context on entry, clear on exit. |

---

**End of Topic 5 — Multithreading, Concurrency & Async**
