# PART A: Core Java — Topic 4: JVM Architecture, Memory Management & Advanced Java Features

> **Duration:** 8–10 hours | **Est. Study Time per question:** 25–40 minutes

---

## Part 1: ADVANCED SERIALIZATION & MEMORY

## Q1. Serialization — Deep Dive

### Concept
Converting an object into a **byte stream** so it can be saved to a file, sent over a network, or stored in a database. Deserialization reverses the process.

### Simple Explanation
Like packing a laptop for shipping: disassemble into parts (serialization), pack in a box, ship, then unpack and reassemble at destination (deserialization).

### Key Components

| Component | Purpose |
|---|---|
| `Serializable` interface | Marker — signals "this class can be serialized" |
| `serialVersionUID` | Version control — ensures sender and receiver have compatible class versions |
| `transient` | Skip this field during serialization |
| `writeObject()` / `readObject()` | Custom serialization logic |
| `readResolve()` | Replaces deserialized object (used by Singleton pattern) |

### Basic Code

```java
@Data
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // ✅ Always declare explicitly
    
    private Long id;
    private String name;
    private String email;
    
    private transient String password;        // ❌ Never serialize passwords
    private transient int loginAttempts = 0;  // Runtime state — no need to persist
}

// Serialization
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    oos.writeObject(user);
}

// Deserialization
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) ois.readObject();
    // password = null, loginAttempts = 0 (default values, not from file)
}
```

### What is `serialVersionUID`?

```java
// Scenario 1: serialVersionUID NOT declared
// V1 sent over network:
class User implements Serializable {
    String name;
}

// Receiver has V2 with new field:
class User implements Serializable {
    String name;
    String email;  // new field added
}
// Result: InvalidClassException — auto-generated UIDs don't match


// Scenario 2: serialVersionUID DECLARED
class User implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
}

// V2 with same UID:
class User implements Serializable {
    private static final long serialVersionUID = 1L;  // SAME UID
    String name;
    String email;  // new field — backward compatible
}
// Result: Deserialization succeeds. email = null in deserialized object.
```

**Rule:** Always declare `serialVersionUID = 1L`. Increment to `2L` only when making **incompatible** changes (removing a field, changing type).

### Custom Serialization

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String name;
    private String password;
    private transient String encryptedPassword;  // computed, not stored
    
    // Custom serialization — encrypt password before writing
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();  // serialize non-transient fields first
        oos.writeObject(encrypt(password));  // manually write encrypted password
    }
    
    // Custom deserialization
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();  // deserialize non-transient fields first
        String encrypted = (String) ois.readObject();
        this.password = decrypt(encrypted);  // decrypt and populate
    }
    
    private String encrypt(String raw) { 
        return Base64.getEncoder().encodeToString(raw.getBytes()); 
    }
    
    private String decrypt(String enc) { 
        return new String(Base64.getDecoder().decode(enc)); 
    }
}
```

### Interview Answer

> "Serialization converts an object to a byte stream for network transfer or persistence. I always implement `Serializable` and declare `serialVersionUID = 1L` explicitly — this prevents `InvalidClassException` when the class evolves. Use `transient` for passwords and runtime-only state.
>
> For sensitive data, I override `writeObject()` and `readObject()` — encrypt before serialization, decrypt after. For example, password is encrypted before writing and decrypted during deserialization.
>
> Common pitfall: forgetting that the parent class must also be `Serializable` if you want parent fields serialized. Static fields are never serialized — they belong to the class, not instances."

> *Likely follow-up: "What happens if `serialVersionUID` mismatches between sender and receiver?"*

---

## Q2. `transient` Keyword

### Concept
A **Java language keyword** that excludes a field from serialization — even though the object is serializable.

### When to Use

```java
public class Session implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String userId;
    private Instant createdAt;
    
    // Runtime-only state — don't persist
    private transient int requestCount;            // reset to 0 on deserialization
    private transient Map<String, Object> cache;   // rebuild on app restart
    
    // Security — never serialize
    private transient String oauthToken;
    private transient String password;
    
    // Derived fields — recompute after deserialization
    private transient boolean isExpired;
}
```

### After Deserialization

```
Before serialization:
  requestCount = 42
  cache = {key1=val1, key2=val2}
  password = "secret"

After deserialization:
  requestCount = 0        // default int value
  cache = null            // default object value
  password = null         // you must handle this in readObject()
```

### Interview Answer

> "`transient` is a Java language keyword, not Spring-specific. It tells the serialization mechanism to skip that field. Use it for passwords, OAuth tokens, runtime counters, and derived fields. After deserialization, `transient` fields get default values — null for objects, 0 for numbers. If you need them populated, override `readObject()` and recompute them manually."

---

## Q3. Singleton Breaking During Serialization

### The Problem

```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}  // private constructor
    public static Singleton getInstance() { return INSTANCE; }
}

// Test
Singleton s1 = Singleton.getInstance();

// Serialize
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
oos.writeObject(s1);

// Deserialize
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton s2 = (Singleton) ois.readObject();

// ❌ BROKEN: s1 != s2  (two different instances!)
```

### Why It Breaks

During deserialization, Java creates a **new instance** using reflection — bypasses the private constructor. Singleton guarantee violated.

### Solution: `readResolve()`

```java
public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
    
    // ✅ FIX: Return existing instance instead of deserialized new one
    private Object readResolve() {
        return INSTANCE;  // replaces deserialized object with existing singleton
    }
}

// Now: s1 == s2 ✓
```

### How `readResolve()` Works

```
1. Deserialization creates new object (let's call it tempObj)
2. JVM checks if readResolve() method exists
3. If yes: calls readResolve(), returns that object instead of tempObj
4. tempObj is garbage collected
5. Result: singleton preserved
```

### Interview Answer

> "Serialization breaks singleton because deserialization creates a new instance via reflection, bypassing the private constructor. The fix: implement `readResolve()` method that returns the existing singleton instance. During deserialization, JVM calls `readResolve()` and replaces the newly created object with the singleton. Without this, you end up with two instances.
>
> **Follow-up:** *What about enum-based singletons?*
> — Enums have built-in serialization safety. Java guarantees singleton behavior even with serialization — no `readResolve()` needed. That's why 'Effective Java' recommends enum for singletons."

---

## Q4. Lombok with Serialization

### The Issue

Lombok generates constructors, getters, setters. If you're using `@Data` or `@AllArgsConstructor`, it may interfere with the default deserialization process if not careful.

### Best Practice

```java
import lombok.Data;
import java.io.Serializable;

@Data  // ✅ Works fine with Serializable
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // ⚠️ MUST be explicit — Lombok won't generate
    
    private Long id;
    private String name;
    private String email;
    
    private transient String password;
}
```

### Key Rules

1. **Always declare `serialVersionUID` manually** — Lombok doesn't generate it.
2. **`@Data` is safe** — generates no-arg constructor implicitly (required for deserialization).
3. **`@AllArgsConstructor` alone is NOT safe** — removes no-arg constructor → deserialization fails.

```java
// ❌ BREAKS DESERIALIZATION
@AllArgsConstructor  // removes no-arg constructor
public class User implements Serializable { }

// ✅ FIX
@AllArgsConstructor
@NoArgsConstructor   // explicitly add no-arg constructor
public class User implements Serializable { }
```

### Interview Answer

> "Lombok and serialization work fine together. The key rule: always declare `serialVersionUID` explicitly — Lombok won't generate it. If using `@AllArgsConstructor`, also add `@NoArgsConstructor` — deserialization requires a no-arg constructor. `@Data` includes `@RequiredArgsConstructor` which is safe since it doesn't remove the implicit no-arg constructor when there are no final fields."

---

## Q5. Try-With-Resources

### Concept
Automatically closes resources (files, DB connections, network sockets) when try block exits — even if an exception is thrown. Eliminates resource leaks and verbose finally blocks.

### Simple Explanation
Restaurant: without try-with-resources, you manually remember to turn off lights when leaving. With try-with-resources, smart lights auto-turn-off when you exit.

### Old Way (Pre-Java 7)

```java
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("data.txt"));
    String line = reader.readLine();
    // ... process
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (reader != null) {
        try {
            reader.close();  // manually close, may throw exception too
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### New Way (Java 7+)

```java
// Resource automatically closed at end of try block
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    String line = reader.readLine();
    // ... process
} catch (IOException e) {
    e.printStackTrace();
}
// reader.close() called automatically — even if exception thrown
```

### Multiple Resources

```java
try (
    FileInputStream fis = new FileInputStream("input.txt");
    FileOutputStream fos = new FileOutputStream("output.txt");
    BufferedReader reader = new BufferedReader(new InputStreamReader(fis))
) {
    String line;
    while ((line = reader.readLine()) != null) {
        fos.write(line.getBytes());
    }
}
// All 3 resources closed automatically in REVERSE order: reader → fos → fis
```

### Requirements

Any class used in try-with-resources must implement `AutoCloseable` (or `Closeable`, which extends `AutoCloseable`).

```java
public class DatabaseConnection implements AutoCloseable {
    private Connection conn;
    
    public DatabaseConnection(String url) throws SQLException {
        this.conn = DriverManager.getConnection(url);
    }
    
    public ResultSet query(String sql) throws SQLException {
        return conn.createStatement().executeQuery(sql);
    }
    
    @Override
    public void close() throws SQLException {
        if (conn != null && !conn.isClosed()) {
            conn.close();
            System.out.println("Connection closed");
        }
    }
}

// Usage
try (DatabaseConnection db = new DatabaseConnection("jdbc:...")) {
    ResultSet rs = db.query("SELECT * FROM users");
    // ... process
}  // close() called automatically
```

### Interview Answer

> "Try-with-resources automatically closes resources at the end of the try block, even if an exception is thrown. Any class implementing `AutoCloseable` can be used. It eliminates resource leaks and removes verbose finally blocks. Multiple resources are closed in reverse order of declaration. Common use: DB connections, file streams, HTTP clients. Without it, forgetting to close a connection in a high-traffic app causes connection pool exhaustion."

---

## Q6. Date-Time API (`java.time`)

### Why Old APIs Were Bad

| Old API | Problem |
|---|---|
| `java.util.Date` | Mutable — thread-unsafe. Month is 0-indexed (Jan = 0). No timezone support. |
| `java.util.Calendar` | Mutable, verbose, counterintuitive. Month still 0-indexed. |
| `SimpleDateFormat` | **Not thread-safe** — causes data corruption in multi-threaded apps. |

### New API (`java.time` — Java 8+)

```
✅ Immutable — thread-safe by default
✅ Fluent API — chainable methods
✅ Clear separation: LocalDate, LocalTime, LocalDateTime, ZonedDateTime, Instant
✅ Month is 1-indexed (Jan = 1) — natural
✅ DateTimeFormatter is thread-safe
```

### Key Classes

| Class | Use Case | Example |
|---|---|---|
| `LocalDate` | Date only (no time, no timezone) | Birthday: `2000-05-15` |
| `LocalTime` | Time only (no date, no timezone) | Meeting: `14:30:00` |
| `LocalDateTime` | Date + time (no timezone) | Log ts: `2026-03-13T14:30:00` |
| `ZonedDateTime` | Date + time + timezone | Flight: `2026-03-13T14:30:00+05:30[IST]` |
| `Instant` | Unix timestamp (UTC) | DB timestamp, server logs |
| `Duration` | Time-based (h, m, s) | Video: `PT2H30M` |
| `Period` | Date-based (y, m, d) | Age: `P5Y3M10D` |

### Code Examples

```java
// Current date/time
LocalDate today = LocalDate.now();                    // 2026-03-13
LocalTime now = LocalTime.now();                      // 14:30:45.123
LocalDateTime dateTime = LocalDateTime.now();         // 2026-03-13T14:30:45.123
Instant instant = Instant.now();                      // 2026-03-13T09:00:45.123Z (UTC)

// Parsing
LocalDate date = LocalDate.parse("2026-03-13");
LocalDateTime dt = LocalDateTime.parse("2026-03-13T14:30:00");
ZonedDateTime zdt = ZonedDateTime.parse("2026-03-13T14:30:00+05:30[Asia/Kolkata]");

// Formatting
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy HH:mm");
String formatted = dateTime.format(formatter);        // "13-03-2026 14:30"

// Calculations — immutable, returns new instance
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);
LocalDateTime tomorrow2PM = dateTime.plusDays(1).withHour(14).withMinute(0);

// Duration and Period
Duration duration = Duration.between(time1, time2);   // PT5H30M
Period period = Period.between(birthDate, today);     // P26Y2M10D

// Timezone conversion
ZonedDateTime istTime = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));
ZonedDateTime utcTime = istTime.withZoneSameInstant(ZoneId.of("UTC"));
```

### Real Project Usage

```java
@Entity
public class Order {
    @Id private Long id;
    
    // ✅ Use Instant for DB timestamps (always UTC)
    private Instant createdAt;
    
    // ✅ Use LocalDate for business dates (delivery date, birth date)
    private LocalDate deliveryDate;
    
    @PrePersist
    void onCreate() {
        createdAt = Instant.now();  // UTC timestamp
    }
}

// Service — calculate delivery date
public LocalDate calculateDeliveryDate(LocalDate orderDate) {
    return orderDate.plusDays(3);  // 3-day delivery
}

// Controller — return ISO-8601 strings to frontend
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    Order order = orderRepo.findById(id);
    return OrderDto.builder()
        .createdAt(order.getCreatedAt().toString())  // "2026-03-13T09:00:45.123Z"
        .deliveryDate(order.getDeliveryDate().toString())  // "2026-03-16"
        .build();
}
```

### Thread-Safety Comparison

```java
// ❌ Old API — NOT thread-safe
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// Used by multiple threads → data corruption

// ✅ New API — thread-safe
DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");
// Can be static final, shared across threads safely
```

### Interview Answer

> "`java.time` solves all problems of the old Date/Calendar APIs: it's immutable (thread-safe), has clear separation (LocalDate for dates, Instant for timestamps), and formatters are thread-safe. I use `Instant` for all database timestamps (always UTC), `LocalDate` for business dates like delivery date or birthday, and `ZonedDateTime` when timezone matters (flight times, scheduled tasks).
>
> Old `SimpleDateFormat` was not thread-safe — in a multi-threaded controller it caused random data corruption. `DateTimeFormatter` is thread-safe — can be a static final constant.
>
> **Follow-up:** *How do you handle timezone in APIs?*
> — Store everything in UTC (Instant). When sending to frontend, convert to user's timezone if needed. The API accepts and returns ISO-8601 strings with timezone offsets."

---

## Q7. Sealed Classes & Records

### Sealed Classes (Java 17+)

#### Concept

Restrict which classes can extend/implement a type. All permitted subclasses must be declared explicitly.

#### Simple Explanation

VIP club: only specific pre-approved people can enter. No random person can join. The club (sealed class) explicitly lists who's allowed.

#### Code

```java
public sealed class PaymentMethod permits CreditCard, DebitCard, UPI {
    // Only these 3 classes can extend PaymentMethod
    // No one else can create subclasses
}

public final class CreditCard extends PaymentMethod { }
public final class DebitCard extends PaymentMethod { }
public non-sealed class UPI extends PaymentMethod { }  // allows further subclasses

// ❌ This will NOT compile
// public class Bitcoin extends PaymentMethod { }  // ERROR: not in permits list
```

#### Why Use Sealed Classes

```java
// Pattern matching with exhaustiveness checking
public BigDecimal processFee(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc -> cc.getAmount().multiply(new BigDecimal("0.02"));
        case DebitCard dc -> dc.getAmount().multiply(new BigDecimal("0.01"));
        case UPI upi -> BigDecimal.ZERO;
        // No default needed — compiler knows all possible types
    };
}
```

### Records (Java 14+)

#### Concept

Immutable data carriers. Boilerplate-free: no need to write constructor, getters, `equals()`, `hashCode()`, `toString()`.

#### Traditional Immutable Class (30 lines)

```java
public final class User {
    private final Long id;
    private final String name;
    private final String email;
    
    public User(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    @Override
    public boolean equals(Object o) { /* ... */ }
    
    @Override
    public int hashCode() { /* ... */ }
    
    @Override
    public String toString() { /* ... */ }
}
```

#### Record — Same Thing in 1 Line

```java
public record User(Long id, String name, String email) {}

// Auto-generated:
// - Constructor: new User(1L, "John", "john@example.com")
// - Getters: user.id(), user.name(), user.email()  (no "get" prefix!)
// - equals(), hashCode(), toString()
// - All fields are private final (immutable)
```

#### Records in Practice

```java
// DTO for API responses
public record OrderDto(Long id, String status, BigDecimal total, Instant createdAt) {}

@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    Order order = orderRepo.findById(id);
    return new OrderDto(order.getId(), order.getStatus(), order.getTotal(), order.getCreatedAt());
}

// Compact constructor — validation
public record User(Long id, String name, String email) {
    public User {  // compact constructor syntax
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
}

// Custom methods allowed
public record Point(int x, int y) {
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
}
```

#### Records vs Traditional Classes

| Feature | Record | Traditional Class |
|---|---|---|
| Boilerplate | Zero | High |
| Immutability | Enforced | Manual |
| Inheritance | ❌ Cannot extend | ✅ Can extend |
| Use for | DTOs, value objects | Complex domain objects |
| Getter naming | `user.name()` | `user.getName()` |

#### Interview Answer

> "Records eliminate boilerplate for immutable data carriers — DTOs, value objects. One line gives you constructor, getters (without 'get' prefix), equals, hashCode, toString. All fields are private final. I use them for API response DTOs and internal value objects.
>
> Records can't extend classes (but can implement interfaces). For complex domain entities with behavior, I stick with traditional classes or Lombok.
>
> Sealed classes restrict who can extend a type — useful for closed type hierarchies like payment methods. Combined with pattern matching, the compiler checks exhaustiveness — no default case needed in switch.
>
> **Follow-up:** *Can you use records with JPA entities?*
> — Not recommended. JPA requires mutable fields and a no-arg constructor. Records are immutable and don't support no-arg constructors naturally. Use records for read-only DTOs, regular classes for entities."

---

## Part 2: JVM FUNDAMENTALS

## Q8. JVM Architecture — End to End

### Concept
The JVM (Java Virtual Machine) is the runtime engine that executes Java bytecode, abstracting hardware differences so Java code runs on any platform.

### Simple Explanation
Think of JVM as a universal translator. You write Java code, the compiler converts it to bytecode (`.class` files) — a platform-neutral language. The JVM on each machine translates that bytecode into native machine instructions. That's "Write Once, Run Anywhere."

### JVM Architecture Layers

```
Your Java code → javac → Bytecode (.class)
                                ↓
                    ┌─── JVM ─────────────────────┐
                    │                             │
                    │  1. ClassLoader Subsystem   │
                    │     Loading → Linking →     │
                    │     Initialization          │
                    │                             │
                    │  2. Runtime Data Areas      │
                    │     Method Area (Metaspace) │
                    │     Heap                    │
                    │     Stack (per thread)      │
                    │     PC Register             │
                    │     Native Method Stack     │
                    │                             │
                    │  3. Execution Engine        │
                    │     Interpreter             │
                    │     JIT Compiler            │
                    │     Garbage Collector       │
                    └─────────────────────────────┘
```

### 1. ClassLoader Subsystem

```java
// Three phases:
// Loading: finds .class file (Bootstrap → Extension → Application classloader)
// Linking:
//   - Verify: checks bytecode is valid
//   - Prepare: allocates memory for static variables (default values)
//   - Resolve: replaces symbolic references with direct references
// Initialization: runs static blocks and assigns static field values

// ClassLoader hierarchy (parent delegation model):
// Bootstrap ClassLoader → loads core Java (java.lang.*, java.util.*)
//   └── Extension ClassLoader → loads JDK extensions (lib/ext)
//       └── Application ClassLoader → loads your classes (classpath)

// Always asks parent first. If parent can't load it, child tries.
// Prevents malicious class overriding java.lang.String
```

### 2. Runtime Data Areas

```
Method Area (Metaspace in Java 8+):
  - Class metadata: class names, method names, field names
  - Bytecode of methods
  - Static variables
  - Constant pool
  Shared across all threads. In Java 8+: off-heap (native memory, not limited by -Xmx)

Heap:
  - All object instances
  - Instance variables
  - Arrays
  Shared across all threads. Controlled by -Xms and -Xmx.

Stack (per thread):
  - One Stack per thread
  - Each method call creates a Stack Frame:
      - Local variables
      - Operand stack (intermediate values)
      - Reference to Constant Pool
  - Destroyed when method returns

PC Register (per thread):
  - Points to the currently executing bytecode instruction

Native Method Stack:
  - Used when Java calls native (C/C++) code via JNI
```

### 3. Execution Engine

```java
// Interpreter: executes bytecode line by line
// Fast startup but slow for hot code paths

// JIT Compiler (Just-In-Time): identifies hot methods (executed many times)
// and compiles them to native machine code → huge speed improvement
// Tiered compilation: C1 (fast compile) → C2 (optimised compile)

// How JIT detects hot methods:
// Each method has a counter. When counter > threshold → JIT compiles it
// Result cached in Code Cache (native memory)

// Garbage Collector: automatic memory reclamation (see Q9-Q10)
```

### Interview Answer

> "The JVM has three main subsystems. The ClassLoader loads `.class` files in three phases — loading, linking (verify/prepare/resolve), and initialisation — using a parent delegation model to prevent class spoofing. The Runtime Data Areas include: the Heap for objects, the Stack per thread for method frames, and Metaspace for class metadata. The Execution Engine interprets bytecode initially, then JIT-compiles hot methods to native code for performance. The JIT's tiered compilation is why Java applications start slightly slow but get faster over time — it's optimising as it learns what gets called most."

> *Likely follow-up: "What is the parent delegation model in ClassLoader and why does it exist?"*

#### Related Concepts

- **Parent delegation model:** Child classloader always asks parent first. Prevents `java.lang.String` from being overridden by a custom class on the classpath. If parent can't load → child tries.
- **What is the Code Cache?** Native memory where JIT-compiled machine code is stored. If full → JIT stops compiling → performance degrades. Configurable with `-XX:ReservedCodeCacheSize`.
- **Bootstrap ClassLoader is written in C** — it's the only loader without a Java parent. `ClassLoader.getParent()` returns `null` for classes loaded by bootstrap.

---

## Q9. JDK vs JRE vs JVM

### Concept

Three nested layers of the Java platform: JVM executes code, JRE adds the standard libraries, JDK adds developer tools.

### Simple Explanation
- **JVM** = the engine of a car (runs code)
- **JRE** = the car (engine + body + everything needed to drive)
- **JDK** = the car + the factory + all tools to build more cars

```
JDK
├── Development Tools: javac, jar, javadoc, jdb (debugger), jshell
└── JRE
    ├── Java Standard Library: java.lang, java.util, java.io, Spring, etc.
    └── JVM
        └── Executes bytecode (.class files)
```

```java
// In practice:
// Production server: install JRE (or JDK 11+ which ships JRE embedded)
// Developer machine: install JDK

// In Docker:
FROM eclipse-temurin:21-jdk  // For building (has javac)
FROM eclipse-temurin:21-jre  // For running (smaller image, no javac)

// Multi-stage Docker build (best practice):
FROM eclipse-temurin:21-jdk AS builder
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre  // Final image: smaller, no build tools
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Interview Answer

> "JVM is the runtime that executes bytecode — it's the 'Write Once, Run Anywhere' layer. JRE = JVM + standard libraries needed to run Java applications. JDK = JRE + development tools like javac (compiler), jar, javadoc, and jshell. In production you traditionally installed JRE only. Since Java 11, JDK ships with JRE embedded, so most teams just install JDK everywhere. For Docker images I use a JDK base for building and a JRE base for the final production image to keep it small."

#### Related Concepts

- **GraalVM native image:** Compiles Java to a native binary ahead-of-time. No JVM at runtime. Near-instant startup, low memory — ideal for serverless/Lambda. Trade-off: no dynamic class loading, longer build time.
- **jlink (Java 9+):** Creates a minimal custom JRE containing only the modules your app uses. Reduces image size significantly.

---

## Part 3: MEMORY MANAGEMENT

## Q10. Heap vs Stack Memory

### Concept
The JVM uses two primary memory regions: the Heap for objects that live beyond a single method call, and the Stack for method execution frames that are created and destroyed with each method call.

### Simple Explanation
- **Stack** = a pile of trays in a cafeteria. You put a new tray on top when a method starts, take it off when the method returns. Very organised, automatic.
- **Heap** = the warehouse. Objects are stored here and can live as long as something refers to them. The GC cleans up the ones nobody references anymore.

### Code

```java
public class MemoryDemo {
    // Static field → Method Area (Metaspace), not heap
    private static final String APP_NAME = "Demo";

    public void process() {
        // Stack: primitive 'count' — lives in this stack frame
        int count = 5;

        // Stack: 'user' reference — the reference lives on stack
        // Heap: the User OBJECT lives on heap
        User user = new User("Alice");

        // Stack: 'name' reference
        // Heap: "Alice" string object (may be in String pool)
        String name = user.getName();

        helper(count); // New stack frame pushed
        // After helper() returns: helper's frame popped
    } // process() returns: its stack frame popped
      // user reference gone from stack → User object eligible for GC
}

// Memory layout during process():
// Stack (thread):           Heap:
// ┌──────────────────┐     ┌────────────────────────────┐
// │ process frame    │     │ User object { name="Alice"}│
// │  count = 5       │────→│                            │
// │  user → (ref)    │     │ String "Alice"             │
// │  name → (ref)    │     └────────────────────────────┘
// └──────────────────┘
```

| | Stack | Heap |
|---|---|---|
| **Stores** | Primitives, object references, method frames | All object instances, arrays |
| **Size** | Small (512KB–1MB default per thread) | Large (configured by -Xmx) |
| **Lifetime** | Method duration | Until GC collects |
| **Thread safety** | Each thread has its own stack | Shared — needs synchronisation |
| **Error when full** | `StackOverflowError` (infinite recursion) | `OutOfMemoryError` |
| **Speed** | Very fast (LIFO, cache-friendly) | Slower (pointer-chasing) |

```java
// StackOverflowError — infinite recursion fills the stack
public int factorial(int n) {
    return n * factorial(n - 1); // Missing base case → stack overflow
}

// OutOfMemoryError — heap exhausted
List<byte[]> leak = new ArrayList<>();
while (true) {
    leak.add(new byte[1024 * 1024]); // 1MB each → heap fills → OOM
}
```

### Interview Answer

> "The Stack stores method frames — local variables, primitive values, and object references. Each thread gets its own stack. When a method is called, a new frame is pushed; when it returns, the frame is popped — automatic, no GC needed. Stack overflow happens with infinite recursion. The Heap stores all object instances, shared across threads. Objects live until the GC determines nothing references them. OutOfMemoryError means the heap is full — usually a memory leak. The key insight: primitives in local variables go on the Stack, but primitives inside objects go on the Heap because the whole object is on the Heap."

> *Likely follow-up: "Where do static variables live?"*

#### Related Concepts

- **Where do static variables live?** In the Method Area (Metaspace in Java 8+), not the heap. They persist for the entire JVM lifetime.
- **Where does String pool live?** In the Heap since Java 7 (was in PermGen before). Interned strings and string literals are stored in the pool.
- **Escape analysis (JIT optimisation):** If the JIT compiler determines an object never escapes the method (not returned, not stored externally), it may allocate it on the stack instead of heap — no GC pressure. Transparent to the developer.

---

## Q11. StackOverflowError vs OutOfMemoryError

### Concept
Two JVM crash errors with different root causes: `StackOverflowError` = too many nested method calls; `OutOfMemoryError` = heap (or other memory region) exhausted.

### StackOverflowError

```java
// Cause: infinite (or very deep) recursion fills the thread's stack
public int badFactorial(int n) {
    return n * badFactorial(n - 1); // No base case → infinite recursion
}

// Fix: add base case
public int factorial(int n) {
    if (n <= 1) return 1; // Base case stops recursion
    return n * factorial(n - 1);
}

// Fix for genuinely deep recursion: convert to iterative
public int factorialIterative(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) result *= i;
    return result;
}

// Or increase stack size (rarely needed):
// java -Xss4m MyApp   (default is ~512KB-1MB)
```

### OutOfMemoryError

```java
// Most common cause: memory leak — objects held in a collection that grows forever
@Service
public class LeakyService {
    private final List<UserSession> sessions = new ArrayList<>(); // Static-scoped list

    public void onLogin(UserSession session) {
        sessions.add(session); // Added but NEVER removed → grows forever
    }
    // Fix: use WeakReference, or evict entries, or use a bounded cache
}

// Another cause: too many threads (each needs stack memory)
for (int i = 0; i < 100_000; i++) {
    new Thread(() -> { /* work */ }).start(); // OutOfMemoryError: unable to create new native thread
}

// Different OOM flavours:
// OutOfMemoryError: Java heap space         → heap full, increase -Xmx or fix leak
// OutOfMemoryError: GC overhead limit exceeded → GC taking >98% time, <2% reclaimed
// OutOfMemoryError: Metaspace               → class metadata full, increase -XX:MaxMetaspaceSize
// OutOfMemoryError: unable to create new native thread → too many threads

// Diagnosing OOM:
// 1. Add: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
// 2. Analyse heap dump with VisualVM, MAT (Eclipse Memory Analyser), or JProfiler
// 3. Look for: largest retained heap, growing collections, cached objects not evicted
```

### Key Difference

| | StackOverflowError | OutOfMemoryError |
|---|---|---|
| **Memory region** | Stack (per-thread) | Heap / Metaspace / native |
| **Cause** | Infinite/very deep recursion | Object leak or insufficient heap |
| **Recovery** | Caught as Error — usually fatal | Caught as Error — usually fatal |
| **Diagnosis** | Thread dump (jstack) | Heap dump (-XX:+HeapDumpOnOutOfMemoryError) |
| **Fix** | Add base case / convert to iterative | Fix leak or increase -Xmx |

### Interview Answer

> "StackOverflowError means a thread's stack space is exhausted from too many nested method calls — almost always infinite or missing-base-case recursion. It's fixed by adding a base case or converting to an iterative approach. OutOfMemoryError means the heap (or another memory region) is full. The most common cause is a memory leak — objects being held in a collection that grows without bound. I diagnose OOM by enabling `-XX:+HeapDumpOnOutOfMemoryError` and analysing the dump in VisualVM — look for which collection or cache is holding the most retained heap."

> *Likely follow-up: "What is GC overhead limit exceeded?"*

#### Related Concepts

- **GC overhead limit exceeded:** JVM is spending more than 98% of its time in GC but only reclaiming less than 2% of heap — effectively means the app is nearly dead from GC thrashing. Usually a leak.
- **Metaspace OOM:** Too many classes loaded (class generation at runtime via reflection, Groovy scripts, hot deployment). Increase `-XX:MaxMetaspaceSize` or fix class generation.
- **WeakReference and SoftReference for caches:** `WeakReference` — collected as soon as no strong reference. `SoftReference` — collected only when heap is under pressure. Ideal for memory-sensitive caches.

---

## Part 4: GARBAGE COLLECTION

## Q12. Garbage Collection — Full GC vs Partial GC

### Concept
GC automatically reclaims memory from objects that are no longer reachable, preventing manual memory management bugs.

### Simple Explanation
Think of the heap as a house. Minor/Partial GC is like tidying one room — the nursery (Young Gen) — quickly and often. Full GC is a full house deep-clean — everything stops, every room gets checked. You want Full GC to be rare.

### Heap Memory Layout

```
JVM Heap:
┌─────────────────────────────────────────────────────────────────┐
│  Young Generation                   │  Old Generation           │
│  ┌──────────┬───────────┬─────────┐ │  (Long-lived objects)     │
│  │  Eden    │ Survivor0 │Survivor1│ │                           │
│  │(new objs)│  (S0)     │  (S1)   │ │                           │
│  └──────────┴───────────┴─────────┘ │                           │
└─────────────────────────────────────────────────────────────────┘

Non-Heap: Metaspace (class metadata — off heap in Java 8+)
```

### Object Lifecycle

```java
// 1. Object created → Eden (Young Gen)
Object obj = new Object(); // Goes to Eden

// 2. Minor GC happens when Eden fills:
//    - Mark live objects in Eden
//    - Copy survivors to S0 (or S1), increment age counter
//    - Clear Eden
//    - Next GC: copy S0 survivors to S1, clear S0

// 3. After ~15 Minor GCs (age threshold): object PROMOTED to Old Gen
//    (tunable with -XX:MaxTenuringThreshold=15)

// 4. Old Gen fills → Major GC (collects just Old Gen)
//    or Full GC (collects everything + Metaspace)

// Full GC also triggered by:
// - System.gc() call (just a hint — JVM may ignore)
// - Concurrent GC can't keep up → falls back to Full GC
// - Explicit: jcmd <pid> GC.run
```

### GC Event Types

| Event | Scope | Pause | Trigger |
|---|---|---|---|
| **Minor GC** | Young Gen only | Very short (ms) | Eden fills |
| **Major GC** | Old Gen | Longer | Old Gen fills |
| **Full GC** | Entire Heap + Metaspace | Longest (STW) | Fallback, low memory |

### Mark-and-Sweep Algorithm

```
Phase 1 — MARK:
  Start from GC Roots (thread stacks, static fields, JNI refs, active threads)
  Graph traverse: mark every reachable object as "alive"
  Unmarked objects = unreachable = garbage

Phase 2 — SWEEP:
  Reclaim memory of all unmarked objects

Phase 3 — COMPACT (optional, done by G1/ZGC):
  Move live objects together to eliminate fragmentation
  Update all references to point to new locations
```

### Interview Answer

> "GC prevents memory leaks by automatically reclaiming unreachable objects. The heap is split into Young and Old Generation. New objects are created in Eden (Young Gen). Minor GC is fast and frequent — it only cleans Eden and Survivor spaces using a copy-and-promote strategy. Objects that survive enough Minor GCs are promoted to Old Gen. Full GC is a Stop-The-World event that pauses all application threads while it collects the entire heap — in production you want this to be rare. I tune GC by setting `-Xmx` appropriately and monitoring for promotion failures or high Full GC frequency in GC logs."

> *Likely follow-up: "What is a Stop-the-World event?"*

#### Related Concepts

- **Stop-the-World (STW):** All application threads are paused while GC runs. Even G1 and ZGC have STW phases but keep them very short. STW during Full GC can cause visible latency spikes.
- **Promotion failure:** Old Gen doesn't have enough space to receive objects promoted from Young Gen → triggers Full GC.
- **GC logging:** Enable with `-Xlog:gc*:file=/var/log/gc.log:time,level,tags` (Java 9+). Shows timing and cause of each GC event.

---

## Q13. Garbage Collectors — Types and When to Choose

### Concept
The JVM offers multiple GC algorithms, each making different trade-offs between throughput, latency, and memory footprint.

### Simple Explanation
GC algorithms are like different cleaning crews:
- **Serial** = one cleaner, stops everything to clean
- **Parallel** = multiple cleaners, still stops everything but faster
- **G1** = smart crew that cleans in sections with minimal pauses
- **ZGC** = a crew so fast the building barely notices they were there

### GC Types Comparison

```java
// Serial GC: single-threaded, stop-the-world
// -XX:+UseSerialGC
// Use: development, small embedded apps, <100MB heap

// Parallel GC (Throughput GC): multi-threaded, stop-the-world
// -XX:+UseParallelGC  -XX:ParallelGCThreads=8
// Default before Java 9
// Use: batch jobs, high throughput over low latency

// G1 GC (Garbage First): default since Java 9
// -XX:+UseG1GC  -XX:MaxGCPauseMillis=200
// Divides heap into ~2000 equal-size regions (1-32MB each)
// Collects regions with most garbage first (hence "Garbage First")
// STW pauses < 200ms by default
// Use: most web applications, 4GB-100GB heaps

// ZGC: ultra-low latency, concurrent (Java 15+)
// -XX:+UseZGC
// STW pauses < 1ms regardless of heap size
// Works concurrently with application threads (coloured pointers)
// Use: real-time systems, very large heaps (multi-TB), latency-sensitive APIs
// Trade-off: slightly lower throughput than G1

// Shenandoah (OpenJDK): similar to ZGC
// -XX:+UseShenandoahGC
// Also sub-millisecond pauses, concurrent compaction
```

### G1 Deep Dive (most asked)

```
G1 Heap Layout (not contiguous Young/Old — flexible regions):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ S  │ O  │ E  │ O  │ H  │ O  │ E  │
└────┴────┴────┴────┴────┴────┴────┴────┘
E=Eden, S=Survivor, O=Old, H=Humongous (large objects ≥ 50% of region)

G1 phases:
1. Young GC (STW, ~few ms): collect all Eden + Survivor regions
2. Concurrent Marking: mark live objects across Old regions (runs WITH app)
3. Mixed GC (STW): collect Young + some Old regions with most garbage
4. Full GC (last resort, STW): emergency fallback if concurrent can't keep up
```

### When G1 vs ZGC

| | G1 | ZGC |
|---|---|---|
| Java version | Java 9+ default | Java 15+ |
| Pause target | ~200ms | <1ms |
| Throughput | Higher | Slightly lower |
| Memory overhead | Lower | Higher (coloured pointers) |
| Best for | Most microservices | Latency-critical, huge heaps |

### Interview Answer

> "G1 is the default since Java 9 and the right choice for most microservices — it targets sub-200ms pauses by dividing the heap into regions and collecting the most garbage-dense ones first. ZGC is for when you can't tolerate even 200ms pauses — real-time systems or very large heaps. It runs almost entirely concurrent with the application, keeping pauses under 1ms. Serial GC is for tiny apps. Parallel GC is for batch jobs where throughput matters more than latency. In my services I use G1 with `-XX:MaxGCPauseMillis=100` and monitor via `/actuator/metrics/jvm.gc.pause`."

> *Likely follow-up: "What is the difference between concurrent and parallel GC?"*

#### Related Concepts

- **Concurrent vs Parallel GC:** Parallel = multiple GC threads, but still STW (application paused). Concurrent = GC runs simultaneously WITH application threads (application not fully paused). G1 has both phases. ZGC is mostly concurrent.
- **Humongous objects (G1):** Objects ≥ 50% of a region size. Allocated directly in Old Gen, GC'd only during Full GC or concurrent cycle. Large arrays/byte buffers can cause issues — prefer off-heap (ByteBuffer.allocateDirect) for very large objects.
- **How to diagnose GC issues:** Enable GC logging → look for: frequent Full GC, promotion failure, long Young GC pauses, GC overhead limit exceeded.

---

## Q14. Where Does the String Pool Live? (Java 7, 8, 11)

### Concept
The String pool (also called String intern pool) is a special cache that stores unique string literals to avoid creating duplicate objects.

### Simple Explanation
The String pool is like a postal address directory. When you create `"hello"` as a literal, Java checks: does this string exist in the directory? If yes — point to the existing one. If no — add a new entry. This way ten classes all using `"hello"` share one object.

### Location Across Versions

```
Java 6 and earlier:
  → String pool in PermGen (Permanent Generation)
  → PermGen = fixed-size, off-heap
  → Problem: pool could fill up → OutOfMemoryError: PermGen space
  → Default small size (32MB–128MB)

Java 7:
  → String pool MOVED to the Heap
  → Can be GC'd like any other object
  → Heap is much larger and resizable

Java 8:
  → PermGen REMOVED, replaced by Metaspace (unlimited by default)
  → String pool still in Heap
  → Metaspace holds class metadata only — NOT strings

Java 11+:
  → String pool still in Heap
  → No change from Java 8
  → Compact Strings (Java 9+): strings of Latin-1 chars use byte[] instead of char[] → 50% memory saving for ASCII-heavy apps
```

### Code: String Pool Behaviour

```java
// String literal → String pool
String a = "hello"; // Checks pool: not found → adds to pool, a points to it
String b = "hello"; // Checks pool: found → b points to SAME object as a
System.out.println(a == b); // true — same reference from pool

// new String() → Heap (bypasses pool)
String c = new String("hello"); // Forces new object on heap, NOT in pool
System.out.println(a == c);     // false — different objects even though same content
System.out.println(a.equals(c)); // true — same content

// intern() — explicitly add to pool or return existing
String d = c.intern(); // "hello" already in pool → returns pool reference
System.out.println(a == d); // true — d now points to pool object

// Concatenation:
String e = "hel" + "lo"; // Compile-time constant → folded to "hello" → pool
System.out.println(a == e); // true

String f = "hel";
String g = f + "lo";       // Runtime concatenation → new heap object (NOT pool)
System.out.println(a == g); // false
```

### Interview Answer

> "The String pool was in PermGen in Java 6 and earlier — a fixed-size non-heap region that could run out with too many string literals. Java 7 moved the pool to the Heap so it grows dynamically and strings can be GC'd when no longer referenced. Java 8 removed PermGen entirely — it's now Metaspace. From Java 7 onwards the pool lives on heap. The practical implication: always use `.equals()` to compare string content, never `==`. `==` compares references and only works by coincidence when both strings come from the pool."

> *Likely follow-up: "What does String.intern() do and when would you use it?"*

#### Related Concepts

- **String.intern():** Adds the string to the pool (or returns the existing pool entry). Rarely used in modern code — mostly a performance optimisation when storing millions of the same string (e.g., parsing repeated field names from files).
- **Compact Strings (Java 9+):** `String` internally uses `byte[]` instead of `char[]`. Latin-1 strings use 1 byte per char instead of 2 → ~50% memory reduction for typical English text.
- **StringDeduplication (G1 GC):** `-XX:+UseStringDeduplication` — G1 can detect duplicate string objects on the heap and point them to the same backing char array. Works at GC time, transparent to the application.

---

## Q15. JVM Memory Tuning & Diagnostic Tools

### Concept
JVM memory flags control heap size and GC behaviour. Proper tuning prevents both OutOfMemoryError and excessive GC overhead.

### Key JVM Flags

```bash
# Heap size
-Xms512m              # Initial heap (start small, grows as needed)
-Xmx2g                # Maximum heap (never exceeds this)
                      # Rule: set -Xms = -Xmx in containers (avoids resize overhead)

# GC selection
-XX:+UseG1GC          # G1 (default Java 9+)
-XX:+UseZGC           # ZGC (Java 15+, ultra-low latency)
-XX:MaxGCPauseMillis=200  # G1 pause target (soft target, not guaranteed)

# Metaspace
-XX:MaxMetaspaceSize=256m  # Cap class metadata (unlimited by default)

# Container-aware (Docker/K8s)
-XX:MaxRAMPercentage=75.0  # Use 75% of container's memory limit for heap
                            # Better than hard-coding -Xmx in containers

# Diagnostic flags
-XX:+HeapDumpOnOutOfMemoryError    # Auto-dump heap on OOM
-XX:HeapDumpPath=/tmp/heapdump.hprof
-Xlog:gc*:file=/var/log/gc.log:time,level,tags  # GC logging (Java 9+)

# GC stats
-verbose:gc           # Basic GC output to stdout
```

### Diagnostic Tools

```bash
# jstack — thread dump (for deadlocks, StackOverflow diagnosis)
jstack <pid>
jstack <pid> | grep -A 10 "DEADLOCK"

# jmap — heap dump (for OutOfMemoryError diagnosis)
jmap -dump:format=b,file=heapdump.hprof <pid>
# Or trigger via jcmd:
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# jcmd — comprehensive JVM diagnostics
jcmd <pid> VM.flags                    # Show all JVM flags in effect
jcmd <pid> GC.run                      # Trigger GC manually
jcmd <pid> Thread.print                # Thread dump
jcmd <pid> VM.native_memory            # Native memory tracking

# jstat — live GC statistics
jstat -gcutil <pid> 1000               # GC summary every 1 second
# Output: S0 S1 E O M CCS YGC YGCT FGC FGCT GCT
# E=Eden%, O=Old%, FGC=Full GC count

# VisualVM / JProfiler / Async-profiler
# - Visual heap dump analysis
# - Object retention tree (who is keeping what alive)
# - CPU profiling, allocation profiling

# Spring Boot Actuator (production-friendly)
GET /actuator/metrics/jvm.memory.used
GET /actuator/metrics/jvm.gc.pause
GET /actuator/metrics/jvm.memory.max
```

### Practical Tuning Rules

```bash
# Rule 1: In containers, ALWAYS set -XX:MaxRAMPercentage instead of -Xmx
# Bad: -Xmx1g in a 4GB container — wastes 3GB
# Good: -XX:MaxRAMPercentage=75 → uses ~3GB automatically

# Rule 2: Leave headroom for non-heap memory
# Non-heap includes: Metaspace, thread stacks, Code Cache, DirectByteBuffer
# General: set -Xmx to ~70-75% of total container/machine RAM

# Rule 3: Monitor before tuning
# Enable GC logging in production, measure actual pause times and frequency
# before changing GC settings

# Rule 4: -Xms = -Xmx for containerised apps
# Avoids heap resize overhead (resize = Full GC)

# Rule 5: Size thread pool based on threads
# Each thread = stack memory (-Xss, default ~256KB-1MB)
# 1000 threads × 512KB = 500MB just for stacks
```

### Interview Answer

> "Key flags: `-Xms` for initial heap, `-Xmx` for max heap, `-XX:MaxRAMPercentage` for containers (avoids hardcoding). I always set `-XX:+HeapDumpOnOutOfMemoryError` in production — you want a heap dump or diagnosing OOM is nearly impossible. For diagnostics: `jstat -gcutil` for live GC stats, `jmap` or `jcmd` for heap dumps, `jstack` for thread dumps and deadlock detection. In Spring Boot I expose `/actuator/metrics/jvm.gc.pause` to Grafana for continuous GC monitoring. The key tuning insight is to leave 20-30% RAM headroom for non-heap: thread stacks, Metaspace, Code Cache, and DirectByteBuffers."

> *Likely follow-up: "How do you diagnose a memory leak in production?"*

#### Related Concepts

- **Memory leak diagnosis flow:** 1. Get heap dump (jmap or OOM auto-dump). 2. Open in VisualVM/MAT. 3. Find "dominator tree" — objects retaining the most memory. 4. Trace who is holding the reference. 5. Common culprits: static collections, ThreadLocal not cleaned, listeners not deregistered, Hibernate first-level cache in long transactions.
- **Off-heap memory (DirectByteBuffer):** Not counted in `-Xmx`. Used by NIO, Netty, Kafka clients. Can cause OOM that doesn't show in heap dump. Monitor with `jcmd <pid> VM.native_memory`.
- **G1 GC tuning:** Don't micro-tune G1 — it's self-tuning. Set `MaxGCPauseMillis`, give it enough heap, let it work. Only tune if you have consistent Full GC events.

---

## Q16. Memory Management: 4 Apps × 2GB on a 2GB/4GB Server

### Concept
JVM heap allocation is lazy — `-Xmx` sets a ceiling, not an immediate reservation. The OS manages physical memory with virtual memory and swap.

### On a 2GB RAM Server (4 apps × 2GB max heap)

```
Total RAM: 2 GB
4 apps × -Xmx2g each = 8 GB virtual address space needed

Reality:
- JVM with -Xmx2g doesn't consume 2GB immediately
- Starts at -Xms (e.g., 256MB) and grows under load
- OS uses virtual memory — maps pages to physical RAM on demand
- If all 4 apps peak simultaneously → brutal swap I/O → system nearly unusable
- Likely result: Linux OOM Killer terminates processes

Rule of thumb: total -Xmx across all apps should not exceed (RAM - OS overhead)
OS typically needs: 500MB-1GB
Safe formula: sum of all -Xmx ≤ TotalRAM × 0.85
```

### On a 4GB RAM Server (more realistic)

```bash
# If apps don't all peak at the same time:
# App 1: 1.0 GB active
# App 2: 800 MB active
# App 3: 1.2 GB active
# App 4: 600 MB active
# Total active: 3.6 GB → fits with 4GB minus OS overhead (≈400MB)

# Best practice configuration:
java -Xms256m -Xmx1500m -XX:MaxMetaspaceSize=256m MyApp
# -Xms small: don't reserve more than needed at startup
# -Xmx 1.5GB: gives headroom (4 × 1.5 = 6GB max, won't all hit at once)
# MaxMetaspace: cap class metadata growth

# In containers (Docker/K8s) — MUCH BETTER to use:
docker run -m 2g my-app java -XX:MaxRAMPercentage=75 -jar app.jar
# JVM sees container limit (2GB), uses 75% = 1.5GB for heap
# Remaining 25% (500MB) for OS + non-heap JVM memory

# Monitor with Spring Actuator:
# /actuator/metrics/jvm.memory.used  → current heap usage
# /actuator/metrics/jvm.memory.max   → configured max
```

### Interview Answer

> "You need to separate the max heap ceiling from actual active usage. JVM with `-Xmx2g` starts small and grows under load — it doesn't immediately consume 2GB. On a 2GB server with 4 such apps, if they all peak simultaneously it's infeasible — the system will thrash on swap and likely OOM-kill processes. On a 4GB server if they don't all peak together, they can coexist. In production I set `-Xmx` conservatively — typically 70-75% of available RAM split across apps — and use `-XX:MaxRAMPercentage` in containers so the JVM automatically respects Docker memory limits."

> *Likely follow-up: "What is the Linux OOM Killer and how does it affect Java apps?"*

#### Related Concepts

- **Linux OOM Killer:** When RAM + swap is exhausted, Linux kills the process with the highest "OOM score" (approximated by memory usage). Java processes are common victims. Mitigate with: `vm.overcommit_memory=2` or request-based limits.
- **JVM off-heap memory footprint:** A JVM process uses more than just the heap. Add: Metaspace (~256MB), thread stacks (N threads × 256KB), Code Cache (~240MB), DirectByteBuffers. A 2GB heap process can consume 2.5-3GB total.
- **`-XX:MaxRAMPercentage` in K8s:** If the K8s pod memory limit is 2Gi and you set `MaxRAMPercentage=75`, heap = 1.5Gi. Without this, the JVM sees the NODE's full memory (e.g., 32GB) and miscalculates, leading to OOM kills from the kubelet.

---

## Summary: Key Topics at a Glance

| Topic | Questions | Key Points |
|---|---|---|
| **Serialization** | Q1-Q4 | `serialVersionUID`, `transient`, `readResolve()` |
| **Modern Java** | Q5-Q7 | Try-with-resources, `java.time`, Records, Sealed classes |
| **JVM Architecture** | Q8-Q9 | ClassLoader, Runtime Areas, Execution Engine, JDK/JRE layers |
| **Memory Model** | Q10-Q11 | Stack vs Heap, SOE vs OOM, GC roots, memory layout |
| **Garbage Collection** | Q12-Q15 | Minor/Full GC, G1/ZGC, String pool location, JVM tuning |
| **Memory Capacity** | Q16 | -Xmx ceiling, virtual memory, container memory limits |

---

**End of Topic 4**
