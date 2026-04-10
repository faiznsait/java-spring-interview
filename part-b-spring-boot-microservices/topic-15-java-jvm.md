# Topic 15: Java JVM and Runtime

## Q82. Dynamic Method Dispatch

### Concept
Runtime polymorphism — JVM decides which method to invoke at runtime based on the **actual object type**, not the reference type.

### Simple Explanation
A remote control (reference) can point at TV or AC (actual objects). Press "power" → TV turns on TV-specific behavior, AC turns on AC-specific behavior. Same button, different behavior decided at runtime.

### How It Works
```
1. Compile time: Compiler checks if method exists in Reference Type
2. Runtime: JVM looks at Actual Object Type and invokes ITS version
```

### Code Example
```java
class Animal {
    void sound() { System.out.println("Generic animal sound"); }
}

class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }
}

class Cat extends Animal {
    @Override
    void sound() { System.out.println("Meow"); }
}

public class Test {
    public static void main(String[] args) {
        Animal ref;                      // Reference type = Animal
        
        ref = new Dog();                 // Actual object = Dog
        ref.sound();                     // Bark — invokes Dog's sound()
        
        ref = new Cat();                 // Actual object = Cat
        ref.sound();                     // Meow — invokes Cat's sound()
        
        // Decision made at RUNTIME, not compile time
    }
}
```

### Real Use Case
```java
public class PaymentProcessor {
    public void process(PaymentMethod method, BigDecimal amount) {
        method.charge(amount);  // Runtime decides: CreditCard.charge() or UPI.charge()
    }
}

// Same code handles 10 payment types — extensible without changing processor
```

### Interview Answer
> "Dynamic method dispatch is runtime polymorphism. The JVM decides which method to invoke based on the actual object type at runtime, not the reference type at compile time. It's the foundation of the Strategy pattern — you pass a `PaymentMethod` reference, but at runtime it could be `CreditCard`, `UPI`, or `Wallet`, each with its own `charge()` implementation. This makes code extensible without modification — add new payment types without changing `PaymentProcessor`.
>
> **Follow-up:** *What happens with private/static/final methods?*
> — They don't participate in dynamic dispatch. Private = not inherited. Static = resolved at compile time (static binding). Final = can't be overridden."

---

## Q83. Serialization — Deep Dive

### Concept
Converting an object into a **byte stream** so it can be saved to a file, sent over a network, or stored in a database. Deserialization = reverse.

### Simple Explanation
Packing a laptop for shipping: disassemble into parts, pack in a box (serialization). At destination: unpack and reassemble (deserialization).

### Key Components

| Component | Purpose |
|---|---|
| `Serializable` interface | Marker — signals "this class can be serialized" |
| `serialVersionUID` | Version control — ensures sender and receiver have compatible class versions |
| `transient` | Skip this field during serialization |
| `writeObject()` / `readObject()` | Custom serialization logic |

### Basic Code
```java
@Data
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // ✅ Always declare explicitly
    
    private Long id;
    private String name;
    private String email;
    
    private transient String password;  // ❌ Never serialize passwords
    private transient int loginAttempts; // Runtime state — no need to persist
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
// V1: Sent over network
class User implements Serializable {
    String name;
}
// Receiver has V2 with new field
class User implements Serializable {
    String name;
    String email;  // new field added
}
// Result: InvalidClassException — auto-generated UIDs don't match
```

```java
// Scenario 2: serialVersionUID DECLARED
class User implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
}
// V2 with same UID
class User implements Serializable {
    private static final long serialVersionUID = 1L;  // SAME UID
    String name;
    String email;  // new field — backward compatible
}
// Result: Deserialization succeeds. email = null in deserialized object
```

**Rule:** Always declare `serialVersionUID = 1L`. Increment to `2L` only when making **incompatible** changes (removing field, changing type).

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
    
    private String encrypt(String raw) { return Base64.getEncoder().encodeToString(raw.getBytes()); }
    private String decrypt(String enc) { return new String(Base64.getDecoder().decode(enc)); }
}
```

### Most Asked Checkpoints
- ✅ Always implement `Serializable` — even for nested objects
- ✅ Always declare `serialVersionUID` explicitly
- ✅ Use `transient` for passwords, temporary state, derived fields
- ✅ Parent class must be `Serializable` for child class serialization to include parent fields
- ❌ `static` fields are NOT serialized (they belong to class, not instance)
- ❌ `transient` fields get default values (null/0/false) after deserialization

### Interview Answer
> "Serialization converts an object to a byte stream for network transfer or persistence. I always implement `Serializable` and declare `serialVersionUID = 1L` explicitly — this prevents `InvalidClassException` when class evolves. Use `transient` for passwords and runtime-only state.
>
> For sensitive data, I override `writeObject()` and `readObject()` — encrypt before serialization, decrypt after. For example, password is encrypted before writing and decrypted during deserialization.
>
> Common pitfall: forgetting that parent class must also be `Serializable` if you want parent fields serialized. Static fields are never serialized — they belong to the class, not instances."

---

## Q84. `transient` Keyword

### What It Is
A **Java language keyword** (not framework-specific) that excludes a field from serialization.

### When to Use
```java
public class Session implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String userId;
    private Instant createdAt;
    
    // Runtime-only state — don't persist
    private transient int requestCount;           // reset to 0 on deserialization
    private transient Map<String, Object> cache;   // rebuild on app restart
    
    // Security — never serialize
    private transient String oauthToken;
    private transient String password;
    
    // Derived fields — recompute after deserialization
    private transient boolean expired;
}
```

### After Deserialization
```java
// Before serialization
requestCount = 42
cache = {key1=val1, key2=val2}
password = "secret"

// After deserialization
requestCount = 0          // default int value
cache = null              // default object value
password = null           // you must handle this in readObject()
```

### Interview Answer
> "`transient` is a Java language keyword, not Spring-specific. It tells the serialization mechanism to skip that field. Use it for passwords, OAuth tokens, runtime counters, and derived fields. After deserialization, `transient` fields get default values — null for objects, 0 for numbers. If you need them populated, override `readObject()` and recompute them manually."

---

## Q85. Lombok with Serialization

### The Issue
Lombok generates constructors, getters, setters. If you're using `@Data` or `@AllArgsConstructor`, it may interfere with the default deserialization process if not careful.

### Best Practice
```java
import lombok.Data;
import java.io.Serializable;

@Data  // ✅ Works fine with Serializable
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // ⚠️ MUST be explicit, Lombok won't generate
    
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

## Q86. Singleton Breaking During Serialization

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

## Q87. Try-With-Resources

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

## Q88. Date-Time API (`java.time`)

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
| `LocalDateTime` | Date + time (no timezone) | Log timestamp: `2026-03-13T14:30:00` |
| `ZonedDateTime` | Date + time + timezone | Flight time: `2026-03-13T14:30:00+05:30[Asia/Kolkata]` |
| `Instant` | Unix timestamp (UTC) | DB timestamp, server logs |
| `Duration` | Time-based (hours, minutes, seconds) | Video length: `PT2H30M` |
| `Period` | Date-based (years, months, days) | Age: `P5Y3M10D` |

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
Duration duration = Duration.between(time1, time2);   // PT5H30M (5 hours 30 min)
Period period = Period.between(birthDate, today);     // P26Y2M10D (26 years 2 months 10 days)

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

## Q89. Sealed Classes and Records

### Sealed Classes (Java 17+)

### Concept
Restrict which classes can extend/implement a type. All permitted subclasses must be declared explicitly.

### Simple Explanation
VIP club: only specific pre-approved people can enter. No random person can join. The club (sealed class) explicitly lists who's allowed.

### Code
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

### Why Use Sealed Classes
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

### Concept
Immutable data carriers. Boilerplate-free: no need to write constructor, getters, `equals()`, `hashCode()`, `toString()`.

### Traditional Immutable Class
```java
// 30 lines for a simple data class
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

### Record — Same Thing in 1 Line
```java
public record User(Long id, String name, String email) {}

// Auto-generated:
// - Constructor: new User(1L, "John", "john@example.com")
// - Getters: user.id(), user.name(), user.email()  (no "get" prefix!)
// - equals(), hashCode(), toString()
// - All fields are private final (immutable)
```

### Records in Practice
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

### Records vs Traditional Immutable Classes

| Feature | Record | Traditional Class |
|---|---|---|
| Boilerplate | Zero | High (constructor, getters, equals, etc.) |
| Immutability | Enforced | Manual (`final` fields) |
| Inheritance | ❌ Cannot extend classes | ✅ Can extend |
| Use for | DTOs, value objects | Complex domain objects with behavior |
| Getter naming | `user.name()` | `user.getName()` |

### Interview Answer
> "Records eliminate boilerplate for immutable data carriers — DTOs, value objects. One line gives you constructor, getters (without 'get' prefix), equals, hashCode, toString. All fields are private final. I use them for API response DTOs and internal value objects.
>
> Records can't extend classes (but can implement interfaces). For complex domain entities with behavior, I stick with traditional classes or Lombok.
>
> Sealed classes restrict who can extend a type — useful for closed type hierarchies like payment methods. Combined with pattern matching, the compiler checks exhaustiveness — no default case needed in switch.
>
> **Follow-up:** *Can you use records with JPA entities?*
> — Not recommended. JPA requires mutable fields and a no-arg constructor. Records are immutable and don't support no-arg constructors naturally. Use records for read-only DTOs, regular classes for entities."

---

## Q90. JVM Architecture

### High-Level View
```
┌─────────────────────────────────────────────────────────┐
│                    JVM Architecture                     │
├─────────────────────────────────────────────────────────┤
│  1. ClassLoader Subsystem                               │
│     Loading → Linking → Initialization                  │
├─────────────────────────────────────────────────────────┤
│  2. Runtime Data Areas                                  │
│     ┌────────────┬──────────────────────────────────┐   │
│     │ Per-Thread │ Shared Across All Threads        │   │
│     ├────────────┼──────────────────────────────────┤   │
│     │ PC Register│ Heap (objects)                   │   │
│     │ JVM Stack  │ Method Area (class metadata)     │   │
│     │ Native     │ String Pool (inside Heap in 7+)  │   │
│     │ Method     │ Runtime Constant Pool            │   │
│     │ Stack      │                                  │   │
│     └────────────┴──────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│  3. Execution Engine                                    │
│     Interpreter + JIT Compiler + GC                     │
└─────────────────────────────────────────────────────────┘
```

### 1. ClassLoader Subsystem

**Three Phases:**
```
1. Loading
   - Bootstrap ClassLoader (loads core Java classes from rt.jar)
   - Extension ClassLoader (loads from ext directory)
   - Application ClassLoader (loads from classpath)

2. Linking
   - Verification: bytecode is valid
   - Preparation: allocate memory for static variables
   - Resolution: resolve symbolic references to actual references

3. Initialization
   - Execute static blocks
   - Initialize static variables
```

### 2. Runtime Data Areas

#### Per-Thread (Each Thread Has Its Own)
```java
// PC Register (Program Counter)
// Stores address of current instruction being executed

// JVM Stack
// Stores method frames (local variables, partial results, method calls)
public void methodA() {
    int x = 10;        // stored in Stack Frame for methodA
    methodB(x);
}
public void methodB(int y) {
    int z = y + 5;     // stored in Stack Frame for methodB
}
// Frame for methodB pushed on top of Frame for methodA
// When methodB returns, its frame is popped

// Native Method Stack
// For native methods (JNI calls to C/C++ code)
```

#### Shared Across All Threads
```java
// Heap
// ALL objects live here (new keyword)
User user = new User();  // User object allocated on Heap

// Method Area (also called Metaspace in Java 8+)
// Stores class-level data:
//   - Class structure (fields, methods, constructors)
//   - Method bytecode
//   - Runtime constant pool
//   - Static variables

// String Pool (inside Heap in Java 7+)
String s1 = "hello";     // stored in String Pool
String s2 = "hello";     // reuses same object from pool
s1 == s2;                // true (same reference)
```

### 3. Execution Engine

```
Interpreter
  - Reads bytecode line by line
  - Slow for repeated code

JIT Compiler (Just-In-Time)
  - Identifies "hot" methods (frequently executed)
  - Compiles bytecode → native machine code
  - Caches compiled code for reuse
  - Result: near-native performance for hot paths

Garbage Collector
  - Identifies unreferenced objects
  - Reclaims memory
  - Types: Serial, Parallel, G1, ZGC
```

### Interview Answer
> "JVM has three main components: ClassLoader loads and initializes classes in three phases (loading, linking, initialization). Runtime Data Areas include per-thread (Stack, PC Register) and shared areas (Heap for objects, Method Area for class metadata). Execution Engine has an Interpreter for initial execution, JIT Compiler that converts hot bytecode to native code for performance, and Garbage Collector for automatic memory management.
>
> **Follow-up:** *Where do local variables and objects live?*
> — Local variables (primitives and references) live on the Stack in method frames. Actual objects always live on the Heap. When method returns, its stack frame is popped — local variables gone. But if the object is still referenced elsewhere, it stays on Heap until GC collects it."

---

## Q91. JDK vs JRE vs JVM

### Simple Explanation
```
JVM = Engine of a car (runs the bytecode)
JRE = Car with engine + fuel + tires (runtime environment to run apps)
JDK = Car factory (has tools to build + run apps)
```

### Detailed Breakdown

| Component | What It Contains | Who Needs It |
|---|---|---|
| **JVM** | Execution engine, GC, runtime data areas | Everyone (inside JRE) |
| **JRE** | JVM + Standard libraries (java.lang, java.util, etc.) | End users running apps |
| **JDK** | JRE + Development tools (javac, jar, javadoc, debugger) | Developers |

### Visual Hierarchy
```
┌───────────────────────────────────────────────┐
│                    JDK                        │
│  ┌─────────────────────────────────────────┐  │
│  │              JRE                        │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │            JVM                     │ │  │
│  │  │  - Interpreter                    │ │  │
│  │  │  - JIT Compiler                   │ │  │
│  │  │  - Garbage Collector              │ │  │
│  │  └────────────────────────────────────┘ │  │
│  │                                         │  │
│  │  - Core Libraries (java.lang, etc.)    │  │
│  │  - java command                        │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  - javac (compiler)                           │
│  - jar (packager)                             │
│  - javadoc (documentation generator)          │
│  - jdb (debugger)                             │
└───────────────────────────────────────────────┘
```

### Practical Examples

#### End User (Only Needs JRE)
```bash
# User wants to run a Java application (SomeApp.jar)
java -jar SomeApp.jar

# Requirements: JRE only
# JRE has:
#   - JVM to execute bytecode
#   - Standard libraries (java.lang.*, java.util.*, etc.)
```

#### Developer (Needs JDK)
```bash
# 1. Write code
# HelloWorld.java

# 2. Compile (needs javac from JDK)
javac HelloWorld.java     # produces HelloWorld.class (bytecode)

# 3. Run (uses JRE inside JDK)
java HelloWorld

# 4. Package (needs jar from JDK)
jar cf app.jar *.class

# 5. Generate docs (needs javadoc from JDK)
javadoc HelloWorld.java
```

### Which JVM Implementation?
JVM is a **specification**. Multiple implementations exist:
- **HotSpot** (Oracle/OpenJDK) — most common
- **OpenJ9** (Eclipse) — lower memory footprint
- **GraalVM** — ahead-of-time compilation, polyglot
- **Azul Zing** — ultra-low-latency GC

### Interview Answer
> "JVM is the execution engine — runs bytecode, manages memory, does garbage collection. JRE is JVM plus standard libraries — everything needed to run Java apps. JDK is JRE plus development tools — compiler (javac), packager (jar), debugger (jdb). End users need JRE. Developers need JDK. In production Docker images, we install JRE only to reduce image size — no need for javac in runtime."

---

## Q92. Heap vs Stack

### Simple Explanation
**Stack:** Office desk — temporary work area. Current task's papers (local variables). Task done → clear desk. Fast, small, organized.

**Heap:** Office warehouse — long-term storage. Big items (objects) stored here. Takes time to find things. Cleaned periodically (GC).

### Side-by-Side Comparison

| | Stack | Heap |
|---|---|---|
| **Stores** | Local variables, method frames | Objects (all instances created with `new`) |
| **Scope** | Per-thread (each thread has own Stack) | Shared across all threads |
| **Lifetime** | Method exits → frame popped → variables gone | Until no references exist → GC collects |
| **Size** | Small (default ~1MB per thread) | Large (default 256MB - 4GB depending on JVM) |
| **Speed** | Very fast (LIFO, simple pointer manipulation) | Slower (complex allocation, fragmentation) |
| **Errors** | StackOverflowError (too deep recursion) | OutOfMemoryError (too many objects) |
| **Management** | Automatic (push/pop) | Automatic (GC) |

### Code Example
```java
public class Test {
    public static void main(String[] args) {
        int x = 10;               // ← Stack: primitive local variable
        User user = new User();   // ← Stack: reference variable 'user'
                                  // ← Heap: actual User object
        processUser(user);
    }
    
    public static void processUser(User u) {
        String name = u.getName();  // ← Stack: reference 'name'
                                     // ← Heap: String object (if not in String pool)
        System.out.println(name);
    }
}

// Visualization:
// STACK (Thread-1)              HEAP (Shared)
// ──────────────                ─────────────
// main() frame:                 User object @0x100
//   x = 10                        - id = 1
//   user = @0x100 ───────────────►- name = "John"
//                                   
// processUser() frame:          String object @0x200
//   u = @0x100 ──────────────────►  "John"
//   name = @0x200 ───────────────►
```

### What Happens When Heap is Full?

```
1. Heap fills up (too many objects)
2. GC tries to collect unreferenced objects
3. If GC reclaims enough → program continues
4. If GC can't reclaim enough → throws OutOfMemoryError
5. Application crashes (or catch and handle gracefully)
```

```java
// Simulate OutOfMemoryError
List<byte[]> list = new ArrayList<>();
while (true) {
    list.add(new byte[1024 * 1024]);  // 1MB object
    // Eventually: java.lang.OutOfMemoryError: Java heap space
}

// Fix: increase heap size
// java -Xmx4g MyApp  (max 4GB heap)
```

### Interview Answer
> "Stack stores local variables and method frames — one per thread. When a method is called, a frame is pushed; when it returns, the frame is popped. Stack is fast but small — causes StackOverflowError if recursion is too deep.
>
> Heap stores all objects — shared across threads. It's large but slower. When heap fills up, GC runs to reclaim unreferenced objects. If GC can't free enough space, you get OutOfMemoryError.
>
> Key: primitives and references live on Stack if they're local variables. Actual objects always live on Heap. When a method exits, its Stack frame is gone, but if the object is still referenced elsewhere, it stays on Heap until GC."

---

## Q93. StackOverflowError vs OutOfMemoryError

### StackOverflowError

**Cause:** Too many method calls without returning (usually infinite/deep recursion). Each call pushes a frame on the Stack. Eventually Stack runs out of space.

```java
// Infinite recursion
public void recurse() {
    recurse();  // calls itself forever
}
// Result: java.lang.StackOverflowError
```

**Common Scenarios:**
```java
// 1. Infinite recursion (forgot base case)
public int factorial(int n) {
    return n * factorial(n - 1);  // ❌ no base case
}

// 2. Circular method calls
public void methodA() { methodB(); }
public void methodB() { methodA(); }  // ❌ infinite loop

// 3. Very deep recursion (even with base case)
factorial(100000);  // too deep for default stack size
```

**Diagnostics:**
```
Stack trace shows repeated method calls
Example:
  at com.app.Service.recurse(Service.java:10)
  at com.app.Service.recurse(Service.java:10)
  at com.app.Service.recurse(Service.java:10)
  ... (repeats 1000+ times)
```

**Fixes:**
```java
// 1. Add base case
public int factorial(int n) {
    if (n <= 1) return 1;  // ✅ base case
    return n * factorial(n - 1);
}

// 2. Convert to iteration
public int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}

// 3. Increase stack size (temporary workaround)
// java -Xss2m MyApp  (2MB stack per thread, default is ~1MB)
```

---

### OutOfMemoryError

**Cause:** Heap runs out of space — too many objects and GC can't reclaim enough.

**Common Scenarios:**

#### 1. Memory Leak
```java
// Classic leak: objects held in memory unintentionally
public class CacheService {
    private static Map<String, byte[]> cache = new HashMap<>();
    
    public void cache(String key, byte[] data) {
        cache.put(key, data);  // ❌ Never removed — grows forever
    }
}
// Fix: use WeakHashMap or eviction policy (Guava Cache, Caffeine)
```

#### 2. Large Collections
```java
// Loading entire DB table into memory
List<User> users = userRepo.findAll();  // ❌ 10 million users → OOM
// Fix: pagination or streaming
userRepo.findAll(PageRequest.of(0, 1000));
```

#### 3. Heap Too Small
```bash
# Default heap may be too small for your app
# Current: -Xmx256m (256MB max heap)
# Fix: increase
java -Xmx4g -jar app.jar  # 4GB max heap
```

**Diagnostic Tools:**

| Tool | Purpose |
|---|---|
| `jmap -heap <pid>` | View current heap usage |
| `jmap -dump:live,format=b,file=heap.bin <pid>` | Dump heap to file |
| **Eclipse MAT** / **VisualVM** | Analyze heap dump — find memory leaks |
| `jstat -gc <pid> 1000` | Monitor GC activity every 1s |
| `-XX:+HeapDumpOnOutOfMemoryError` | Auto-dump heap when OOM occurs |

**Heap Dump Analysis Flow:**
```
1. App crashes with OutOfMemoryError
2. Heap dump auto-created (if flag enabled)
3. Open dump in Eclipse MAT
4. MAT shows:
   - Leak suspects (objects consuming most memory)
   - Dominator tree (which object is holding references)
   - Shortest path to GC root (why object isn't collected)
5. Fix: remove reference / add eviction policy / increase heap
```

---

### Interview Comparison

| | StackOverflowError | OutOfMemoryError |
|---|---|---|
| **Memory Area** | Stack (per-thread) | Heap (shared) |
| **Common Cause** | Infinite/deep recursion | Memory leak, large collections, heap too small |
| **Stack Trace** | Repeating method calls | Various — depends on where allocation fails |
| **Quick Fix** | Add base case, convert to iteration | Increase heap (`-Xmx`), fix memory leak |
| **Diagnostic** | Stack trace shows recursion | Heap dump + MAT analysis |

### Interview Answer
> "StackOverflowError happens when Stack runs out of space — usually infinite or very deep recursion. Each method call pushes a frame; too many pushes → error. Fix by adding a base case or converting recursion to iteration. Can also increase stack size with `-Xss` but that's a workaround, not a solution.
>
> OutOfMemoryError means Heap is full and GC can't reclaim enough memory. Common causes: memory leaks (objects never released), loading huge collections (entire DB table), or heap size too small for the workload. I diagnose with heap dumps — enable `-XX:+HeapDumpOnOutOfMemoryError`, analyze with Eclipse MAT to find leak suspects. Fix by removing references, adding cache eviction policies, or increasing heap with `-Xmx`.
>
> **Follow-up:** *How do you prevent memory leaks?*
> — Use weak references for caches (`WeakHashMap`), close resources with try-with-resources, avoid static collections that grow unbounded, use cache libraries with eviction (Caffeine, Guava), and periodically profile with VisualVM in staging to catch leaks before production."

---

## Q94. Garbage Collectors & Mark-and-Sweep

### Garbage Collection — What It Solves
In C/C++, you manually allocate (`malloc`) and free (`free`) memory. Forget to free → memory leak. Free twice → crash. Java automates this with GC.

### Mark-and-Sweep Algorithm (Foundation of Most GCs)

```
Phase 1: MARK
  Start from "GC Roots" (active Stack references, static variables, JNI references)
  Traverse object graph, mark all reachable objects

Phase 2: SWEEP
  Scan entire Heap
  Unreachable objects (not marked) → reclaim their memory
  Marked objects → unmark for next GC cycle
```

```
Before GC:
  Stack:  ref1 → Object A → Object B
                            
  Heap:   Object A (reachable)
          Object B (reachable)
          Object C (unreachable — no references)
          Object D (unreachable)

After MARK phase:
  A ✓ marked
  B ✓ marked
  C ✗ not marked
  D ✗ not marked

After SWEEP phase:
  A and B remain
  C and D memory reclaimed
```

### Types of Garbage Collectors

#### 1. Serial GC (`-XX:+UseSerialGC`)
```
Single-threaded
Freezes all application threads during GC ("Stop-The-World")
Use for: Small apps, single-core machines, client apps
❌ Not for production servers
```

#### 2. Parallel GC (`-XX:+UseParallelGC`)
```
Multiple GC threads (parallel marking/sweeping)
Still STW, but faster than Serial
Use for: Batch processing, throughput-critical apps
Default in Java 8
Trade-off: High throughput, but long pause times
```

#### 3. G1 GC (`-XX:+UseG1GC`)
```
Divides Heap into regions (~2000 regions of 1-32MB each)
Collects regions with most garbage first (Garbage-First)
Predictable pause times (target: < 200ms)
Use for: Default choice for most production apps
Default in Java 9+
Mix of young + old generation collection
```

#### 4. ZGC (`-XX:+UseZGC`)
```
Ultra-low-latency GC (pause times < 10ms even with 4TB heaps)
Concurrent — most work done without stopping app threads
Use for: Latency-sensitive apps (trading systems, real-time analytics)
Available: Java 11+ (production-ready in Java 15+)
```

#### 5. Shenandoah (`-XX:+UseShenandoahGC`)
```
Similar to ZGC — low pause times
Concurrent compacting GC
Use for: Low-latency apps
Available: OpenJDK 12+
```

### GC Generations (G1, Parallel)

```
YOUNG GENERATION (short-lived objects)
  Eden Space: new objects allocated here
  Survivor Spaces (S0, S1): objects that survived one GC

OLD GENERATION (long-lived objects)
  Objects that survived multiple GCs promoted here

┌──────────────────────────┬─────────────────────────┐
│   Young Generation       │   Old Generation        │
│  ┌──────┬─────┬─────┐    │                         │
│  │ Eden │ S0  │ S1  │    │   Long-lived objects    │
│  └──────┴─────┴─────┘    │                         │
└──────────────────────────┴─────────────────────────┘

Minor GC: cleans Young (frequent, fast)
Major GC: cleans Old (infrequent, slower)
Full GC: cleans entire Heap (rare, longest pause)
```

### Real Configuration Example
```yaml
# Spring Boot application.properties
# Using G1 GC with tuning
java.opts: >
  -Xms2g -Xmx4g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/logs/heapdumps
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -Xloggc:/var/logs/gc.log
```

### Monitoring GC
```bash
# View GC activity in real-time
jstat -gc <pid> 1000

# Output:
# S0C    S1C    EC       EU       OC       OU
# 1024   1024   8192     4532     20480    15234
# (Eden, Survivor, Old capacities and usage)

# Analyze GC logs
# Tools: GCViewer, GCEasy (online tool)
```

### Interview Answer
> "Garbage collection automates memory management. The Mark-and-Sweep algorithm marks reachable objects starting from GC roots, then sweeps the heap to reclaim unmarked objects.
>
> For production, I use **G1 GC** (default in Java 9+) — predictable pause times under 200ms, handles large heaps well, balances throughput and latency. For ultra-low-latency requirements (trading systems), **ZGC** keeps pause times under 10ms even with multi-terabyte heaps.
>
> I monitor with `jstat` and analyze GC logs with GCEasy. If Full GCs are frequent, it signals a memory leak or undersized heap — I'd take a heap dump and analyze with Eclipse MAT.
>
> **Follow-up:** *What causes long GC pauses?*
> — Large heap with Parallel GC (long STW). Too many live objects during Old Gen collection. Memory leaks causing frequent Full GCs. Solution: switch to G1/ZGC, tune GC params (`-XX:MaxGCPauseMillis`), fix memory leaks, increase heap size if undersized."

---

## Q95. String Pool Location Across JDK Versions

### The Evolution

| JDK Version | String Pool Location | Why | Implications |
|---|---|---|---|
| **JDK 6 and earlier** | PermGen (Permanent Generation) | Part of Method Area | Fixed size, never GC'd → easy OOM |
| **JDK 7** | **Moved to Heap** | Allow GC of unused strings | Can be collected, dynamic sizing |
| **JDK 8** | Heap (PermGen removed, replaced with Metaspace) | Metaspace for class metadata only | String pool remains in Heap |
| **JDK 11+** | Heap | Consistent with 7+ | No change |

### Why the Move to Heap (JDK 7+)?

```
OLD (JDK 6): String Pool in PermGen
  Problem 1: PermGen has fixed size (-XX:MaxPermSize)
  Problem 2: Strings never garbage collected
  Problem 3: Lots of unique strings → PermGen OOM

NEW (JDK 7+): String Pool in Heap
  ✅ Dynamic sizing (grows with heap)
  ✅ Unused strings can be GC'd
  ✅ No separate tuning needed
```

### Where Does Heap Memory Reside?
Heap is part of **JVM process memory** in **RAM**. It's not a separate file on disk (unless you dump it with `jmap`).

```
Physical RAM
  ↓
Operating System
  ↓
JVM Process
  ├── Heap (objects, String pool)
  ├── Metaspace (class metadata)
  ├── Stack (per thread)
  ├── Native memory (ThreadStacks, DirectByteBuffers, etc.)
  └── Code cache (JIT compiled code)
```

### String Pool Behavior (All JDK Versions)
```java
// Literal → goes to String Pool
String s1 = "hello";
String s2 = "hello";
s1 == s2;  // true (same object from pool)

// new keyword → bypasses pool, creates new object on Heap
String s3 = new String("hello");
String s4 = new String("hello");
s3 == s4;  // false (two different objects)
s3 == s1;  // false

// intern() → moves to pool (or returns existing)
String s5 = new String("hello").intern();
s5 == s1;  // true (intern() found "hello" in pool and returned it)
```

### Interview Answer
> "In JDK 6, String pool was in PermGen — fixed size, strings never GC'd, easy to run out of space. JDK 7 moved it to the Heap — dynamic sizing, unused strings can be garbage collected, no separate tuning. JDK 8 removed PermGen entirely, replaced with Metaspace for class metadata, but String pool remained in Heap.
>
> Heap memory resides in RAM as part of the JVM process. String literals go to the pool and are reused. `new String()` bypasses the pool — creates a new object on Heap. `intern()` adds to pool or returns existing.
>
> **Follow-up:** *What is Metaspace?*
> — Metaspace replaced PermGen in JDK 8. It stores class metadata (structure, methods, fields). Unlike PermGen (fixed size), Metaspace grows dynamically using native memory — max size controlled by `-XX:MaxMetaspaceSize`. Reduces PermGen OOM issues from hot deployments and dynamic class loading."

---

## Q96. Heap Memory Tuning & Diagnostic Tools

### JVM Memory Flags

| Flag | Purpose | Example |
|---|---|---|
| `-Xms` | Initial heap size | `-Xms512m` (start with 512MB) |
| `-Xmx` | Maximum heap size | `-Xmx4g` (max 4GB) |
| `-Xss` | Stack size per thread | `-Xss2m` (2MB per thread) |
| `-XX:MaxMetaspaceSize` | Max Metaspace (class metadata) | `-XX:MaxMetaspaceSize=256m` |
| `-XX:MaxDirectMemorySize` | Max off-heap memory (NIO) | `-XX:MaxDirectMemorySize=1g` |

### Best Practices
```bash
# Production Spring Boot app
java -Xms2g -Xmx2g \           # Same min/max → avoids resizing overhead
     -Xss512k \                 # 512KB per thread (default is ~1MB)
     -XX:MaxMetaspaceSize=256m \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/logs/heapdump.hprof \
     -jar app.jar
```

### Why Set `-Xms` = `-Xmx`?
```
Different values:
  -Xms512m -Xmx4g
  
  Start: 512MB
  As heap fills → JVM requests more memory from OS (resize)
  Resizing = expensive operation, causes pauses

Same values:
  -Xms2g -Xmx2g
  
  JVM allocates full 2GB at startup
  No runtime resizing → no resize pauses
  More predictable performance
```

### Diagnostic Tools

#### 1. `jps` — List Java Processes
```bash
jps -l
# Output:
# 12345 com.example.MyApp
# 12346 org.springframework.boot.loader.JarLauncher
```

#### 2. `jstat` — Monitor GC and Heap
```bash
jstat -gc 12345 1000     # GC stats every 1 second
jstat -gcutil 12345      # GC percentage utilization
```

#### 3. `jmap` — Heap Dump & Memory Info
```bash
# View heap summary
jmap -heap 12345

# Dump heap to file
jmap -dump:live,format=b,file=heap.hprof 12345

# Histogram of object counts
jmap -histo 12345 | head -20
```

#### 4. `jstack` — Thread Dump
```bash
jstack 12345 > threads.txt
# Analyze: deadlocks, blocked threads, thread states
```

#### 5. **VisualVM** (GUI)
```
Download: https://visualvm.github.io
Features:
  - Real-time CPU, memory, thread monitoring
  - Heap dump analysis
  - Profiler (method-level CPU/memory usage)
  - Visual GC plugin
```

#### 6. **Eclipse MAT** (Memory Analyzer Tool)
```
Use for: Analyzing heap dumps
Download: https://www.eclipse.org/mat/

Open heap.hprof → MAT shows:
  - Leak suspects (automatic analysis)
  - Dominator tree (what's holding memory)
  - Histogram (object counts)
  - OQL (Object Query Language) for custom queries
```

#### 7. **JMeter** (Load Testing)
```
Not for memory analysis — for simulating load
Use JMeter to generate traffic → observe memory/GC under load
Combined with VisualVM/MAT for profiling
```

#### 8. **Profilers** (Production-Grade)
- **YourKit** — commercial, low overhead
- **JProfiler** — commercial, comprehensive
- **Async-profiler** — open-source, low overhead, flame graphs

### Real Diagnostic Flow
```
Symptom: App slowing down in production

1. Check GC activity
   jstat -gc <pid> 1000
   → Frequent Full GCs → memory pressure

2. Take heap dump
   jmap -dump:live,format=b,file=heap.hprof <pid>

3. Analyze in MAT
   → Leak suspect: CacheService holding 3GB of User objects
   → Dominator tree shows static Map never evicted

4. Fix code
   Replace HashMap with Caffeine cache with TTL eviction

5. Re-test in staging
   Load test with JMeter + monitor with VisualVM
   → Heap stable, no Full GCs, memory usage plateaus

6. Deploy to prod
```

### Interview Answer
> "For heap tuning, I set `-Xms` and `-Xmx` to the same value — avoids expensive runtime resizing. Typical production Spring Boot app: `-Xms2g -Xmx2g` for medium load, `-Xms4g -Xmx4g` for high traffic. Set `-XX:MaxMetaspaceSize=256m` to prevent Metaspace growing unbounded from dynamic class loading.
>
> For diagnostics: `jstat -gc` to monitor GC in real-time, `jmap -dump` to capture heap snapshot, Eclipse MAT to analyze heap dumps and find memory leaks. VisualVM for live profiling in staging — shows method-level CPU and memory hotspots.
>
> Always enable `-XX:+HeapDumpOnOutOfMemoryError` in production — auto-dumps heap when OOM happens so I can analyze the root cause post-mortem. I use JMeter for load testing and async-profiler for production profiling with minimal overhead.
>
> **Follow-up:** *How do you decide heap size?*
> — Start with app baseline (no load) plus expected data volume. Load test in staging, monitor peak heap usage with VisualVM. Set `-Xmx` to 1.5-2× peak usage for headroom. Monitor GC frequency — if frequent Full GCs, increase heap. If most heap unused, decrease to free resources for other apps."

---

## Quick Revision Card — Section 10

| Topic | Key Point |
|---|---|
| **Dynamic Dispatch** | Runtime polymorphism — JVM decides method based on actual object, not reference type |
| **Serialization** | Convert object ↔ byte stream. Always declare `serialVersionUID`. Use `transient` for passwords. |
| **transient** | Java keyword — skip field during serialization. After deserialization = default value (null/0). |
| **Lombok + Serialization** | Safe with `@Data`. Add `@NoArgsConstructor` if using `@AllArgsConstructor`. Declare `serialVersionUID` manually. |
| **Singleton + Serialization** | Breaks singleton — fix with `readResolve()` returning existing instance. |
| **Try-with-resources** | Auto-closes `AutoCloseable` resources. Eliminates resource leaks. |
| **java.time** | Immutable, thread-safe. `Instant` for timestamps, `LocalDate` for business dates, `ZonedDateTime` for timezone. |
| **Records** | Immutable data carriers. Zero boilerplate. Getters = `user.name()` not `user.getName()`. |
| **Sealed Classes** | Restrict subclasses with `permits`. Enables exhaustive pattern matching. |
| **JVM Architecture** | ClassLoader → Runtime Data Areas (Stack per-thread, Heap shared) → Execution Engine (Interpreter + JIT + GC) |
| **JDK vs JRE vs JVM** | JVM = engine. JRE = JVM + libs (for running). JDK = JRE + tools (for development). |
| **Heap vs Stack** | Heap = objects (shared, GC'd). Stack = local vars/frames (per-thread, auto-popped). |
| **StackOverflow** | Too deep recursion. Fix: base case or iteration. Diagnostic: repeating stack trace. |
| **OutOfMemory** | Heap full. Fix: increase `-Xmx`, fix leaks. Diagnostic: heap dump + MAT. |
| **GC Types** | G1 (default, predictable pauses), ZGC (ultra-low latency <10ms), Parallel (throughput). |
| **String Pool** | JDK 6: PermGen. JDK 7+: Heap (can be GC'd). Literals reused, `new String()` bypasses pool. |
| **Heap Tuning** | `-Xms2g -Xmx2g` (same value). `-XX:MaxMetaspaceSize=256m`. `-XX:+HeapDumpOnOutOfMemoryError`. |
| **Diagnostic Tools** | `jstat` (GC monitor), `jmap` (heap dump), MAT (leak analysis), VisualVM (profiling). |

---

**End of Section 10**

---

## Additional Deep-Dive (Q82-Q96)

### Senior Interview Focus

- Explain JVM issues with evidence: GC logs, heap dump, thread dump, and CPU profile correlation.
- Distinguish allocation pressure problems from memory leaks; many candidates mix these.
- Tie tuning decisions to workload pattern (latency-sensitive vs throughput-heavy), not random JVM flags.

### Real Project Usage

- During production latency spikes, teams typically compare GC pause time against request latency percentiles before changing heap settings.
- String and object churn analysis usually gives bigger wins than raw heap increase.

---

## GC Pause Effects on Multithreading — Stop-The-World and Latency

### Concept
Garbage collection pauses ("stop-the-world" events) freeze all application threads to sweep memory. During a GC pause, no code executes, no requests process. In multithreaded systems, especially those with concurrent threads waiting on locks or I/O, a long GC pause cascades: delayed requests pile up, thread pools fill, downstream databases get thundering herd queries, and recovery is difficult. Production systems must understand GC latency impact and tune accordingly.

### Simple Explanation
Imagine a restaurant kitchen:
- **Pauseless system** (ideal): Waiters always serving customers. Smooth.
- **GC pause** (stop-the-world): Manager calls "KITCHEN FREEZE!" — all cooks, prep staff, everyone stops. No orders taken, no food served for 2 seconds. Customers wait. Queue builds.
- **Multiple worker threads during pause**: 100 waiters all stop at once. For 2 seconds, none serve. Customers queue indefinitely. When pause ends, all 100 waiters restart — thundering herd on kitchen.

### How Pauses Cascade

```
Time T0: GC pauses (STW event)
├─ Thread 1: Frozen at lock.wait()
├─ Thread 2: Frozen mid-request
├─ Thread 3: Frozen in database query (stalled backend)
├─ Thread 4-100: Frozen, waiting for their turn

Time T0+100ms: GC pause ends
├─ All 100 threads wake at once
├─ All 100 queue new DB queries (thundering herd)
├─ Database sees 100 requests spike
├─ Database CPU spikes to 100%
├─ Response times balloon (100ms → 5000ms)
├─ Caller timeouts → cascading failures

Time T0+200ms: Partial recovery, but next GC pause comes
└─ Cycle repeats
```

### Problem 1: Long GC Pause Causes Request Timeout

```java
// Application config
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);    // 5 second timeout
        factory.setReadTimeout(5000);
        return new RestTemplate(factory);
    }
}

// Request scenario
@RestController
public class OrderController {
    @Autowired
    private RestTemplate restTemplate;
    
    @PostMapping("/order")
    public OrderDto createOrder(@RequestBody OrderRequest req) {
        // Call remote payment service
        PaymentDto payment = restTemplate.postForObject(
            "http://payment-service:8080/process",
            req,
            PaymentDto.class  // 5s timeout
        );
        // ...
    }
}

// Problem:
// Time T0: Request starts
// Time T0+3s: GC full pause begins (3 second pause)
// Time T0+6s: GC pause ends, request resumes
// Time T0+8s: Remote call returns
// ❌ Total elapsed: 8 seconds > 5 second timeout
// ❌ SocketTimeoutException even though service responded
```

#### ✅ Solution 1: Tune GC to Reduce Pause Duration

```bash
# JVM flags for low-latency (G1GC, small pauses)
java -Xmx4G \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -Xlog:gc*,safepoint:file=gc.log:time,uptime,level,tags \
    -jar app.jar

# -XX:MaxGCPauseMillis=200 → G1GC tries to keep pauses < 200ms
# Default full GC pause can be 5-15 seconds on 4GB heap
# Tuned G1GC: 100-300ms pauses
```

```java
// Correlate GC pauses with request latency
// After deploying above flags:

// Before:
gc.log:
  [2026-04-10 10:30:00] Full GC: 15234 ms  ← 15 second pause
  requests failed with timeout: 1256

// After G1GC tuning:
gc.log:
  [2026-04-10 11:00:00] GC: 145 ms   ← 145 millisecond pause
  [2026-04-10 11:00:10] GC: 178 ms   ← 178 millisecond pause
  requests failed with timeout: 2 (normal)

// ✅ Pause reduced 100x. Timeout failures drop from 1256 to 2.
```

### Problem 2: GC-Induced Concurrency Bottleneck

```java
// High-throughput trading system
@Service
public class TradeExecutor implements Runnable {
    @Autowired
    private TradeRepository repo;
    private static final int THREAD_COUNT = 32;  // 32 concurrent threads
    
    public void executeAllTrades(List<Trade> trades) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(THREAD_COUNT);
        for (Trade trade : trades) {
            executor.submit(() -> processTradeWrapper(trade));
        }
        executor.awaitTermination(60, TimeUnit.SECONDS);
    }
    
    private void processTradeWrapper(Trade trade) {
        // Busy-wait loop to coordinate with other traded
        while (!trade.isProcessed()) {
            synchronized (trade) {
                // Lock-heavy coordination
            }
            Thread.sleep(1);  // Polling
        }
    }
}

// Scenario without GC tuning:
// All 32 threads processing trades, high throughput, everything smooth
// Time T0: Major GC pause starts (7 seconds)
// All 32 threads freeze
// Time T0+7s: GC ends, all 32 threads resume at once
// All 32 try to acquire synchronized lock on trade object
// Lock contention explosively high
// Good performance drops to bad instantly, briefly
// Time T0+8s: All 32 threads stall waiting for lock
// CPU drops to near 0 even with 32 threads
// Next GC pause comes while all threads idle
// System thrashes
```

#### ✅ Solution 2: Reduce Allocation Churn

```java
// Before: Creates lots of temporary objects
@Service
public class TradeProcessor {
    public BigDecimal computeBonus(Trade trade, int count) {
        List<Trade> similarTrades = new ArrayList<>();  // Allocation 1
        for (int i = 0; i < count; i++) {
            Trade t = new Trade(...);  // Allocations 2-N
            similarTrades.add(t);
        }
        BigDecimal total = BigDecimal.ZERO;  // Object allocation
        for (Trade t : similarTrades) {
            total = total.add(t.getBonus());  // Immutable BigDecimal creates new objects
        }
        return total;
    }
}

// Heap churn: 1000 MB/sec allocation → GC every second, pauses every 3 seconds

// After: Reuse objects, reduce allocation
@Service
public class TradeProcessor {
    private final ThreadLocal<StringBuilder> buffer = 
        ThreadLocal.withInitial(StringBuilder::new);
    
    public long computeBonus(Trade trade, int count) {
        StringBuilder sbuffer = buffer.get();
        sbuffer.setLength(0);
        
        // Reuse buffer, no allocation
        long total = 0L;  // long instead of BigDecimal (no object)
        for (int i = 0; i < count; i++) {
            total += calculateBonusValue(trade, i);
        }
        return total;
    }
}

// Heap churn: 10 MB/sec → GC every 100 seconds, minimal pauses

// Measurement:
// Before: 1500 GC pauses/hour, avg 200ms → cumulative 300s pause time
// After:  15 GC pauses/hour, avg 50ms → cumulative 12.5s pause time
// ✅ Throughput improved ~25x
```

### Problem 3: Multithreaded Lock Contention After Pause

```java
// Thread pool + lock-heavy coordination
@Service
public class BatchProcessor {
    private final Object sharedLock = new Object();
    private List<Result> results = new ArrayList<>();
    
    public void processBatch(List<Item> items) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(16);
        
        for (Item item : items) {
            executor.submit(() -> {
                Result result = processItem(item);
                synchronized (sharedLock) {  // All threads compete for this lock
                    results.add(result);
                }
            });
        }
        
        executor.awaitTermination(60, TimeUnit.SECONDS);
    }
}

// Scenario:
// 16 threads, smooth operation, lock contention low (16 threads, sequential adds)
// Full GC pause at T0 (5 seconds)
// All 16 threads freeze
// At T0+5s, GC ends
// All 16 threads wake and compete for sharedLock
// With GC pause, OS scheduler also jitters
// Lock holder is uncertain; lots of context switches
// Throughput collapses to near-nothing for 1-2 seconds post-pause
```

#### ✅ Solution 3: Use Lock-Free Collections

```java
@Service
public class BatchProcessor {
    // Lock-free, thread-safe collection (backed by CAS loops)
    private final ConcurrentLinkedQueue<Result> results = new ConcurrentLinkedQueue<>();
    
    public void processBatch(List<Item> items) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(16);
        
        for (Item item : items) {
            executor.submit(() -> {
                Result result = processItem(item);
                results.offer(result);  // Lock-free add
            });
        }
        
        executor.awaitTermination(60, TimeUnit.SECONDS);
    }
}

// Behavior post-GC pause:
// 16 threads wake
// Each calls results.offer() (lock-free, fast CAS operations)
// No lock contention
// Throughput recovers faster
```

### Real Project Usage

**Scenario: Payment processing latency SLA**
```
SLA: 95th percentile payment approval latency < 1 second

Before tuning:
- Baseline latency: 200ms (good)
- GC pause: every 10 seconds, 8 second pause
- During pause: all requests stall
- 95th percentile: 8200ms (blown SLA)

After G1GC tuning (MaxGCPauseMillis=150):
- Baseline latency: 200ms
- GC pause: every 3 seconds, \ 150ms pause
- During pause: 150ms spike, request queued briefly
- 95th percentile: 450ms (SLA met)
```

**Diagnostic: Correlate GC with request latency**

```bash
# Produce GC log
java -Xmx8G -XX:+UseG1GC -Xlog:gc*,safepoint:file=gc.log:time,uptime,level,tags -jar app.jar

# Check GC frequency and duration
grep "Pause" gc.log | head -20
# Output:
# [2026-04-10 10:30:02.150] [GC pause (G1 Evacuation Pause) ... 145 ms]
# [2026-04-10 10:30:15.300] [GC pause ... 178 ms]

# Correlate with app logs:
# [2026-04-10 10:30:02] Request latency spike: 500ms (probably GC)
# [2026-04-10 10:30:15] Request latency spike: 600ms (GC)
```

### Tuning Decision Tree

```
GC pause too long?
├─ YES: Full GC (stop-the-world > 1 second)
│   └─ Action: Switch to G1GC, -XX:MaxGCPauseMillis=150
│
├─ Side effect: More frequent but shorter pauses
│   └─ Monitor: GC time per second (useful metric)
│
GC too frequent (pauses add up)?
├─ YES: Heap churn too high
│   └─ Action: Reduce allocations (pool objects, use primitives, StringBuilder)
│
Thread contention post-GC?
├─ YES: Lock-heavy code
│   └─ Action: ConcurrentHashMap, ConcurrentLinkedQueue, avoid synchronized
```

### Interview Answer

> "GC pauses are stop-the-world events — all threads freeze. In a system with 100 concurrent request threads, all 100 freeze for the duration. If pause is 5 seconds, customers wait 5 seconds even though processing would take 200ms.
>
> With multiple threads, the impact multiplies. After a long pause, all threads wake together (thundering herd). They compete for locks and resources. Throughput collapses briefly. Lock contention explodes. If the system uses explicit synchronization, it's especially bad.
>
> Real example: High-frequency trading system. 32 trader threads processing orders. Good throughput. One full GC pause (8 seconds). All 32 froze. When they woke, 32 tried to acquire the same locks to log trade results. Lock contention caused 30 seconds of near-zero throughput even though GC lasted only 8 seconds. Recovery was slow.
>
> Solutions: First, tune GC for shorter pauses. G1GC with MaxGCPauseMillis=150 gives 100-200ms pauses instead of 5-10 second full GC pauses. Second, reduce allocation churn — fewer objects created means less GC pressure. Third, use lock-free collections where possible (ConcurrentHashMap, ConcurrentLinkedQueue). They recover faster post-pause.
>
> Bottom line: GC pauses hit hard in multithreaded systems. Monitor both GC logs and request latency percentiles. If 95th percentile spikes every 10 seconds, look for GC pauses in logs. Correlate them. Tune accordingly."

**Follow-up likely:** "How do you know if your pause is acceptable?" → Compare GC pause duration to your SLA (if SLA is 500ms and GC pause is 300ms, requests during pause miss SLA).

---

## Quick Revision Card — GC Pause Effects

| Scenario | Impact | Measurement | Tuning |
|----------|--------|-------------|--------|
| **Long pause (> 1s)** | Cascading backlog, thundering herd | 95th percentile latency | Switch to G1GC, lower MaxGCPauseMillis |
| **Frequent pauses** | Cumulative stall time high | GC pause time per second | Reduce churn: pool objects, StringBuilder |
| **High thread count** | Lock contention post-pause | Context switches spike after pause | Use ConcurrentHashMap, lock-free collections |
| **Allocation churn** | Heap fills faster → more GC | Allocation rate (MB/sec) | Reuse objects, primitives instead of objects |
| **Full GC during pause** | Compaction pauses added | jstat -gcutil | Tune heap size, young gen ratio |
| **Virtual threads (Java 21)** | More threads = more contention risk | Thread count monitoring | Monitor for GC-induced hangs |

---
