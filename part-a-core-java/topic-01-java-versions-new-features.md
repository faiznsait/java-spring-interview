# PART A: Core Java — Topic 1: Java Versions & New Features

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer → Related Concepts

---

## Q1. Java 8, 11, 17, and 21 — Key Features by Release

### Concept
Each LTS (Long-Term Support) release of Java adds language features, performance improvements, and API modernisation to reduce boilerplate and improve developer productivity.

### Simple Explanation
Think of Java releases like iPhone generations — each adds features the previous couldn't do. Java 8 was the game-changer (like the original iPhone with third-party apps), Java 17 was the mature leap (like iPhone X with serious power), and Java 21 is where concurrency got revolutionised (Virtual Threads).

---

## Java 8 (2014) — The Revolution

### Problem
Before Java 8: Writing functional-style code required verbose anonymous inner classes.

```java
// Before Java 8
List<String> names = Arrays.asList("Bob", "Alice", "Charlie");
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

long count = 0;
for (String name : names) {
    if (name.startsWith("A")) {
        count++;
    }
}
```

### Solution: Java 8 Features

#### 1. Lambda Expressions
```java
// After Java 8
names.sort((a, b) -> a.compareTo(b));  // 1 line vs 5 lines

// Lambda syntax: (parameters) -> expression
Comparator<String> comp = (a, b) -> a.compareTo(b);
```

**When to use:** Functional interfaces with simple logic. For complex logic, use traditional classes.

#### 2. Stream API — Functional Data Processing
```java
// Sequential processing
long count = names.stream()
    .filter(n -> n.startsWith("A"))     // Intermediate (lazy)
    .map(String::toUpperCase)             // Intermediate (lazy)
    .count();                             // Terminal (executes pipeline)

// Collect to list
List<String> uppercase = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Find first match
Optional<String> first = names.stream()
    .filter(n -> n.length() > 5)
    .findFirst();

// Custom reduction
String concatenated = names.stream()
    .reduce("", (a, b) -> a + b);  // Slow for strings! Use Collectors.joining()
```

**Lazy evaluation:** Intermediate operations don't execute until a terminal operation is called.

```java
int counter = 0;
List<String> result = names.stream()
    .peek(n -> counter++)             // counter stays 0 — no terminal operation!
    .limit(2)
    .collect(Collectors.toList());    // Now peek runs (limit prevents full stream)
// counter = 2, not 3
```

#### 3. Optional — Null Safety
```java
// Before: NPE-prone
User user = userRepository.findById(id);
if (user != null && user.getAge() > 18) {
    System.out.println(user.getName());
}

// After: Expresses intent clearly
Optional<User> user = userRepository.findById(id);
user.filter(u -> u.getAge() > 18)
    .map(User::getName)
    .ifPresent(System.out::println);

// Common operations
Optional<User> user = Optional.ofNullable(maybeNull);
String name = user.map(User::getName).orElse("Unknown");
boolean present = user.isPresent();
user.ifPresent(u -> log.info("User: " + u.getName()));

// anti-pattern: calling .get() without check
String name = user.get();  // ❌ throws NoSuchElementException if empty
String name = user.orElseThrow(() -> new UserNotFound());  // ✅ explicit exception
```

#### 4. Default & Static Methods in Interfaces
```java
interface Greeter {
    String greet();  // abstract method
    
    // Default method — concrete behavior, overridable
    default void greetLoudly() {
        System.out.println(greet().toUpperCase());
    }
    
    // Static method — utility on interface, not inherited
    static Greeter english() {
        return () -> "Hello";
    }
}

class FriendlyGreeter implements Greeter {
    @Override
    public String greet() {
        return "Hi there!";
    }
}

// Usage
FriendlyGreeter fg = new FriendlyGreeter();
System.out.println(fg.greet());          // Hi there!
fg.greetLoudly();    // HI THERE!
System.out.println(Greeter.english().greet());  // HELLO

// Default method conflict resolution (diamond rule):
interface A { default String who() { return "A"; } }
interface B { default String who() { return "B"; } }
class C implements A, B {
    @Override
    public String who() { return "C->" + A.super.who(); }
}
// If a class and an interface both define the same method, class method wins.
```

#### 5. Functional Interfaces — The Foundation of Lambdas & Streams

**What is a Functional Interface?**
An interface with exactly **one** abstract method. Lambdas are shorthand for implementing them.

```java
// Any interface with 1 method is functional:
@FunctionalInterface
interface Comparator<T> {
    int compare(T a, T b);  // Single abstract method
    
    // Can have default/static methods:
    default Comparator<T> reversed() { ... }
}

// Before Java 8:
Collections.sort(names, new Comparator<String>() {
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// After Java 8 (lambda):
Collections.sort(names, (a, b) -> a.compareTo(b));
```

**The 4 Core Functional Interfaces (Used Everywhere in Streams):**

```java
// 1. Predicate<T> — test something, return boolean
// Used in: .filter()
Predicate<String> isLong = s -> s.length() > 5;
List<String> filtered = names.stream()
    .filter(isLong)  // Predicate<String>
    .collect(Collectors.toList());

// Compose predicates
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> both = isLong.and(startsWithA);
Predicate<String> either = isLong.or(startsWithA);
Predicate<String> negated = isLong.negate();

// 2. Function<T, R> — transform input T to output R
// Used in: .map()
Function<String, Integer> length = String::length;
List<Integer> lengths = names.stream()
    .map(length)  // Function<String, Integer>
    .collect(Collectors.toList());

// Chain transformations
Function<String, String> trim = String::trim;
Function<String, String> upper = String::toUpperCase;
Function<String, String> combined = trim.andThen(upper);
String result = combined.apply("  hello  ");  // "HELLO"

// 3. Consumer<T> — accept input, no return (side effects)
// Used in: .forEach()
Consumer<String> print = System.out::println;
Consumer<String> log = msg -> logger.info(msg);
Consumer<String> both = print.andThen(log);

names.forEach(both);  // Prints AND logs each name

// 4. Supplier<T> — no input, return T (factory/lazy)
// Used in: Optional.orElseGet()
Supplier<LocalDate> today = LocalDate::now;
Supplier<DefaultUser> defaultUser = () -> new DefaultUser();

String name = Optional.ofNullable(user)
    .map(User::getName)
    .orElseGet(() -> "Unknown");  // Supplier called only if empty

// 5. BiFunction<T, U, R> — two inputs, one output
BiFunction<String, String, String> concat = (a, b) -> a + " " + b;
String greeting = concat.apply("Hello", "World");  // "Hello World"
```

**Why Interview Cares:**
Every Stream operation uses functional interfaces under the hood. Understanding them deeply is the difference between knowing Streams and mastering them. Interviewers may ask:
- "What's the difference between Function and BiFunction?"
- "When would you use orElseGet vs orElse?" (Supplier vs eager evaluation)
- "Can you compose multiple predicates?" (and/or/negate)

---

#### 6. Method References — Shorthand for Lambda
```java
// Lambda
names.forEach(n -> System.out.println(n));

// Method reference (cleaner)
names.forEach(System::out::println);

// 4 types of method references:
// 1. Static: ClassName::staticMethod
Integer[] nums = {3, 1, 4, 1, 5};
Arrays.sort(nums, Integer::compare);

// 2. Instance: instance::instanceMethod
String str = "Hello";
Predicate<String> isEmpty = str::equals;

// 3. Specific type: Type::instanceMethod
names.stream().forEach(String::toUpperCase);  // calls method on each element

// 4. Constructor: ClassName::new
Stream<String> stream = Stream.of("a", "b");
List<String> list = stream.map(ArrayList::new).flatMap(l -> l.stream()).collect(Collectors.toList());
```

#### 8. New Date/Time API (java.time.*)
```java
// Before (mutable, confusing):
java.util.Date now = new Date();           // Current time
Calendar cal = Calendar.getInstance();
cal.add(Calendar.DATE, 7);
Date nextWeek = cal.getTime();

// After (immutable, intuitive):
LocalDate today = LocalDate.now();                       // Date only
LocalTime current = LocalTime.now();                     // Time only
LocalDateTime now = LocalDateTime.now();                 // Both
ZonedDateTime utc = ZonedDateTime.now(ZoneId.of("UTC")); // With timezone

LocalDate nextWeek = today.plusDays(7);     // Returns NEW instance
LocalDate birthday = LocalDate.of(1995, 5, 15);
LocalDate nextBirthday = birthday.plusYears(1);

LocalDateTime deadline = LocalDateTime.of(2026, 12, 31, 23, 59, 59);
Duration gap = Duration.between(LocalDateTime.now(), deadline);
long daysLeft = gap.toDays();
```

#### 7. CompletableFuture — Async Without Callback Hell
```java
// Before: Callback pyramid of doom
service.getUserAsync(id, user -> {
    service.getOrdersAsync(user, orders -> {
        service.processAsync(orders, result -> {
            callback.onSuccess(result);
        });
    });
});

// After: CompletableFuture composition
CompletableFuture.supplyAsync(() -> userService.getUser(id))
    .thenApply(user -> orderService.getOrders(user))  // Chain transformations
    .thenApply(orders -> processOrders(orders))
    .thenAccept(result -> callback.onSuccess(result))
    .exceptionally(ex -> {
        log.error("Error", ex);
        return null;
    });
```

**Critical: thenApply vs thenCompose (Common Interview Question)**

```java
// thenApply: When transformation is SYNC (returns a plain value)
CompletableFuture<User> user = fetchUserAsync(id);
CompletableFuture<String> name = user
    .thenApply(u -> u.getName());  // Transforms synchronously

// Result: CompletableFuture<String>

// thenCompose: When transformation RETURNS ANOTHER FUTURE (async-to-async)
CompletableFuture<Order> order = user
    .thenCompose(u -> fetchOrderAsync(u.getId()));  // Chains async calls, flattens result

// Result: CompletableFuture<Order> (NOT nested)

// WRONG: Avoid this!
CompletableFuture<CompletableFuture<Order>> nested = user
    .thenApply(u -> fetchOrderAsync(u.getId()));  // ❌ Creates nested Future
    // Later: nested.get().get()  ← ugly

// RIGHT: Use thenCompose
CompletableFuture<Order> flat = user
    .thenCompose(u -> fetchOrderAsync(u.getId()));  // ✓ Flattens automatically
    // Later: flat.get()  ← clean
```

**Production Pattern: Parallel Composition**
```java
// Sequential (slow):
CompletableFuture<User> user = fetchUserAsync(id);
CompletableFuture<Orders> orders = user
    .thenCompose(u -> fetchOrdersAsync(u));
// Wait for user -> then wait for orders

// Parallel (fast):
CompletableFuture<User> user = fetchUserAsync(id);
CompletableFuture<Orders> orders = fetchOrdersAsync(id);  // Start immediately
CompletableFuture<UserOrder> result = user
    .thenCombine(orders, (u, o) -> new UserOrder(u, o));
// Both run in parallel, wait for both

// Or for multiple:
CompletableFuture.allOf(user, orders, settings)
    .thenApply(v -> new Dashboard(user.get(), orders.get(), settings.get()))
    .thenAccept(dashboard -> renderUI(dashboard));
```
```

---

## Java 11 (2018) — Quality of Life Improvements

### Problem
Java 10 had var; Java 11 improved String utilities, gave modern HttpClient, better module system.

### Solutions

#### 1. New String Methods
```java
// isBlank() — checks if only whitespace (unlike isEmpty())
"  ".isEmpty();        // false — has whitespace
"  ".isBlank();        // true — empty after stripping whitespace
"\t\n".isBlank();      // true — Unicode whitespace included

// lines() — stream of lines
String multiline = "line1\nline2\nline3";
long lineCount = multiline.lines().count();  // 3
multiline.lines().forEach(System.out::println);

// strip() — Unicode-aware trim
"  hello  ".strip();          // "hello" — removes Unicode whitespace
"  hello  ".trim();           // "hello" — removes ASCII whitespace only
"  \u00A0  hello  \u00A0  ".strip();   // "hello" — handles non-breaking space

// repeat() — string multiplication
"ha".repeat(3);  // "hahaha"
```

#### 2. var Keyword — Local Variable Type Inference
```java
var names = List.of("Alice", "Bob");      // Infers List<String>
var cache = new ConcurrentHashMap<String, User>();
var factory = Executors.newFixedThreadPool(10);  // Better readability

// NOT for fields or parameters — only local variables
public void method(var param) {}  // ❌ Compile error

class Example {
    var field = "test";  // ❌ Compile error
    
    void test() {
        var local = "ok";  // ✅ Allowed
    }
}
```

#### 3. HTTP Client API — Modern Replacement for HttpURLConnection
```java
HttpClient client = HttpClient.newHttpClient();

// GET request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .GET()
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.statusCode());   // 200
System.out.println(response.body());         // JSON response

// POST request (async)
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"name\": \"Alice\"}"))
    .build();

client.sendAsync(postRequest, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

---

## Java 17 (2021) — Sealed Classes, Records, Pattern Matching

### Problem
- Writing immutable classes required 20+ lines of boilerplate
- Inheritance hierarchies could be extended unexpectedly
- Pattern matching required explicit casts

### Solutions

#### 1. Records — Immutable Data Carriers (Replaces Lombok @Value)

```java
// Before (20 lines):
public final class Point {
    private final int x;
    private final int y;
    
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int getX() { return x; }
    public int getY() { return y; }
    
    @Override
    public boolean equals(Object o) { ... }
    
    @Override
    public int hashCode() { ... }
    
    @Override
    public String toString() { ... }
}

// After (1 line):
record Point(int x, int y) {}

// Auto-generated:
Point p = new Point(3, 4);
System.out.println(p.x());       // Accessor: x() (not getX)
System.out.println(p);           // toString: Point[x=3, y=4]
System.out.println(p.equals(new Point(3, 4)));  // true

// Immutability guaranteed
// p.x = 5;  // ❌ Compile error — records are final
```

#### Record Validation (Compact Constructor)
```java
record User(String name, int age) {
    // Compact constructor — runs BEFORE field assignment
    public User {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age");
        }
    }
}

User u = new User("Alice", 30);   // ✓ Valid
User u2 = new User("", 25);       // ✗ throws IllegalArgumentException
```

#### 2. Sealed Classes — Restrict Inheritance
```java
// Without sealing — anyone can extend
abstract class PaymentMethod {}
class CreditCard extends PaymentMethod {}
class PayPal extends PaymentMethod {}
class Crypto extends PaymentMethod {}   // ❌ Unexpected!

// With sealing — only approved classes can extend
sealed interface PaymentMethod permits CreditCard, PayPal, ApplePay {}

record CreditCard(String number) implements PaymentMethod {}
record PayPal(String email) implements PaymentMethod {}
record ApplePay(String token) implements PaymentMethod {}
// class Crypto implements PaymentMethod {}  // ❌ Compile error

// Pattern matching on sealed hierarchy is exhaustive:
String process(PaymentMethod payment) {
    return switch (payment) {
        case CreditCard cc -> "Charging card: " + cc.number();
        case PayPal pp -> "Processing PayPal: " + pp.email();
        case ApplePay ap -> "Apple Pay: " + ap.token();
        // No default needed — compiler verifies all cases ✓
    };
}

// If you add new PaymentMethod subclass, this switch FAILS at compile time
// Forces you to handle the new case (safer than runtime surprise)
```

#### 3. Pattern Matching for instanceof
```java
// Before Java 17:
if (obj instanceof String) {
    String s = (String) obj;              // Explicit cast
    System.out.println(s.length());
}

// Java 17+:
if (obj instanceof String s) {            // Variable binding
    System.out.println(s.length());       // s already cast
}

// With guards (logical conditions):
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}

// Nested patterns (preview):
record Point(int x, int y) {}
if (shape instanceof Point(int x, int y) && x > 0 && y > 0) {
    System.out.println("First quadrant: " + x + ", " + y);
}
```

#### 4. Text Blocks — Multi-Line Strings Without Escaping
```java
// Before:
String json = "{\"name\": \"Alice\", \"age\": 30}";

// After:
String json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;

// Automatic indentation handling
String sql = """
    SELECT id, name, email
    FROM users
    WHERE age > 18
    ORDER BY name
    """;

// Useful for HTML, JSON, SQL, XML in code
String html = """
    <html>
        <body>
            <h1>Hello</h1>
        </body>
    </html>
    """;
```

#### 5. Switch Expressions (Standard in 17, Preview in 14)
```java
// Before (statement):
int daysInMonth;
switch (month) {
    case "JAN":
    case "MAR":
        daysInMonth = 31;
        break;      // ← Easy to forget, causes fall-through bugs
    case "FEB":
        daysInMonth = 28;
        break;
    // ...
}

// After (expression):
int daysInMonth = switch (month) {
    case "JAN", "MAR", "MAY" -> 31;        // No fall-through, no break
    case "FEB" -> 28;
    case "APR", "JUN", "SEP" -> 30;
    default -> throw new IllegalArgumentException("Invalid: " + month);
};

// Can return values
String season = switch (month) {
    case "DEC", "JAN", "FEB" -> "Winter";
    case "MAR", "APR", "MAY" -> "Spring";
    case "JUN", "JUL", "AUG" -> "Summer";
    case "SEP", "OCT", "NOV" -> "Fall";
    default -> "Unknown";
};
```

---

## Java 21 (2023) — Virtual Threads + Structured Concurrency

### Problem
Platform threads are expensive:
- 1MB stack per thread
- Each thread = 1 OS thread (context switching overhead)
- Max ~2000 threads per server
- 1M concurrent requests = impossible with platform threads

### Solution: Virtual Threads

#### How Virtual Threads Work
```java
// Virtual threads unmount from "carrier" threads during blocking I/O

┌─────────────────────────────┐
│ JVM Scheduler               │
│ (10 carrier/platform threads) │
└────────┬────────────────────┘
         │
    ┌────┴─────┐
    │           │
  [C1]        [C2]             (Carrier threads)
   │            │
   ├─ VT1      ├─ VT3
   ├─ VT2      ├─ VT4
   └─ VT5      └─ VT6          (Virtual threads)

When VT1 blocks on I/O:
  VT1 unmounts from C1
  C1 picks up next available virtual thread
  When VT1's I/O completes, it remounts on a carrier thread

Result: 10 carriers × 1000 VT each = 10K concurrent with same resource usage
```

#### Virtual Threads in Code
```java
// Before (limited concurrency):
ExecutorService pool = Executors.newFixedThreadPool(200);
for (Request req : requests) {
    pool.submit(() -> {
        String response = expensiveAPI.call();   // Blocks 100ms
                                                  // Thread sits idle
        database.save(response);
    });
}
// Max ~200 concurrent requests, uses 200MB RAM

// After (unlimited concurrency):
ExecutorService pool = Executors.newVirtualThreadPerTaskExecutor();
for (Request req : requests) {
    pool.submit(() -> {
        String response = expensiveAPI.call();   // Unmounts during I/O
                                                  // Carrier thread handles others
        database.save(response);                  // Remounts when data ready
    });
}
// 100K+ concurrent requests, uses minimal RAM (virtual threads are cheap)
```

#### Key Properties
```java
// Creating virtual threads
Thread vt = Thread.ofVirtual()
    .name("worker-%d")
    .start(() -> System.out.println("Running on virtual thread"));

// Executor approach (preferred)
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
exec.submit(() -> System.out.println("Task on virtual thread"));

// Virtual threads can't use synchronized (would pin)
// They must unmount during blocking for scalability
// Synchronized blocks prevent unmounting — avoid!
```

#### Virtual Threads vs Reactive Programming
```java
// Virtual Threads: Imperative, familiar style
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
exec.submit(() -> {
    User user = userService.getUser(id);         // Blocking, but unmounts
    Orders orders = orderService.getOrders(user); // Blocking, but unmounts
    sendEmail(user, orders);                      // Blocking, but unmounts
    
    // Looks synchronous but actually async (unmounts during blocking)
});

// Reactive: Functional style, harder to learn
userService.getUserReactive(id)
    .flatMap(user -> orderService.getOrdersReactive(user))
    .flatMap(orders -> sendEmailReactive(orders))
    .subscribe(
        result -> log.info("Done"),
        error -> log.error("Failed", error)
    );

// Virtual threads win for existing synchronous codebases
// Reactive still needed for CPU-bound pipelines
```

---

## Real Project Usage

### E-Commerce Payment Service (Scaled from 100 to 10K Concurrent)

**Before (Java 11, Thread Pool 200):**
```
Peak load capacity: 200 concurrent requests
P99 latency: 5 seconds (payment gateway latency + queue wait)
When queue full: 429 Too Many Requests
```

**After (Java 21, Virtual Threads):**
```java
@Service
public class PaymentService {
    private final ExecutorService executor = 
        Executors.newVirtualThreadPerTaskExecutor();
    
    public void processPaymentAsync(PaymentRequest req) {
        executor.submit(() -> {
            try {
                Payment result = paymentGateway.charge(req);  // Unmounts during I/O
                database.save(result);                        // Unmounts during I/O
                webhook.notifyMerchant(result);               // Unmounts during I/O
                cache.put(req.getId(), result);
                
                // Total execution: linear, but unmounts during all blocking
            } catch (Exception ex) {
                handleFailure(req, ex);
            }
        });
    }
}
```

**Outcome:**
- Peak capacity: 10,000 concurrent requests
- P99 latency: 50ms (no queue wait)
- Same 8GB server


### Streaming Service (Analytics)

**Virtual threads for event processing:**
```java
ExecutorService processor = Executors.newVirtualThreadPerTaskExecutor();

// Process 100K events/sec
kafkaConsumer.subscribe(List.of("events"));
while (true) {
    ConsumerRecords<String, Event> records = kafkaConsumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Event> record : records) {
        processor.submit(() -> {
            analytics.process(record.value());     // Heavy computation unmounts on DB I/O
            database.save(analytics.result());     // Unmounts during persist
            cache.increment(key);                  // Unmounts during cache network
        });
    }
}

// 100K virtual threads created
// Only ~50 platform threads needed
// All 50 stay busy with minimal context switching
```

---

## Interview Answer (2-3 minutes)

> "Java 8 was the revolution — lambdas and Streams finally brought functional programming to Java without boilerplate. I use Streams heavily: filter, map, flatMap for data pipelines. Optional forces null-safety; I use it instead of null checks.
>
> Java 17 gave us Records for DTOs — one line instead of 20 with auto-generated equals/hashCode/toString. Sealed classes let me create closed type hierarchies with exhaustive pattern matching; the compiler forces me to handle all cases in a switch, which catches bugs early.
>
> Java 21's Virtual Threads are production game-changers. They unmount during I/O blocking, so I can handle 10K+ concurrent requests with the same server. I use `Executors.newVirtualThreadPerTaskExecutor()` for all I/O-bound workloads. The code looks synchronous but behaves asynchronously — no callback hell, easier to debug.
>
> The key insight: older Java versions forced a choice between scalability (reactive, callbacks) and simplicity (blocking, readable). Virtual Threads unify both.
>
> *Likely follow-up: 'Can Virtual Threads replace reactive frameworks like Reactor, WebFlux?' → Answer: For I/O-bound services yes. For CPU-bound or Flux-style backpressure, reactive still needed.*"

---

## Key Gotchas & Common Mistakes

### ❌ Gotcha 1: Optional Anti-patterns
```java
// WRONG — defeats the purpose of Optional
if (user.isPresent()) {
    return user.get().getName();
}
return "Unknown";

// RIGHT — use map + orElse
return user.map(User::getName).orElse("Unknown");

// WRONG — calling get() without check
String name = user.get();  // ← throws NoSuchElementException

// RIGHT — explicit error handling
String name = user.orElseThrow(() -> new UserNotFoundException(id));
```

### ❌ Gotcha 2: Integer Caching Behavior
```java
// This surprises many developers!
Integer a = 127;
Integer b = 127;
System.out.println(a == b);     // true — cached by JVM

Integer a = 128;
Integer b = 128;
System.out.println(a == b);     // false — NOT cached (beyond -128 to 127)

// Always use .equals() for object comparison:
Integer a = 128;
Integer b = 128;
System.out.println(a.equals(b));  // true — correct behavior
```

### ❌ Gotcha 3: Lambda + Effectively Final (Production Mistake)

**The Issue:**
Variables captured in lambdas must be **effectively final** — can't change after initialization. This confuses developers who try to mutate captured state.

```java
int count = 0;
List<String> names = List.of("Alice", "Bob");
names.forEach(n -> {
    count++;  // ❌ Compile error — count is not effectively final
            // Lambda captures VALUE, not reference
});

// count is NOT effectively final because it can change
count = 5;  // ← Changes the local variable

// The fix is NOT to mutate:
int count = 0;
for (String n : names) {
    count++;  // ✓ Imperative style (no lambda)
}
```

**Why This Limitation?**
Java lambdas capture **values** (like closure in functional languages), not mutable references. Once compiled, the lambda doesn't have access to change the original variable.

```java
// What the compiler does internally:
int count = 0;
Consumer<String> consumer = (s) -> {
    System.out.println(count);  // Captures a COPY of count's value
};
count = 5;  // Changes original, but lambda still sees 0
```

**Production Workunarounds (When You Must Mutate Captured State):**

```java
// 1. Use AtomicInteger (thread-safe mutable wrapper)
AtomicInteger count = new AtomicInteger(0);
names.forEach(n -> count.incrementAndGet());  // ✓ Works
System.out.println(count.get());

// 2. Use array (mutable container, hacky but works)
int[] count = {0};
names.forEach(n -> count[0]++);
System.out.println(count[0]);

// 3. Collect into a data structure
List<Integer> counts = names.stream()
    .map(n -> 1)  // Each name → count of 1
    .collect(Collectors.toList());
int total = counts.stream().mapToInt(Integer::intValue).sum();

// 4. BEST: Use functional approach (no mutation)
int total = (int) names.stream()
    .filter(n -> n.length() > 3)
    .count();  // Functional, no mutation needed
```

**Interview Context:**
Interviewers test this when you write stream/lambda code in a live challenge. They watch for:
- Trying to mutate a local variable captured by lambda
- Not understanding closure/value capture
- Defaulting to mutation instead of functional approach

**Production Impact:**
This limitation actually encourages better code:
- Functional style (no side effects)
- Easier to parallelize streams (no race conditions on shared state)
- Clearer semantics (lambda can't surprise with mutations)

### ❌ Gotcha 4: Pattern Matching Exhaustiveness
```java
sealed interface Result permits Success, Failure {}

String message = switch (result) {
    case Success s -> "OK: " + s.value();
    case Failure f -> "Error: " + f.error();
    // If you add new Result subclass, this doesn't compile ✓
};

// Without sealed, the compiler can't enforce exhaustiveness
// BAD: unknown case T appears at runtime
```

### ❌ Gotcha 5: Virtual Thread Pinning
```java
// Virtual thread can't unmount while in synchronized block
synchronized (lock) {
    String response = api.call();  // ❌ Blocks carrier thread (pinning)
                                   // No other virtual threads run
}

// Solution: use ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    String response = api.call();  // ✓ Can unmount if needed
} finally {
    lock.unlock();
}
```

---

## Related Concepts & Advanced Topics

### Sealed Classes + Records = Type-Safe Algebraic Data Types
```java
sealed interface Command permits InsertCmd, UpdateCmd, DeleteCmd {}

record InsertCmd(String table, Map<String, Object> data) implements Command {}
record UpdateCmd(String table, String id, Map<String, Object> data) implements Command {}
record DeleteCmd(String table, String id) implements Command {}

// Exhaustive pattern matching
Result execute(Command cmd) {
    return switch (cmd) {
        case InsertCmd i -> database.insert(i.table(), i.data());
        case UpdateCmd u -> database.update(u.table(), u.id(), u.data());
        case DeleteCmd d -> database.delete(d.table(), d.id());
    };
}
```

### Streams + Records = Clean Data Transformation
```java
List<Map<String, String>> csv = parseCsv("users.csv");
List<User> users = csv.stream()
    .map(row -> new User(row.get("id"), row.get("name"), Integer.parseInt(row.get("age"))))
    .filter(u -> u.age() > 18)
    .collect(Collectors.toList());

// Or directly with records:
record User(String id, String name, int age) {}
List<User> users = csv.stream()
    .map(row -> new User(row.get("id"), row.get("name"), Integer.parseInt(row.get("age"))))
    .filter(u -> u.age() > 18)
    .collect(Collectors.toList());
```

### CompletableFuture + Virtual Threads for Async
```java
// Virtual threads simplify async code
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();

// Instead of reactive chains, use imperative + virtual threads
CompletableFuture<Result> future = CompletableFuture.supplyAsync(() -> {
    User user = userService.getUser(id);        // Blocking but unmounts
    List<Order> orders = orderService.getOrders(user);
    return new Result(user, orders);
}, exec);

future.exceptionally(ex -> {
    log.error("Error", ex);
    return Result.empty();
});
```

### Interview Answer

> "Java 8 was transformational — lambdas, streams, Optional, and CompletableFuture changed how we write Java fundamentally. Java 11 added quality-of-life APIs — better String methods, a modern HttpClient. Java 17 was the next LTS leap — records replaced boilerplate POJOs, sealed classes enabled exhaustive type hierarchies, pattern matching removed ugly casts. Java 21 brings virtual threads — you write blocking code that scales like reactive code without callback hell.
>
> In my project I used Java 17 records for DTOs and pattern matching for instanceof in a service dispatcher."
>
> *Likely follow-up: "What is the difference between a Record and a class with Lombok's @Value?"*

---

## Q2. Switch Expressions — Beyond Traditional switch

### Concept
**Switch Expressions** (Java 14+) provide pattern-based control flow with exhaustiveness checking.
Unlike switch statements, they return values and require all cases handled.

### Simple Explanation
Traditional switch: A cop directing traffic — points you in ONE direction, might fall through unlisted holes. Switch expressions: A vending machine — press button, get item, guaranteed outcome (no falling through).

### Code — Traditional vs Expression

```java
// ❌ TRADITIONAL SWITCH (Java < 14)
int days = switch (month) {
    case "January":
    case "March":
    case "May":
    case "July":
    case "August":
    case "December":
        yield 31;
    case "April":
    case "June":
    case "September":
    case "November":
        yield 30;
    case "February":
        yield 28;
    default:
        yield -1;
};

// ✓ SWITCH EXPRESSION (Java 14+)
int days = switch (month) {
    case "January", "March", "May", "July", "August", "December" -> 31;
    case "April", "June", "September", "November" -> 30;
    case "February" -> 28;
    default -> -1;
};

// ✓ Arrow syntax (->): No fall-through, returns value directly

// Exhaustiveness checking
String type = switch (day) {  // Must handle ALL cases or have default
    case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday" -> "Weekday";
    case "Saturday", "Sunday" -> "Weekend";
    // ✓ All values covered, no default needed
};

// Multiple statements with yield
String size = switch (length) {
    case 0:
        System.out.println("Empty");
        yield "EMPTY";
    case 1, 2, 3:
        System.out.println("Small");
        yield "SMALL";
    default:
        yield "LARGE";
};
```

### Pattern Matching in switch (Java 16+)

```java
// ✓ Type pattern matching
Object obj = "hello";
String result = switch (obj) {
    case String s -> "String: " + s.length();
    case Integer i -> "Integer: " + (i * 2);
    case Double d -> "Double: " + (d + 1.0);
    default -> "Unknown";
};

// ✓ Guarded patterns (Java 17+)
Object value = 42;
String category = switch (value) {
    case Integer i when i < 0 -> "Negative";
    case Integer i when i == 0 -> "Zero";
    case Integer i when i > 0 -> "Positive";
    default -> "Not a number";
};

// ✓ Record pattern matching (Java 19+)
record Point(int x, int y) {}
Point p = new Point(3, 4);

String direction = switch (p) {
    case Point(int x, int y) when x > 0 && y > 0 -> "Northeast";
    case Point(int x, int y) when x < 0 && y > 0 -> "Northwest";
    case Point(int x, int y) when x > 0 && y < 0 -> "Southeast";
    case Point(int x, int y) when x < 0 && y < 0 -> "Southwest";
    case Point(0, _) -> "On Y-axis";
    case Point(_, 0) -> "On X-axis";
    case Point(0, 0) -> "Origin";
};

// Record destructuring in switch:
// Point(int x, int y) — binds fields to variables x, y
// Point(int x, _) — ignores y (underscore wildcard)
```

### Interview Answer

> "Switch expressions were introduced in Java 14 and finalized in Java 17. The key difference from traditional switch statements: expressions return a value, and the compiler enforces exhaustiveness — you must handle ALL cases or provide a default. So zero chance of accidental fallthrough bugs. I use them constantly in model mappers and API response builders — way cleaner than if-else chains and the compiler validates I didn't miss a case."
>
> *Likely follow-up: "What happens if you don't provide a default case in a switch expression?"*

---

## Q3. Text Blocks — Multi-line String Literals (Java 15+)

### Concept
**Text Blocks** provide clean multi-line string literals without escaping newlines.
Useful for JSON, SQL, HTML, XML embedded in code.

### Simple Explanation
Before: Writing multi-line strings required concatenation or escape sequences, messy. Text blocks: write naturally across lines, Java handles formatting.

### Code — Before vs After

```java
// ❌ OLD WAY (concatenation messy)
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30\n" +
"}";

String sql = "SELECT * FROM users\n" +
    "WHERE age > 18\n" +
    "ORDER BY name";

// ✓ TEXT BLOCK (Java 15+)
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;

String sql = """
    SELECT * FROM users
    WHERE age > 18
    ORDER BY name
    """;

String html = """
    <!DOCTYPE html>
    <html>
      <body>
        <h1>Hello, World!</h1>
      </body>
    </html>
    """;
```

### Key Rules

```java
// 1. Opening """ must be on same line as string variable
String valid = """
    content
    """;  // ✓ Valid

String invalid = """
content
    """;  // ❌ Will include leading newline!

// 2. Closing """ determines indentation (dedents all lines)
String text = """
        Line 1
        Line 2
    """;
// Lines dedented to match closing """

// 3. Escape sequences still work
String withTab = """
    Name\tValue
    John\t30
    """;

String withNewline = """
    Line 1\
    
    Line 3
    """;  // Backslash escapes newline

// 4. Incidental whitespace stripped automatically
String sql = """
    SELECT * FROM users
    WHERE age > 18
    ORDER BY name
    """;
// Leading spaces stripped smartly (matches column)

// ✓ Perfect for embedded SQL/JSON in tests
@Test
void testJsonParsing() {
    String json = """
        {
            "id": 1,
            "name": "Alice"
        }
        """;
    
    JsonNode node = objectMapper.readTree(json);
    assertEquals("Alice", node.get("name").asText());
}
```

### Interview Answer

> "Text blocks eliminate the escaping hell of multi-line strings. Before: concatenating dozens of strings with `+` and escaping quotes/newlines. Now: paste the JSON/SQL directly between triple quotes. The compiler handles indentation smartly — it removes the common leading whitespace so your code isn't forced into one column. I use them for SQL queries in @Query annotations and JSON test fixtures."
>
> *Likely follow-up: "What is the difference between a text block and a regular string?"*

---

## Q4. Records — Immutable Data Carriers (Java 16+)

### Concept
**Records** are concise way to define immutable carrier classes for data.
Automatically generates constructor, getters, equals, hashCode, toString.

### Simple Explanation
Record: "Tell me the shape of your data, I'll handle the boilerplate."
Without: Write class, fields, constructor, getters, equals, hashCode, toString (50+ lines).
With: Write record declaration (1 line).

### Code — Class vs Record

```java
// ❌ TRADITIONAL CLASS (boilerplate heavy)
public class Person {
    private final String name;
    private final int age;
    
    public Person(String name, int age) {  // Constructor
        this.name = name;
        this.age = age;
    }
    
    public String name() {  // Getters
        return name;
    }
    
    public int age() {
        return age;
    }
    
    @Override
    public boolean equals(Object o) {  // equals
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }
    
    @Override
    public int hashCode() {  // hashCode
        return Objects.hash(name, age);
    }
    
    @Override
    public String toString() {  // toString
        return "Person{" + "name='" + name + '\'' + ", age=" + age + '}';
    }
}

// ✓ RECORD (one line!)
public record Person(String name, int age) {}

// That's it! Compiler auto-generates all the above.
Person p = new Person("Alice", 30);
System.out.println(p.name());       // "Alice"
System.out.println(p.age());        // 30
System.out.println(p);              // Person[name=Alice, age=30]
System.out.println(p.equals(new Person("Alice", 30)));  // true
```

### Record Features

```java
// Components: name, age are "components" (not fields)
public record User(String name, int age, List<String> roles) {}

// Auto-generated methods:
// Constructor: User(String name, int age, List<String> roles)
// Getters: name(), age(), roles()  [NO get prefix!]
// equals(): Compares all components
// hashCode(): Based on all components
// toString(): User[name=..., age=..., roles=...]

// Components are IMMUTABLE
User u = new User("John", 30, new ArrayList<>());
// u.name() = "John" — read-only
// NO u.setName() method

// It's sealed! Can't inherit from record
public class Employee extends User {}  // ❌ Compile error: Can't extend record

// Components ARE public final fields (special)
// Not as accessible as class fields, but @Getter-like
public record Point(int x, int y) {
    // Canonical constructor is implicit: Point(int x, int y) { this.x = x; this.y = y; }
}

// Custom constructor allowed (compact form)
public record Point(int x, int y) {
    public Point {  // Compact constructor (Java 14+)
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("Coordinates must be positive");
        }
    }
    
    // Long form (less common):
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

Point p = new Point(0, 0);  // ✗ IllegalArgumentException

// Can add static methods
public record Color(int r, int g, int b) {
    public static Color RED = new Color(255, 0, 0);
    public static Color GREEN = new Color(0, 255, 0);
    public static Color BLACK = new Color(0, 0, 0);
}

// Add instance methods
public record Currency(String code) {
    public boolean isUSD() {
        return "USD".equals(code);
    }
}
```

### Records + Pattern Matching

```java
// Records shine with pattern matching
record Message(String sender, String text) {}
record Command(String action) {}

Object msg = new Message("Alice", "Hello Bob");

String result = switch (msg) {
    case Message(String sender, "quit") -> "Goodbye, " + sender;
    case Message(String sender, String text) -> "From " + sender + ": " + text;
    case Command(String action) -> "Action: " + action;
    default -> "Unknown";
};
// OUTPUT: "From Alice: Hello Bob"

// Destructuring with record patterns (Java 19+)
record Point(int x, int y) {}
record Circle(Point center, int radius) {}

Circle circle = new Circle(new Point(0, 0), 5);
String location = switch (circle) {
    case Circle(Point(0, 0), int r) -> "Circle at origin, radius " + r;
    case Circle(Point(int x, int y), int r) when x > 0 && y > 0 -> "Circle in Q1";
    case Circle(Point(_, _), int r) -> "Circle elsewhere, radius " + r;
};
```

### Records in Real Projects

```java
// DTOs become one-liners
public record UserDTO(String id, String email, String name) {}

// API Response wrappers
public record ApiResponse<T>(boolean success, T data, String message) {}

// Spring Boot REST Controller
@RestController
@RequestMapping("/users")
public class UserController {
    
    @PostMapping
    public ApiResponse<UserDTO> createUser(@RequestBody UserDTO request) {
        User user = userService.create(request);
        return new ApiResponse<>(true, new UserDTO(user.getId(), user.getEmail(), user.getName()), "User created");
    }
    
    @GetMapping("/{id}")
    public ApiResponse<UserDTO> getUser(@PathVariable String id) {
        User user = userService.get(id);
        return new ApiResponse<>(true, new UserDTO(user.getId(), user.getEmail(), user.getName()), null);
    }
}

// Event sourcing with records
record UserCreatedEvent(String id, String email, LocalDateTime timestamp) {}
record UserDeletedEvent(String id, LocalDateTime timestamp) {}

sealed interface Event permits UserCreatedEvent, UserDeletedEvent {}
```

### Interview Answer

> "Records are Java 16's syntax for immutable data carriers. You declare the fields in the header and Java auto-generates the canonical constructor, accessors, equals, hashCode, and toString. They're the native alternative to Lombok @Value. Sealed classes restrict which types can implement an interface — the compiler knows the complete set. The synergy is powerful: use sealed + records together, then use exhaustive switch expressions — the compiler guarantees you handled every case. No more forgotten default clauses hiding bugs."
>
> *Likely follow-up: "Can a record implement a sealed interface?"*

---

## Q5. Functional Interfaces — Problem Solved + Deep Dive

### Concept
Before Java 8, passing behaviour required full anonymous class syntax. Functional interfaces standardised single-method contracts, making lambdas possible.

### Predicate — test, compose
```java
Predicate<Integer> isEven     = n -> n % 2 == 0;
Predicate<Integer> isPositive = n -> n > 0;

// Compose
Predicate<Integer> isEvenAndPositive = isEven.and(isPositive);
Predicate<Integer> eitherOr          = isEven.or(isPositive);
Predicate<Integer> isOdd             = isEven.negate();

List<Integer> nums = List.of(-4, -1, 0, 3, 6, 8);
nums.stream().filter(isEvenAndPositive).toList(); // [6, 8]
```

### Function — transform, chain
```java
Function<Employee, String> getName  = Employee::getName;
Function<String, String>   toUpper  = String::toUpperCase;

// andThen: left-to-right pipeline
Function<Employee, String> getUpperName = getName.andThen(toUpper);
getUpperName.apply(emp); // "ALICE"

// compose: right-to-left — f.compose(g) = f(g(x))
Function<Integer, Integer> times2 = x -> x * 2;
Function<Integer, Integer> plus3  = x -> x + 3;
times2.compose(plus3).apply(5); // times2(plus3(5)) = times2(8) = 16
times2.andThen(plus3).apply(5); // plus3(times2(5)) = plus3(10) = 13
```

### Consumer — side effects, chain
```java
Consumer<String> writeToDb    = this::saveToDatabase;
Consumer<String> writeToLog   = logger::info;
Consumer<String> writeToAudit = auditService::log;

// andThen chains: all three consumers run for each element
Consumer<String> allThree = writeToDb.andThen(writeToLog).andThen(writeToAudit);
userNames.forEach(allThree);
```

### Supplier — lazy factory, defer creation
```java
// orElseGet() vs orElse() — the most commonly asked distinction
String result  = Optional.ofNullable(value)
    .orElse(expensiveMethod());       // BAD: expensiveMethod() ALWAYS called

String result2 = Optional.ofNullable(value)
    .orElseGet(() -> expensiveMethod()); // GOOD: only called if Optional is empty
```

### Interview Answer
> "Before Java 8, passing behaviour required anonymous class boilerplate. Functional interfaces standardise the contract: one abstract method = one piece of behaviour. The key practical differences: Predicate composes with and/or/negate, Function chains with andThen/compose, Consumer chains side effects with andThen, and Supplier is about laziness — specially useful with orElseGet() because unlike orElse(), the Supplier only executes when the value is actually needed."
>
> *Likely follow-up: "What is the difference between andThen and compose in Function?"*

### Related Concepts & Common Follow-ups
- **`andThen` vs `compose`:** `f.andThen(g)` = `g(f(x))` (left-to-right). `f.compose(g)` = `f(g(x))` (right-to-left). Think of `andThen` like a pipeline step.
- **`Stream.of()` vs `Arrays.stream()`:** `Arrays.stream(int[])` returns an `IntStream` (primitive stream). `Stream.of(int[])` returns `Stream<int[]>` (one element — the array itself). Common gotcha.

---

## Q88. Date-Time API (`java.time`) — Why It Replaced `Date` and `Calendar`

### Concept
`java.time` (introduced in Java 8) provides an immutable, thread-safe, and intuitive date-time API to replace the broken `java.util.Date` and `Calendar`.

### Simple Explanation
Old `Date` was like a whiteboard in a shared room — anyone could erase it and write different values (mutable, not thread-safe). `java.time` classes are like printed receipts — once printed, they can never be changed. Any operation produces a new receipt.

### What Was Wrong With Old API
```java
// java.util.Date problems:
Date date = new Date();
date.setTime(0); // Mutable! Any method can change it — breaks contracts
// Thread-unsafe: two threads modifying same Date = corruption
// Confusing: months 0-indexed (January = 0), year = actual year - 1900
// SimpleDateFormat is not thread-safe

// Calendar even worse:
Calendar cal = Calendar.getInstance();
cal.set(2024, 0, 15); // January = 0 — confusing!
```

### `java.time` Key Classes
```java
// LocalDate — date only (no time, no timezone)
LocalDate today     = LocalDate.now();
LocalDate birthday  = LocalDate.of(1990, Month.MAY, 15);
LocalDate nextWeek  = today.plusDays(7);        // Immutable — returns new object
LocalDate lastMonth = today.minusMonths(1);

today.getDayOfWeek();   // WEDNESDAY
today.isLeapYear();     // false
today.isAfter(birthday); // true

// LocalTime — time only (no date, no timezone)
LocalTime now      = LocalTime.now();
LocalTime meeting  = LocalTime.of(14, 30);       // 2:30 PM
LocalTime later    = meeting.plusHours(2);        // 4:30 PM

// LocalDateTime — date + time (no timezone)
LocalDateTime dt = LocalDateTime.of(today, meeting);
LocalDateTime dt2 = LocalDateTime.now();

// ZonedDateTime — date + time + timezone (use for API responses, DB storage)
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("America/New_York"));
ZonedDateTime utc = zdt.withZoneSameInstant(ZoneId.of("UTC"));

// Instant — machine timestamp (nanoseconds since epoch) — best for DB/logging
Instant now2 = Instant.now();
long epochMs = now2.toEpochMilli();

// Duration — time-based amount (hours, minutes, seconds)
Duration duration = Duration.between(LocalTime.of(9, 0), LocalTime.of(17, 0));
duration.toHours(); // 8

// Period — date-based amount (years, months, days)
Period age = Period.between(birthday, today);
age.getYears(); // person's age in years
```

### Date/Time Formatting
```java
// DateTimeFormatter is thread-safe (unlike SimpleDateFormat!)
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String formatted = today.format(fmt);              // "15/03/2024"
LocalDate parsed = LocalDate.parse("15/03/2024", fmt); // back to LocalDate

// ISO formats built-in
today.format(DateTimeFormatter.ISO_DATE); // "2024-03-15"
```

### Converting Legacy ↔ `java.time`
```java
// Date → Instant → LocalDateTime
Date oldDate = new Date();
Instant instant = oldDate.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());

// LocalDateTime → Date (for legacy APIs)
Date newDate = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```

### Real Project Usage
> "I use `Instant` for all DB timestamps and API response timestamps — it stores epoch millis, avoids timezone ambiguity. `LocalDate` for business dates like subscription start/end. `ZonedDateTime` for user-facing events where timezone matters. `DateTimeFormatter` in a static constant because it's thread-safe, unlike `SimpleDateFormat` which we had to put in a `ThreadLocal` with the old API."

### Interview Answer
> "`java.util.Date` was mutable and thread-unsafe — any thread could change it, it had confusing 0-indexed months, and `SimpleDateFormat` required ThreadLocal to be safe. `java.time` fixed all of this: all classes are immutable (every operation returns a new instance), thread-safe, and the API is intuitive. The key classes: `LocalDate` for date-only, `LocalTime` for time-only, `LocalDateTime` for both without timezone, `ZonedDateTime` for timezone-aware, and `Instant` for machine timestamps. I use `Instant` for persistence."
>
> *Likely follow-up: "What is the difference between Period and Duration?"*

### Related Concepts & Common Follow-ups
- **Period vs Duration:** `Period` = date-based (years/months/days). `Duration` = time-based (hours/minutes/seconds/nanos). `Period.between(date1, date2)` vs `Duration.between(time1, time2)`.
- **Why is `SimpleDateFormat` not thread-safe?** It holds internal mutable state during parsing. Two threads formatting simultaneously corrupt each other's results. Fix: either create a new instance per call, use `ThreadLocal<SimpleDateFormat>`, or switch to `DateTimeFormatter` (which is immutable).
- **Storing dates in DB:** Always store as `TIMESTAMP WITH TIME ZONE` or UTC `Instant`. Never store local times without timezone — causes bugs with daylight saving time.

---

## Q89. Sealed Classes and Records — Compared to Traditional Approaches

### Concept
**Records** (Java 16) are a compact syntax for immutable data carriers. **Sealed classes** (Java 17) restrict which classes can extend a type, enabling exhaustive pattern matching.

### Simple Explanation
- **Records** = a data snapshot. Like a printed ticket: created with values, values never change, two tickets with same values are equal.
- **Sealed classes** = a closed family. Like a traffic light: only Red, Yellow, Green can exist — no unknown states, and the compiler knows that.

### Records — Immutable Data Carrier
```java
// Old way: 30 lines of boilerplate
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}

// Java 16 Record: ONE line. Auto-generates everything above.
record Point(int x, int y) {}

Point p = new Point(3, 4);
p.x();          // 3 — accessor (NOT getX())
p.y();          // 4
p.toString();   // "Point[x=3, y=4]"

// Two points with same values are equal:
new Point(3, 4).equals(new Point(3, 4)); // true — unlike plain classes

// Custom compact constructor (validation):
record Point(int x, int y) {
    Point {  // compact constructor — no parameter list needed
        if (x < 0 || y < 0) throw new IllegalArgumentException("Coordinates must be positive");
        // x and y are assigned automatically after this block
    }
}

// Records CAN:
// - Implement interfaces
// - Have static fields and methods
// - Have additional instance methods

// Records CANNOT:
// - Extend classes (implicitly extend Record)
// - Have mutable instance fields
// - Have additional instance fields beyond the record components
```

### Records vs Lombok @Value
| | Record | Lombok @Value |
|---|---|---|
| Language feature | ✅ Native Java | ❌ Annotation processing |
| Reflection tooling | ✅ Works out of the box | ⚠️ May need configuration |
| Custom constructors | ✅ Compact constructor | ✅ @Builder |
| Inheritance | ❌ Can't extend classes | ❌ Same (final class) |
| IDE support | ✅ Built-in | ✅ Plugin needed |

### Sealed Classes — Restricted Inheritance
```java
// Sealed: only listed types can implement/extend
sealed interface Shape permits Circle, Rectangle, Triangle {}

final class Circle    implements Shape { double radius; }
final class Rectangle implements Shape { double width, height; }
final class Triangle  implements Shape { double base, height; }
// No other class can implement Shape — compiler guarantees it

// Power: EXHAUSTIVE switch expressions (compiler catches missing cases)
double area = switch (shape) {
    case Circle    c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle  t -> 0.5 * t.base() * t.height();
    // No default needed — compiler knows these are ALL possible types!
};

// Compare to interface without sealed:
// default clause needed in switch — unknown subtypes could exist
```

### Real Project Usage
> "I replaced all DTO classes with Records — a `UserDTO` that was 45 lines became one line. Sealed interfaces are great for domain result types: `sealed interface PaymentResult permits Success, Failure, Pending {}` — forces callers to handle all outcomes at compile time, no forgotten cases."

### Interview Answer
> "Records are Java 16's syntax for immutable data carriers. You declare the fields in the header and Java auto-generates the canonical constructor, accessors, equals, hashCode, and toString. They're the native alternative to Lombok `@Value`. Sealed classes restrict which types can implement an interface — the compiler knows the complete set. The synergy is powerful: use sealed + records together, then use exhaustive switch expressions — the compiler guarantees you handled every case. No more forgotten `default` clauses hiding bugs."
>
> *Likely follow-up: "Can a record implement a sealed interface?"*

### Related Concepts & Common Follow-ups
- **Can a record implement a sealed interface?** Yes — and it's a perfect combination. `record Circle(double radius) implements Shape {}` where `Shape` is sealed.
- **Pattern matching in switch (Java 21):** `switch(obj) { case Integer i -> ...; case String s -> ...; }` — works on any type, not just enums. The natural extension of sealed + records.
- **Non-sealed keyword:** A subtype of a sealed class that opens the hierarchy back up: `non-sealed class OpenShape extends Shape {}` — now anything can extend `OpenShape`.
- **`record` vs `enum`:** Enum = fixed set of singleton instances (values known at compile time). Record = immutable data carrier (values determined at runtime). Both can implement interfaces.

---

## Summary: When to Use Each Feature

| Feature | Use When | Avoid When |
|---|---|---|
| **Lambdas** | Functional interface with simple logic (filter, map, etc.) | Complex logic (use method or class) |
| **Streams** | Transforming data pipelines (filter → map → collect) | Imperative loops with heavy side effects |
| **Optional** | Method returns "maybe value" | Parameter types (pass null or use wrapper) |
| **Records** | DTO, immutable data carrier | Mutable beans, complex constructors |
| **Sealed classes** | Domain model with finite types | Open extension points |
| **Pattern matching** | Type-based conditionals, instanceof checks | Simple equality checks (stick with if-else) |
| **Virtual Threads** | I/O-bound services (web, API calls, database) | CPU-bound algorithms, real-time constraints |

---

## Follow-up Questions Likely in Interview

1. **"Why is Optional not a perfect null replacement?"**
   - It's a wrapper; adds overhead. Many libraries + legacy code don't use it.
   - Nullability annotations (@Nullable, @Nonnull) still useful for static analysis.

2. **"Can Virtual Threads fully replace reactive code?"**
   - For I/O-bound yes. Reactive still needed for CPU-bound, backpressure, or Flux-style streaming.

3. **"What's the performance cost of records vs traditional classes?"**
   - Negligible after JIT compilation. Records are as fast or faster (simpler).

4. **"How would you migrate a Spring Boot 2.x app to Java 21?"**
   - Upgrade Spring Boot 3.x (breaking changes in Jakarta namespace).
   - Migrate DTOs to records.
   - Replace `@Transactional` thread pool logic with virtual threads where beneficial.

5. **"Can you use sealed classes in your REST API DTOs?"**
   - Yes! Use sealed interfaces + records for polymorphic request/response bodies.
   - Jackson handles serialization/deserialization with `@JsonTypeInfo`.

---

**End of Topic 1 — Ready for review** ✅

