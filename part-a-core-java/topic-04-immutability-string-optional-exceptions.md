# Topic 4 — Immutability, String Internals, Optional & Exception Handling

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer

---

## Part 1: IMMUTABLE CLASS DESIGN & DEFENSIVE COPIES

## Q1. Creating Immutable Classes — Complete Pattern

### Concept
An **immutable class** guarantees that its state cannot change after creation. Every method returns new objects rather than modifying existing ones. Immutable objects are thread-safe by design, enable caching, and simplify reasoning about system state.

### Simple Explanation
Think of a photograph: you take it, it's done, unchangeable. You want a different version? Take a new photo. Never modify the original. Immutable classes work the same way — once created, they never change.

### Complete Immutable Pattern

```java
// ✅ CORRECT: Fully Immutable Class
public final class ImmutableUser {  // ← final prevents subclassing + overriding
    
    // Rule 1: All fields private and final
    private final Long id;
    private final String name;
    private final String email;
    private final List<String> permissions;  // Mutable field — DANGEROUS!
    
    // Rule 2: Constructor accepts all fields, validates, and stores copies of mutable objects
    public ImmutableUser(Long id, String name, String email, List<String> permissions) {
        this.id = id;                                           // Immutable
        this.name = Objects.requireNonNull(name, "Name required");  // Immutable + validated
        this.email = Objects.requireNonNull(email, "Email required");
        // ← CRITICAL: Defensive copy of mutable List
        this.permissions = Collections.unmodifiableList(
            new ArrayList<>(Objects.requireNonNull(permissions))
        );  // Now even if original list is modified, our copy is protected
    }
    
    // Rule 3: Getters return copies of mutable fields (never the original)
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    public List<String> getPermissions() {
        // Return immutable view — caller cannot modify
        return Collections.unmodifiableList(new ArrayList<>(permissions));
        // OR: return List.copyOf(permissions);  // Java 10+, more concise
    }
    
    // Rule 4: No setters! Instead, return new instance with modifications
    public ImmutableUser withEmail(String newEmail) {
        if (this.email.equals(newEmail)) return this;  // Optimization: avoid creating if no change
        return new ImmutableUser(this.id, this.name, newEmail, this.permissions);
    }
    
    // Rule 5: Override equals() and hashCode() (important for use in collections)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ImmutableUser)) return false;
        ImmutableUser user = (ImmutableUser) o;
        return Objects.equals(id, user.id) &&
               Objects.equals(name, user.name) &&
               Objects.equals(email, user.email) &&
               Objects.equals(permissions, user.permissions);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id, name, email, permissions);
    }
    
    // Rule 6: Override toString() for debugging
    @Override
    public String toString() {
        return "ImmutableUser{" +
                "id=" + id + ", name='" + name + '\'' + ", email='" + email + '\'' +
                ", permissions=" + permissions + '}';
    }
}

// Usage
ImmutableUser user = new ImmutableUser(1L, "Alice", "alice@example.com", 
    List.of("READ", "WRITE"));

// Safe: attempting to modify returns new instance
ImmutableUser updatedUser = user.withEmail("newemail@example.com");

System.out.println(user.getEmail());        // "alice@example.com" — unchanged ✓
System.out.println(updatedUser.getEmail()); // "newemail@example.com" — new instance ✓

// Safe: getPermissions() returns immutable copy
List<String> perms = user.getPermissions();
// perms.add("DELETE");  // ← UnsupportedOperationException! ✓
```

### Interview Answer

> "An immutable class needs five things: final class to block subclassing, private fields, final fields to prevent reassignment, no setters, and defensive copies for any mutable field type. The defensive copy is the subtlest rule — if you accept a List in the constructor without copying, the caller still holds the original reference and can mutate it from outside. Same for getters — always return a copy or an unmodifiable view.
>
> In Java 16+ I use records for simple immutable value objects — they enforce all five rules automatically. For complex domain objects with validation logic I still write the class manually. The biggest benefit isn't just thread safety — it's cognitive simplicity. An immutable object can be passed anywhere without defensive thinking about who might change it."
>
> *Likely follow-up: "What is the difference between Java records and a manually written immutable class?"*

### Common Mistakes — Class Not Actually Immutable

```java
// ❌ MISTAKE 1: Mutable field without defensive copy
public class BadUser {
    private final List<String> permissions;  // final, but List is MUTABLE!
    
    public BadUser(List<String> perms) {
        this.permissions = perms;  // ← DANGER: shared reference to original
    }
    
    public List<String> getPermissions() {
        return permissions;  // ← DANGER: returns original reference
    }
}

// Caller can modify:
List<String> perms = new ArrayList<>(List.of("READ"));
BadUser user = new BadUser(perms);
perms.add("WRITE");  // ← Caller's list modified
user.getPermissions().add("DELETE");  // ← Direct mutation through getter!
// BadUser is now mutated — not immutable!

// ✅ FIX: Use defensive copy + unmodifiable wrapper
public class GoodUser {
    private final List<String> permissions;
    
    public GoodUser(List<String> perms) {
        this.permissions = Collections.unmodifiableList(
            new ArrayList<>(perms)  // Defensive copy
        );
    }
    
    public List<String> getPermissions() {
        return permissions;  // Safe: already unmodifiable
    }
}

### Interview Answer

> "Defensive copy prevents external code from silently mutating what should be an immutable object's internals. The two attack vectors are: mutating the original collection after passing it to the constructor, and mutating the collection returned by a getter. Without defensive copies, your 'immutable' object is actually mutable by proxy. The fix: copy on the way in (constructor), and either return a copy or return an unmodifiable view on the way out. I prefer `List.copyOf()` in Java 10+ — it combines the copy and the unmodifiability in one call."
>
> *Likely follow-up: "When would Collections.unmodifiableList() NOT be sufficient?"*

// ❌ MISTAKE 2: Class not final — subclass can break immutability
public class Date implements Immutable {  // ← class not final!
    private final long timestamp;
    
    public Date(long ts) { this.timestamp = ts; }
}

class MutableDate extends Date {  // Oops! Can subclass
    private long mutableTime;
    
    @Override
    public long getTime() {
        return mutableTime;  // ← Returns mutable state!
    }
    
    public void setTime(long t) { mutableTime = t; }  // ← Breaks immutability
}

// ✅ FIX: Mark class as final
public final class Date {  // ← final prevents subclassing
    // ...
}

### Interview Answer

> "For primitives, final locks the value — it can never be assigned again. For object references, final locks the pointer — you can never make the variable point to a different object, but you can call methods on it and change its internal state. A common gotcha: final List<String> still lets you add/remove elements. For an unchangeable collection you need both final AND List.copyOf() or Collections.unmodifiableList().
>
> Practical JMM benefit: final fields are safely published across threads after the constructor finishes — no volatile needed. That's why immutable objects are inherently thread-safe."
>
> *Likely follow-up: "Can a final variable be assigned in a constructor?"*

// ❌ MISTAKE 3: Getter returns mutable object directly
public class Account {
    private final Address address;  // Address is mutable!
    
    public Address getAddress() {
        return address;  // ← Caller can call address.setCity("NYC")
    }
}

// ✅ FIX: Return copy or immutable wrapper
public Address getAddress() {
    return new Address(address);  // Return defensive copy
}
```

---

## Q2. String Fundamentals — Memory, Equality, and Performance

### Concept
`String` is **immutable** in Java — once created, never changes. This immutability enables optimizations (pooling, caching) and thread safety, but requires understanding memory placement and performance implications.

### Simple Explanation
A String is like a permanent engraving on a stone tablet. You can copy it, reference it, but never modify the original. Multiple references to the same text can point to the same tablet (String pooling) because it's guaranteed not to change.

### String Creation — Where Does It Live?

```java
// Type 1: String Literal — stored in STRING POOL (on Heap, special region)
String s1 = "hello";

// Type 2: new String() — creates object on Heap, reference is a NEW object
String s2 = new String("hello");

// Type 3: Concatenation — creates NEW String on Heap
String s3 = "hello" + " world";  // NEW object on heap

String a = "test";
String b = "test";
String c = a + b;  // NEW object, not pooled

// Memory layout (simplified):
//
// HEAP ┌─────────────────────────────────┐
//      │ STRING POOL (Java 8+: in Heap)  │
//      ├─────────────────────────────────┤
//  ┌──┼→ "hello" (shared object)         │        Stack
//  │  │   hashCode=99162, value=[h,e...] │     s1 ──┐
//  │  │                                   │        s2 → [new object on heap]
//  └──┼→ "world" (shared object)          │        s3 → [new object on heap]
//      │   hashCode=113318913            │
//      │                                   │
//      ├─────────────────────────────────┤
//      │ Other HEAP objects               │
//      │  [new String("hello")]           │
//      │  [s1+s2 result: "testhello"]    │
//      └─────────────────────────────────┘
```

### String Pool Details — Java 7 vs Java 8+

```java
// Java 7: String pool was in PERMGEN (separate memory region)
// Problems: LIMITED size, difficult to tune, suffered GC pauses
// Default: -XX:StringTableSize=60013

// Java 8+: String pool MOVED to Heap
// Advantage: Unified garbage collection, easier sizing, larger pool
// Default: -XX:StringTableSize=60013 (can be increased)

// Check pool size:
// -XX:+PrintStringTableStatistics (jdk9+)
// -XX:StringTableSize=1000000  (increase if needed)

// String equality — critical!
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

s1 == s2;        // true — SAME object reference (pooled)
s1 == s3;        // false — DIFFERENT object references
s1.equals(s3);   // true — SAME content ✓

// ✅ Always use .equals() for String comparison, NEVER ==
if (userInput.equals("admin")) {  // ✓ CORRECT
    // ...
}

if (userInput == "admin") {  // ❌ WRONG — might be false even if content matches
    // ...
}
```

### String.intern() — Forced Pooling

```java
// intern() returns the canonical representation — always from pool
String s1 = new String("hello");  // Creates new object on heap
String s2 = "hello";               // References pool

s1 == s2;  // false — different objects

s1 = s1.intern();  // "hello" is interned (or already was)
s1 == s2;  // true — now both reference SAME pool object

// Use case: Memory optimization for known strings
// COMMON STRINGS: country codes, status values, enum-like strings
public class Country {
    public static final String USA = "USA".intern();
    public static final String UK = "UK".intern();
    public static final String CANADA = "CANADA".intern();
    
    // Later:
    String country = ...;
    if (country == Country.USA) {  // Safe: both are interned references
        // ...
    }
}

// ⚠️ WARNING: intern() is expensive (hash table lookup + synchronization)
// Only use if:
// 1. Strings are long (memory savings > overhead)
// 2. Strings are reused many times
// 3. You need identity checks (==)

// Performance: intern() takes O(n) time + synchronization cost
long start = System.nanoTime();
for (int i = 0; i < 1000000; i++) {
    new String("" + i).intern();  // Very slow! 1000000 pool lookups
}
System.out.println((System.nanoTime() - start) / 1e6 + "ms");
```

### Interview Answer
> "String is immutable. In Java 8+, the pool is on the heap, so you don't hit pool-full errors. When you write `String s1 = "hello"`, the JVM looks in the pool — if found, returns that reference. If not found, creates it in the pool. Multiple variables pointing to the same string literal use == correctly. But `new String("hello")` always creates a new object on the heap—a separate allocation. This is why you always use .equals() for string comparison, never ==. intern() manually adds a string to the pool, but it's expensive — O(n) hash lookup plus synchronization. Only use it for known, long-lived strings that are reused."
>
> *Likely follow-up: "Why does the String pool improve performance?"*

---

## Q3. String vs StringBuilder vs StringBuffer — When to Use Each

### Concept
- **String**: immutable, thread-safe by design, GC overhead (creates new objects per edit)
- **StringBuilder**: mutable buffer, NOT thread-safe, for single-threaded performance
- **StringBuffer**: mutable buffer, synchronized (slower), legacy before StringBuilder

### Code Comparison

```java
// ❌ WRONG: String concatenation in loop — creates O(n) new objects
String result = "";
for (String word : words) {  // N = words.length
    result = result + word;  // Creates NEW String each iteration
}  // Time: O(N²), Space: O(N²) due to string copies

// ✅ CORRECT: StringBuilder for mutable operations
StringBuilder sb = new StringBuilder();
for (String word : words) {
    sb.append(word);  // O(1) amortized append to buffer
}
String result = sb.toString();  // O(N) final conversion
// Time: O(N), Space: O(N) ✓

// Performance: 1M string concatenations
// String loop: ~2000ms
// StringBuilder: ~2ms (1000× faster!)

// When each option is right:
String s = "fixed";                    // ✓ String — never changes
String url = "http://" + host + ":";   // ✓ String — small, one-time
String formatted = String.format("User: %s, Age: %d", name, age); // ✓ String

StringBuilder response = new StringBuilder();  // ✓ StringBuilder — building in loop
for (String line : lines) {
    response.append(line).append("\n");
}

StringBuffer legacy = new StringBuffer();  // ❌ Avoid — synchronized overhead
// Only if sharing across threads (unusual)
```

### Performance Table

| Operation | String | StringBuilder | StringBuffer |
|-----------|--------|---------------|--------------|
| **append()** | O(n) creates new | O(1) amortized | O(1) amortized |
| **Thread-safe** | Yes (immutable) | No | Yes (uses locks) |
| **Memory** | Creates copies | Reuses buffer | Reuses buffer |
| **Use case** | Fixed content | Single-thread building | Legacy multi-thread |

### Interview Answer
> "StringBuilder for building strings in a loop (single-threaded), StringBuffer for legacy multi-threaded code (now rarely needed), String for fixed content. The critical performance mistake is concatenating in a loop with String: `String s = ""; for (x in xs) s += x;` is O(n²) — each += creates a new String and copies all previous characters. StringBuilder is O(n) — reuses a buffer. In production I always use StringBuilder for any dynamic string building and StringBuffer almost never (just use AtomicReference<String> if thread-safe updates needed)."
>
> *Likely follow-up: "Why is String concatenation in a loop O(n²)?"*

---

## Part 2: OPTIONAL — AVOIDING NULL

## Q4. Optional Deep Dive — Patterns and Anti-Patterns

### Concept
`Optional<T>` wraps nullable values and forces you to handle the absence case explicitly, eliminating **NullPointerException** risks and clarifying API intent.

### Simple Explanation
A box that either contains a value or is empty. You must check if it's empty before taking out the value. No surprise null pointers — the intent is explicit.

### Correct Optional Usage Patterns

```java
// ❌ ANTI-PATTERN 1: Overusing Optional
Optional<String> name = Optional.of("Alice");
if (name.isPresent()) {  // ← This defeats the purpose!
    System.out.println(name.get());
}
// Just use String directly if you know it's non-null!

// ✅ CORRECT: Use functional operations (map, flatMap, filter)
Optional<User> user = userRepository.findById(1L);

// Option 1: map() — transform if present
Optional<String> userName = user.map(User::getName);

// Option 2: filter() — keep only if condition true
Optional<User> adult = user.filter(u -> u.getAge() >= 18);

// Option 3: flatMap() — flatten nested Optionals
Optional<Address> address = user
    .flatMap(u -> u.getAddress())  // Returns Optional<Address>
    .map(Address::getCity);

// Option 4: orElse() — provide default if empty
String name = user.map(User::getName).orElse("Unknown");

// Option 5: orElseGet() — compute default only if empty (lazy)
String name = user.map(User::getName)
    .orElseGet(() -> loadDefaultName());  // Called only if empty

// Option 6: orElseThrow() — throw exception if empty
User u = user.orElseThrow(() -> new EntityNotFoundException("User not found"));

// Option 7: ifPresent() + ifPresentOrElse() — side effects
user.ifPresent(u -> log.info("Found user: {}", u.getName()));

user.ifPresentOrElse(
    u -> log.info("User: {}", u.getName()),
    () -> log.warn("User not found")
);
```

### Interview Answer

> "Optional was introduced to make the absence of a value an explicit part of the API contract — rather than returning null and hoping callers check. It's not about eliminating all null checks; it's about representing optionality in the type system so callers are forced to think about it. The key methods: map for transforming if present, flatMap when the transformation returns an Optional, orElseGet for lazy defaults, orElseThrow for the mandatory case. It chains beautifully in stream pipelines — you can call .map() multiple times and the whole chain short-circuits at the first empty Optional."
>
> *Likely follow-up: "When should Optional NOT be used?"*

### orElse() vs orElseGet() — Critical Difference

```java
// ❌ MISTAKE: using orElse() with expensive computation
Optional<Profile> profile = findProfile(id);
Profile p = profile.orElse(loadProfileFromDB());  // ← ALWAYS called!
// Even if Optional is present, loadProfileFromDB() executes (wasted!)

// ✅ CORRECT: orElseGet() with lambda (lazy evaluation)
Profile p = profile.orElseGet(() -> loadProfileFromDB());
// Expensive function only called if Optional is empty

// Performance test:
Optional<String> opt = Optional.of("value");

opt.orElse(expensiveOperation());      // 100ms (always runs)
opt.orElseGet(() -> expensiveOperation());  // 0ms (skipped if present) ✓
```

### map() vs flatMap() — When Nesting Occurs

```java
// ❌ DANGER: map() with function returning Optional creates nested Optional
Optional<User> user = ...;
Optional<Optional<Address>> address1 = user.map(u -> u.getAddressOptional());
// Type is Optional<Optional<Address>> — double-wrapped! Awkward to handle

// ✅ CORRECT: flatMap() flattens automatically
Optional<Address> address2 = user.flatMap(u -> u.getAddressOptional());
// Type is Optional<Address> — single-wrapped ✓

// Real scenario:
public Optional<String> getUserCity(Long userId) {
    return userRepository.findById(userId)
        .flatMap(user -> user.getAddress())      // Returns Optional<Address>
        .flatMap(address -> address.getCity());  // Returns Optional<String>
        // Result: Optional<String> — clean without nested optionals
}

// With map() (wrong):
public Optional<String> badVersion(Long userId) {
    return userRepository.findById(userId)
        .map(user -> user.getAddress())          // map returns Optional<Optional<Address>>
        .map(optAddress -> optAddress.flatMap(a -> a.getCity()))  // ← Ugly!
}
```

### Anti-Patterns to Avoid

```java
// ❌ ANTI-PATTERN 1: Get and null-check (defeats purpose)
Optional<User> user = findUser();
if (user.isPresent()) {
    User u = user.get();  // ← Just use String directly if you'll check!
    // ...
}
// Just use: User u = findUser().orElse(null) or findUserDirect()

// ❌ ANTI-PATTERN 2: Optional<Optional<T>> (double-wrapped)
Optional<Optional<String>> value = ...;
// Flattens to: Optional<String> using flatMap()

// ❌ ANTI-PATTERN 3: Storing Optional in fields/parameters
public class User {
    private Optional<String> nickname;  // ← Bad practice!
    // Just use: private String nickname (nullable)
}

// ✅ CORRECT: Optional only for return values and method parameters
public Optional<User> findById(Long id) { ... }  // Good
Optional<String> getNickname() { ... }  // Good
void setNickname(Optional<String> nick) { ... }  // Acceptable but rarely used

// ❌ ANTI-PATTERN 4: Using Optional for primitive values
Optional<Integer> age = Optional.of(25);  // ✓ Works but wasteful

// ✅ CORRECT: Use OptionalInt, OptionalLong, OptionalDouble
OptionalInt age = OptionalInt.of(25);  // No boxing overhead ✓
age.ifPresent(a -> System.out.println(a));

// ❌ ANTI-PATTERN 5: Exception in map() chain not caught
Optional<Integer> result = Optional.of("123")
    .map(Integer::parseInt);  // ← If "ABC", throws NumberFormatException!
// Exception escapes Optional chain

// ✅ CORRECT: Catch exceptions before Optional
Optional<Integer> result = Optional.empty();
try {
    result = Optional.of(Integer.parseInt("123"));
} catch (NumberFormatException e) {
    result = Optional.empty();  // Handle gracefully
}
```

### Interview Answer

> "Optional doesn't eliminate NPE — it makes the absence of a value explicit in the type system so callers handle it intentionally. But it has real drawbacks: not Serializable so it can't be used in JPA entities or Kafka messages; creates object overhead on hot paths; and it's frequently misused.
>
> The golden rule: use it only as a return type, never as a field or parameter, never wrapping a collection. And the interview favourite: `orElse(expensiveMethod())` is EAGER — that expression is always evaluated regardless of whether the Optional has a value. `orElseGet(() -> expensiveMethod())` is LAZY — the supplier only runs when the Optional is empty. This can cause real performance bugs in code that's calling a DB or external API in the `orElse` clause."
>
> *Likely follow-up: "What is the difference between Optional.of() and Optional.ofNullable()?"*

---

## Part 3: EXCEPTION HANDLING

## Q5. Java Exception Hierarchy — Checked vs Unchecked

### Concept
Java divides exceptions into **checked** (must be caught) and **unchecked** (optional to catch). Understanding which is which prevents surprises at runtime.

### Exception Hierarchy Explained

```
                    Throwable
                   /         \
                Error      Exception
               /   |   \      /   |    \
           VirtualMachineError  ...  RuntimeException  IOException  ...
                                       / |    \
                              NullPointerException
                              ArithmeticException
                              IndexOutOfBoundsException
                              ...

CHECKED (must catch/throw):
├─ IOException (file not found, network errors)
├─ SQLException (database errors)
├─ TimeoutException
├─ ParseException
└─ Custom checked exceptions

UNCHECKED (optional to catch):
├─ RuntimeException (all subclasses are unchecked)
│  ├─ NullPointerException — accessing methods on null
│  ├─ IllegalArgumentException — invalid argument value
│  ├─ IndexOutOfBoundsException — array index out of range
│  ├─ ConcurrentModificationException — modified collection while iterating
│  └─ Other programming errors
├─ Error (do not catch — JVM failures)
│  ├─ StackOverflowError — infinite recursion
│  ├─ OutOfMemoryError — heap exhausted
│  └─ Other JVM-level failures
```

### Checked vs Unchecked — When to Use Each

```java
// ✅ CHECKED: For recoverable, external conditions
public class FileReader {
    public String readFile(String filename) throws FileNotFoundException, IOException {
        // FileNotFoundException — file might not exist (expected)
        // IOException — disk I/O might fail (expected)
        // Caller MUST handle these
    }
}

// Usage required to handle:
try {
    String content = reader.readFile("config.txt");
} catch (FileNotFoundException e) {
    log.error("Config file not found, using defaults");
    // Recover gracefully
} catch (IOException e) {
    log.error("Failed to read config", e);
    // Handle I/O failure
}

// ✅ UNCHECKED: For programming errors (preventable)
public List<User> getPage(int pageNumber, int pageSize) {
    if (pageNumber < 0 || pageSize <= 0) {
        throw new IllegalArgumentException("Invalid page params");
    }
    // Not checked — caller should validate, not catch
}

// ❌ DON'T catch unchecked exceptions normally
try {
    users.get(-1);  // ← Will throw IndexOutOfBoundsException
} catch (IndexOutOfBoundsException e) {
    // ❌ Bad — should have validated index before calling
}

// ✅ CORRECT — validate before calling
if (index >= 0 && index < users.size()) {
    users.get(index);  // Safe
}

// ✅ CORRECT — let programming errors propagate
int result = 10 / divisor;  // If divisor=0 → ArithmeticException
// Don't catch it — fix the code to validate divisor
```

### Custom Exceptions — Design Guidelines

```java
// ✅ CREATE checked exception for recoverable external conditions
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
    
    public InsufficientFundsException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Usage:
public void withdraw(double amount) throws InsufficientFundsException {
    if (amount > balance) {
        throw new InsufficientFundsException(
            "Insufficient funds: need " + amount + ", have " + balance
        );
    }
    balance -= amount;
}

// Caller handles:
try {
    account.withdraw(1000);
} catch (InsufficientFundsException e) {
    log.warn(e.getMessage());  // Expected condition
    // Retry, notify user, etc.
}

// ✅ CREATE unchecked exception for programming errors
public class InvalidUserIdException extends RuntimeException {
    public InvalidUserIdException(String message) {
        super(message);
    }
}

// Usage:
public User getUser(Long userId) {
    if (userId == null || userId <= 0) {
        throw new InvalidUserIdException("User ID must be positive: " + userId);
    }
    return repository.findById(userId);
}

// Caller doesn't catch — should validate before calling
if (userId != null && userId > 0) {
    User u = service.getUser(userId);
}
```

### Exception Anti-Patterns to Avoid

```java
// ❌ ANTI-PATTERN 1: Catching Exception (too broad)
try {
    doSomething();
} catch (Exception e) {  // ← Catches everything including programming errors!
    log.error("Error: {}", e.getMessage());
}
// Masks bugs. Specific catches are better.

// ✅ CORRECT: Catch specific exceptions
try {
    doSomething();
} catch (FileNotFoundException e) {
    log.error("File not found", e);
} catch (IOException e) {
    log.error("I/O error", e);
}

// ❌ ANTI-PATTERN 2: Swallowing exceptions silently
try {
    doSomething();
} catch (Exception e) {
    // Silence — no logging!
}

// ✅ CORRECT: Always log or re-throw
try {
    doSomething();
} catch (IOException e) {
    log.error("Operation failed", e);  // Log the error
    throw new RuntimeException("Unexpected error", e);  // Re-throw if critical
}

// ❌ ANTI-PATTERN 3: Throwing generic Exception
public void process() throws Exception {  // ← Too vague!
    // Caller doesn't know what to expect
}

// ✅ CORRECT: Throw specific exceptions
public void process() throws FileNotFoundException, IOException {
    // Caller knows exactly what must be handled
}

// ❌ ANTI-PATTERN 4: Losing the cause (root exception)
try {
    database.connect();
} catch (SQLException e) {
    throw new RuntimeException("Failed");  // ← Lost e!
}

// ✅ CORRECT: Preserve the cause
try {
    database.connect();
} catch (SQLException e) {
    throw new RuntimeException("Failed to connect", e);  // ← Preserves e
}
// Now stack trace shows original cause
```

---

## Q6. Try-With-Resources and Suppressed Exceptions

### Concept
**Try-with-resources** automatically closes resources (files, connections) even if an exception occurs, eliminating manual finally blocks. **Suppressed exceptions** track multiple failures during cleanup.

### Simple Explanation
Like closing a door automatically when you leave a room — even if you forgot to grab your bag (exception), the door still closes.

### Without Try-With-Resources (Old Way)

```java
// ❌ Manual close — error-prone
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    String line = reader.readLine();
    // ... process line
} catch (IOException e) {
    log.error("Failed to read", e);
} finally {
    if (reader != null) {
        try {
            reader.close();  // ← Can throw IOException!
        } catch (IOException e) {
            log.error("Failed to close", e);  // Logs cleanup error
        }
    }
}

// Problems:
// 1. Verbose boilerplate
// 2. close() can throw (must handle)
// 3. Easy to forget finally block
// 4. Multiple resources = nested try-finally (nightmare)
```

### With Try-With-Resources (Java 7+)

```java
// ✅ Automatic close (implement AutoCloseable)
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line = reader.readLine();
    // ... process line
} catch (IOException e) {
    log.error("Failed to read", e);
}
// reader.close() is automatic! No finally needed.

// Multiple resources — all closed in reverse order
try (
    BufferedReader reader = new BufferedReader(new FileReader("input.txt"));
    BufferedWriter writer = new BufferedWriter(new FileWriter("output.txt"))
) {
    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line);
        writer.newLine();
    }
} catch (IOException e) {
    log.error("Failed to process files", e);
}
// Both reader and writer closed automatically, even if exception occurs ✓
```

### Suppressed Exceptions — Multiple Failures

```java
// Scenario: close() throws exception WHILE handling another exception
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    int x = 10 / 0;  // ← Throws ArithmeticException
    // Meanwhile, close() is called and throws IOException
} catch (Exception e) {
    log.error("Exception: {}", e.getMessage());
    // e = ArithmeticException
    // IOException from close() is SUPPRESSED (added to e.getSuppressed())
    
    System.out.println("Primary exception: " + e);
    System.out.println("Suppressed exceptions:");
    for (Throwable suppressed : e.getSuppressed()) {
        System.out.println("  - " + suppressed);  // ← Shows IOException
    }
}

// Output:
// Primary exception: ArithmeticException: / by zero
// Suppressed exceptions:
//   - IOException: Closed

// ✅ CORRECT: Check suppressed to understand all failures
catch (IOException e) {
    for (Throwable suppressed : e.getSuppressed()) {
        if (suppressed instanceof IOException) {
            log.warn("Close failed during cleanup: {}", suppressed.getMessage());
        }
    }
    throw e;
}
```

### Creating Custom AutoCloseable Resources

```java
// ✅ Implement AutoCloseable for safe resource management
public class Database implements AutoCloseable {
    private Connection conn;
    
    public Database(String url) throws SQLException {
        this.conn = DriverManager.getConnection(url);
    }
    
    public void execute(String sql) throws SQLException {
        conn.executeUpdate(sql);
    }
    
    @Override
    public void close() throws SQLException {
        if (conn != null) {
            conn.close();
        }
    }
}

// Usage:
try (Database db = new Database("jdbc:mysql://localhost/mydb")) {
    db.execute("INSERT INTO users VALUES (...)");
} catch (SQLException e) {
    log.error("Database error", e);
}
// conn.close() called automatically ✓
```

---

## Q7. Multi-Catch and Exception Hierarchy

### Concept
**Multi-catch** (Java 7+) allows catching multiple exception types in one catch block, reducing code duplication when the same handler works for different exceptions.

### Code Comparison

```java
// ❌ OLD: Duplicate handling code
try {
    processFile();
} catch (FileNotFoundException e) {
    log.error("File operation failed: {}", e.getMessage());
    metrics.increment("file.error");
} catch (IOException e) {
    log.error("File operation failed: {}", e.getMessage());
    metrics.increment("file.error");
}

// ✅ NEW: Multi-catch (pipe-separated types)
try {
    processFile();
} catch (FileNotFoundException | IOException e) {  // ← Union type
    log.error("File operation failed: {}", e.getMessage());
    metrics.increment("file.error");
}

// Rules for multi-catch:
// 1. Exceptions must NOT be in inheritance relationship
//    catch (IOException | FileNotFoundException e) ← WRONG! FileNotFoundException extends IOException
//    Compiler error: "The exception FileNotFoundException is already caught"
//
// 2. Types must be disjoint (unrelated)
//    catch (IOException | SQLException e) ← OK: no relationship
//
// 3. Can catch from different branches of exception tree
//    catch (SQLException | ParseException e) ← OK: both checked, unrelated
```

### Advanced Pattern — Different Handlers

```java
try {
    readDatabase();
    parseConfig();
    processFile();
} catch (SQLException e) {
    log.error("Database error", e);
    retryDatabaseOperation();
} catch (ParseException e) {
    log.error("Config parse error", e);
    useDefaultConfig();
} catch (FileNotFoundException e) {
    log.error("File not found", e);
    createDefaultFile();
} catch (IOException e) {
    log.error("I/O error", e);
    // Generic I/O handler (parent of FileNotFoundException)
}

// ✅ Multi-catch where handling is the same
try {
    readDatabase();
    parseConfig();
    processFile();
} catch (SQLException | ParseException e) {  // Different sources, same handler
    log.error("Configuration error", e);
    loadDefaults();
} catch (FileNotFoundException | IOException e) {  // Related but different recovery
    if (e instanceof FileNotFoundException) {
        createDefaultFile();
    } else {
        retryWithBackoff();
    }
}
```

### Interview Answer
> "Multi-catch lets you handle multiple unrelated exceptions in one block—they don't need a common superclass. It's syntactic sugar for avoiding copy-paste of identical log/recovery code. The key rule: types must be disjoint (don't catch both IOException and FileNotFoundException—FileNotFoundException is-a IOException, so use single catch). In production this prevents exception hierarchy mistakes and keeps catch blocks DRY. Rare edge case: if you throw from the catch block, the thrown exception type is the union (compile-time), so calling code must handle all caught types—forces clarity on what can actually escape."
>
> *Likely follow-up: "What happens if you accidentally catch a parent and child exception in multi-catch?"*

---

## Real Project Usage — Topic 4 (Immutability & Exception Handling)

### Scenario 1: Defensive Copy Leak in User Service
> "Our User service accepted a List<String> of roles in the constructor but stored it directly. A caller modified the list after construction — roles changed without going through the service. Fix: store a defensive copy with `Collections.unmodifiableList(new ArrayList<>(roles))` and return copies from getters. Now the internal state is protected."

### Scenario 2: String Concatenation in Loop (Performance Disaster)
> "Payment API logs were slow because we concatenated transaction details in a loop: `String log = ""; for (...) log += txn.details();` (O(n²)). With 1000 transactions, this created 1000 intermediate String objects. Switched to StringBuilder: `StringBuilder sb = new StringBuilder(); for (...) sb.append(...);`. Latency dropped 60%."

### Scenario 3: Optional.get() without isPresent() Check
> "Data pipeline crashed with NoSuchElementException because a method returned `Optional<Config>` but a junior used `.get()` directly without checking. Fixed by enforcing `.orElseThrow(ConfigNotFoundException::new)` in code review and using `.ifPresentOrElse()` patterns. Code is now intent-explicit."

### Scenario 4: Exception Hierarchy in Bulk Operations
> "Bulk email service caught `Exception e` broadly, swallowing configuration errors silently. Refactored: separated checked exceptions for external failures (IOException → retry) from unchecked for bugs (NPE → investigate). Multi-catch: `catch (IOException | SocketTimeoutException e)` for distinct retry logic."

---

## Quick Revision Card — Topic 4

| Question | Core Concept & Key Points |
|----------|---|
| **Q1** | Immutable class: final class, final fields, defensive copy on mutable, return copies from getters, no setters. Pattern: constructor validates + copies, methods return new instances. |
| **Q2** | String is immutable. String pool (Java 8+: on Heap). String s1="hello", s2="hello" → same reference (==). new String("hello") → different reference. Always use .equals(), never ==. intern() for known strings. |
| **Q3** | StringBuilder for building (single-threaded), StringBuffer for legacy, String for fixed. String concat in loop = O(n²), StringBuilder = O(n). orElse() lazy: use orElseGet(). |
| **Q4** | Optional: map() transforms, flatMap() flattens, filter() keeps if condition, orElse() default. Avoid: isPresent()+get(), storing in fields, Optional<Optional<T>>. |
| **Q5** | Checked exceptions (IOException, SQLException): recoverable, must catch. Unchecked (RuntimeException): programming errors, don't need to catch. Custom checked for external failures, unchecked for bugs. |
| **Q6** | Try-with-resources: implements AutoCloseable. Automatic close even on exception. Suppressed exceptions: check e.getSuppressed() for cleanup failures. |
| **Q7** | Multi-catch (A | B | C): exceptions must be unrelated. Reduces code duplication. FileNotFoundException | IOException → WRONG (inheritance). |

---

**End of Topic 4 — Immutability, String Internals, Optional & Exception Handling**
