# PART A: Core Java — Topic 2: Call by Value, Method Overloading & Memory Model

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer → Related Concepts

---

## Q1. Java Pass-by-Value: Primitives vs Objects vs References

### Concept
Java is **always call by value** — but for objects, the *value* being passed is the **reference** (memory address), not a copy of the object itself. This distinction is critical for understanding why object mutations persist after method calls but reassignments don't.

### Simple Explanation
Think of it like a house key:
- **Primitive** → you give someone a photocopy of a number written on paper. They can change their copy; yours stays intact.
- **Object** → you give someone a copy of your house key (the address/reference). They can walk in and rearrange the furniture (mutate the object). But if they swap their copy of the key for a different house's key (reassign the reference), you still hold your original key to the original house.

### How It Works

#### Stack vs Heap Memory Layout

```
When you execute:
    int count = 10;
    User user = new User("Alice");

Stack (per thread):          Heap (shared):
┌─────────────────┐         ┌──────────────────────────┐
│ count: 10       │         │ User {                   │
│ user:  0x1234 ──┼────────→│   name: "Alice"          │
└─────────────────┘         │   age: 30                │
                            │ }                        │
                            └──────────────────────────┘

Primitives: value stored directly on STACK
Objects: reference (memory address) on STACK, actual object on HEAP
```

#### Case 1: Primitive Parameter (Completely Isolated)

```java
static void changePrimitive(int x) {
    x = 100;  // Only affects LOCAL copy on this method's stack frame
}

int num = 10;
changePrimitive(num);
System.out.println(num);  // 10 — UNCHANGED

// Why? Stack memory is isolated per method call:
// 1. num = 10 on main()'s stack frame
// 2. changePrimitive() creates its OWN stack frame with x = 10 (copy of value)
// 3. x = 100 modifies changePrimitive()'s frame only
// 4. changePrimitive() stack frame destroyed; main's num never touched
```

#### Case 2: String Parameter (Immutable Reference Swap)

```java
static void changeString(String s) {
    s = "changed";  // s now points to a NEW String object; caller's ref unchanged
}

String str = "hello";
changeString(str);
System.out.println(str);  // "hello" — UNCHANGED

// Why?
// 1. Stack: str references String object "hello" (address 0x5000)
// 2. changeString() creates its own stack frame; s = 0x5000 (copy of REFERENCE)
// 3. s = "changed" makes s point to NEW object at 0x6000
// 4. original str still points to 0x5000 ("hello")
// 5. The old string 0x5000 eligible for GC when str goes out of scope
```

#### Case 3: StringBuffer Parameter (Mutation Visible, Reassignment Not)

```java
static void changeStringBuffer(StringBuffer sb) {
    sb.append(" World");         // Mutates the SAME object caller sees
    sb = new StringBuffer("X");  // Reassignment NOT visible to caller
}

StringBuffer sb = new StringBuffer("Hello");
changeStringBuffer(sb);
System.out.println(sb);  // "Hello World" — MUTATION visible, NOT "X"

// Why?
// 1. Stack: sb references StringBuffer at 0x3000
// 2. changeStringBuffer() creates its own stack frame; local sb = 0x3000
// 3. sb.append() MUTATES the object at 0x3000 → now "Hello World"
// 4. sb = new StringBuffer("X") makes changeStringBuffer's local sb point to 0x4000
// 5. caller's sb still points to 0x3000 "Hello World"
// 6. The new StringBuffer at 0x4000 is GC'd when changeStringBuffer() returns
```

#### Case 4: Custom Object Parameter (Mutation Visible, Reassignment Not)

```java
class Employee {
    private String name;
    public Employee(String name) { this.name = name; }
    public void setName(String name) { this.name = name; }
    public String getName() { return name; }
}

static void changeEmployee(Employee e) {
    e.setName("Bob");            // Caller sees this mutation
    e = new Employee("Carol");   // This reassignment is NOT visible
}

Employee emp = new Employee("Alice");
changeEmployee(emp);
System.out.println(emp.getName());  // "Bob" — mutation visible, NOT "Carol"

// Same principle as StringBuffer:
// - Mutation (setName) changes the original object → visible to caller
// - Reassignment (e = new Employee) only affects local reference → invisible to caller
```

### Complete Example with Output

```java
public class PassingDemo {

    static void changePrimitive(int x) {
        x = 100;  
    }

    static void changeString(String s) {
        s = "changed";  
    }

    static void changeStringBuffer(StringBuffer sb) {
        sb.append(" World");         
        sb = new StringBuffer("X");  
    }

    static void changeEmployee(Employee e) {
        e.setName("Bob");            
        e = new Employee("Carol");   
    }

    public static void main(String[] args) {
        int num = 10;
        changePrimitive(num);
        System.out.println(num);         // 10 — unchanged

        String str = "hello";
        changeString(str);
        System.out.println(str);         // "hello" — unchanged

        StringBuffer sb = new StringBuffer("Hello");
        changeStringBuffer(sb);
        System.out.println(sb);          // "Hello World" — mutation visible

        Employee emp = new Employee("Alice");
        changeEmployee(emp);
        System.out.println(emp.getName()); // "Bob" — mutation visible
    }
}

// Output:
// 10
// hello
// Hello World
// Bob
```

### Real Project Usage

**Defensive Copy Pattern (Immutability):**
```java
public class User {
    private String name;
    private List<String> tags;

    // WRONG: exposes mutable list
    public List<String> getTags() {
        return tags;  // Caller can mutate: getTags().add("hacker")
    }

    // RIGHT: return copy or unmodifiable view
    public List<String> getTags() {
        return Collections.unmodifiableList(tags);  // Mutation attempt throws
    }

    public User(String name, List<String> tags) {
        this.name = name;
        this.tags = new ArrayList<>(tags);  // Defensive copy IN
    }
}

// Caller tries mutation:
User user = new User("Alice", List.of("admin", "user"));
user.getTags().add("hacker");  // throws UnsupportedOperationException
```

### Interview Answer

> "Java is strictly call by value. For primitives, a copy of the **value** is passed — the original variable is completely isolated. For objects, a copy of the **reference** (memory address) is passed. This means methods can mutate the object's state through the reference (e.g., `setName()` changes what the caller sees), but cannot make the caller's variable point to a different object (reassignment inside the method is invisible to the caller).
>
> String behaves like a primitive from an observability standpoint because it's immutable — any apparent 'change' creates a new object, leaving the caller's reference pointing to the original. StringBuffer and custom objects are mutable; mutations are visible to the caller, but reassignments are not.
>
> This distinction is crucial for defensive copying: if you return a mutable object like a List, the caller can mutate it, breaking your invariants. That's why we return `Collections.unmodifiableList()` or `List.copyOf()` instead."
>
> *Likely follow-up: "So if Java is always call by value, why do object mutations persist after a method call but reassignments don't?"* → Answer: Because you're modifying the object itself (mutation) through the shared reference, not changing what the reference points to.

---

## Q2. Method Overloading: String vs Object, Passing null

### Concept
When you call an overloaded method with `null`, Java picks the **most specific** type in the inheritance hierarchy that matches. The resolution follows a priority: exact type match → autobox/widen → varargs. Ambiguity at the same level causes compile error.

### Simple Explanation
Java's overload resolution is like a job hiring filter — it always picks the most qualified (most specific) candidate. `String` is more specific than `Object` because `String extends Object`. When you pass `null`, Java can't determine the "type" of null, so it looks for the **most specific parameter type that accepts null** — which is the most specific type in the entire overload set.

### How It Works (Overload Resolution Rules)

**Priority (Java picks first matching rule):**
1. Exact type match (e.g., `String` → parameter type `String`)
2. **Widening** (primitive: `int` → `long`; reference: `String` → `Object`)
3. **Autoboxing** (primitive: `int` → `Integer`)
4. **Varargs** (last resort)

**For `null` specifically:**
- `null` has no type
- When multiple overloads could accept `null`, Java picks the **most specific** (lowest in inheritance hierarchy)
- If two overloads are equally specific, it's ambiguous → compile error

### Code Examples

#### Example 1: String is More Specific than Object

```java
class OverloadDemo {
    static void print(String s) {
        System.out.println("String: " + s);
    }

    static void print(Object o) {
        System.out.println("Object: " + o);
    }

    public static void main(String[] args) {
        print("hello");    // String: hello    — literal is type String
        print(42);         // Object: 42       — int autoboxed to Integer, which extends Object
        print(null);       // String: null     — String is MORE specific than Object
    }
}

// Output:
// String: hello
// Object: 42
// String: null
```

**Why `null` chooses `String`?**
- Both overloads accept `null` (since all reference types do)
- `String` is more specific (in inheritance hierarchy below `Object`)
- Java applies "most specific applicable method" rule
- Result: `String` overload is chosen

#### Example 2: Equally Specific = Ambiguous

```java
class AmbiguousOverload {
    static void print(String s) {
        System.out.println("String: " + s);
    }

    static void print(Integer i) {
        System.out.println("Integer: " + i);
    }

    public static void main(String[] args) {
        print("hello");    // String: hello    — obvious
        print(42);         // Integer: 42      — autoboxed
        print(null);       // COMPILE ERROR: ambiguous!
    }
}

// Compile error:
// error: reference to print is ambiguous
// both method print(String) in AmbiguousOverload and method print(Integer) in AmbiguousOverload match
```

**Why ambiguous?**
- Both `String` and `Integer` could accept `null`
- Neither is a subtype of the other
- Java can't determine which is "more specific"
- Error at compile time (good!)

**Fix with explicit cast:**
```java
print((String) null);   // Forces String overload
print((Integer) null);  // Forces Integer overload
```

#### Example 3: Three Overloads (String, CharSequence, Object)

```java
class ThreeOverloads {
    static void print(String s) {
        System.out.println("String");
    }

    static void print(CharSequence cs) {
        System.out.println("CharSequence");
    }

    static void print(Object o) {
        System.out.println("Object");
    }

    public static void main(String[] args) {
        print("hello");  // String        — exact match
        print(null);     // String        — most specific (String < CharSequence < Object)
    }
}

// Output:
// String
// String

// Inheritance: String → CharSequence → Object (both interfaces/classes String implements)
// String is most specific → chosen for null
```

**Specificity ordering: `String` → `CharSequence` → `Object`**

#### Example 4: Primitive Widening vs Autoboxing

```java
class WideneVsAutobox {
    static void test(long x) {
        System.out.println("long: " + x);
    }

    static void test(Integer x) {
        System.out.println("Integer: " + x);
    }

    public static void main(String[] args) {
        test(5);  // long: 5
        // Why? Widening (int → long) is preferred over autoboxing (int → Integer)
    }
}

// Output:
// long: 5
```

**Widening beats autoboxing:**
- `5` is an `int`
- Could widen to `long` (widening rule)
- Could autobox to `Integer` (autoboxing rule)
- Widening > autoboxing in priority
- Result: `long` overload chosen

### Real Project Usage

**Builder API (Leverages Overloading):**
```java
public class HttpRequestBuilder {
    private String url;
    private String method = "GET";
    private Map<String, String> headers = new HashMap<>();

    public HttpRequestBuilder url(String url) {
        this.url = url;
        return this;
    }

    // Overload: string value
    public HttpRequestBuilder header(String key, String value) {
        headers.put(key, value);
        return this;
    }

    // Overload: object value (converted to String)
    public HttpRequestBuilder header(String key, Object value) {
        headers.put(key, value.toString());
        return this;
    }

    public static void main(String[] args) {
        HttpRequestBuilder req = new HttpRequestBuilder()
            .url("https://api.example.com/users")
            .header("Content-Type", "application/json")  // String overload
            .header("X-Retry-Count", 3);                 // Object overload (autoboxed)
    }
}
```

**API Design (Be Careful with Overloading!):**

```java
// CONFUSING: Multiple overloads that differ subtly
public class Container {
    public void add(List<String> items) { }
    public void add(List<Integer> items) { }
    // Problem: Type erasure! List<String> and List<Integer> are the SAME at runtime
    // Both compiled to List → compiler error!

    // BETTER: Different method names
    public void addStrings(List<String> items) { }
    public void addIntegers(List<Integer> items) { }
}
```

### Interview Answer

> "Yes, overloading with `String` and `Object` is valid. When you pass `null`, Java applies the **most specific applicable method** rule — it picks the overload whose parameter type is most specific, meaning closest to the bottom of the inheritance tree. Since `String` extends `Object`, `String` is more specific, so the `String` overload gets called even though both accept `null`.
>
> The gotcha: when two overloads are **equally specific** — like `String` and `Integer` (neither is a subtype of the other) — then `null` is ambiguous and you get a compile error. You have to cast: `print((String) null)`.
>
> More broadly, Java's overload resolution has a priority: exact type match > widening (int to long) > autoboxing (int to Integer) > varargs. Widening beats autoboxing, which is why passing `5` to an overloaded method with both `long` and `Integer` parameters chooses `long`.
>
> As API designers, we avoid overloading primitives with their boxed types or generic types that differ only in type parameters, because type erasure makes them ambiguous at runtime."
>
> *Likely follow-up: "What if I have three overloads: String, CharSequence, and Object — which gets called with null?"* → Answer: `String`, still most specific.

---

## Q3. Heap and Stack Memory Model

### Concept
The JVM uses two primary memory regions: the **Heap** for objects that persist until no references exist, and the **Stack** for method execution frames that are automatically created and destroyed with each method call.

### Simple Explanation
- **Stack** = a pile of trays at a cafeteria. When a method starts, you take a fresh tray (stack frame). When the method finishes, you put the tray back (frame destroyed). Super organized, automatic cleanup.
- **Heap** = a warehouse. Objects go in whenever you `new` them. They stay until nobody references them anymore. The garbage collector periodically cleans up the unreferenced garbage.

### How It Works

#### Stack Frame Structure

```
Each method call creates a Stack Frame:
┌─────────────────────────────────────┐
│ Local Variables (method parameters, │
│                 local vars)         │
│                                     │
│ Operand Stack (intermediate values  │
│                for computation)     │
│                                     │
│ Reference to Constant Pool          │
└─────────────────────────────────────┘

Stack grows DOWNWARD (newer frames on top):
┌──────────────────┐
│ Frame: main()    │ ← Bottom (main starts first)
├──────────────────┤
│ Frame: process() │ ← Middle (called from main)
├──────────────────┤
│ Frame: helper()  │ ← Top (called from process)
└──────────────────┘

When helper() returns:
- Its frame is popped (destroyed)
- Local vars and refs in helper frame → gone

When process() returns:
- Its frame is popped
- Local vars and refs in process frame → gone
- If no other ref to object → eligible for GC
```

#### Memory Layout Example

```java
public class MemoryDemo {
    private static final String APP_NAME = "MyApp";  // Metaspace, not heap

    public void process() {
        int count = 5;                      // Stack: primitive
        User user = new User("Alice");      // Stack: ref, Heap: object
        String name = user.getName();       // Stack: ref, Heap: "Alice" (string pool)
        
        helper(count);
        // After helper() returns: helper's stack frame DESTROYED
        // user still references object on heap (not GC'd yet)
    }

    private void helper(int x) {
        String temp = "temp";  // Stack: ref, Heap: "temp" string
        // When helper() returns: temp ref destroyed
        // "temp" string eligible for GC (if no other refs)
    }

    public static void main(String[] args) {
        MemoryDemo demo = new MemoryDemo();  // Heap: MemoryDemo instance
        demo.process();
        // After process() returns: process() frame destroyed
        // demo still references object (in scope in main)
    }
}

Memory Layout During Execution:
┌─ STACK (Thread) ─────────┐  ┌─ HEAP ────────────────────┐
│ main() frame:            │  │ User { name="Alice" }     │
│  count: 5 ─────────────┬─┼─→│                           │
│  user: 0x1000 ────────┼─┼─→│ String "Alice"            │
│                        │ │  │                           │
│ process() frame:       │ │  │ String "temp" (eligible)  │
│  name: 0x2000 ────────┼─┼─→│                           │
│                        │ │  │ MemoryDemo instance       │
│ helper() frame:        │ │  │                           │
│  temp: 0x3000 ────────┼─┼─→│                           │
└────────────────────────┘  └───────────────────────────┘
```

#### Stack Destruction

```java
public void example() {
    User user = new User("Bob");     // Stack: ref, Heap: object @ 0x5000
    {
        User temp = new User("Carol");   // Stack: ref, Heap: object @ 0x6000
        System.out.println(temp);        // Carol
    } // temp reference DESTROYED (out of scope)
    // 0x6000 object now eligible for GC (temp was only ref)
    
    System.out.println(user);        // Bob (user still in scope)
    // user reference still alive
} // method returns
// user reference DESTROYED
// 0x5000 object now eligible for GC
```

### Stack Overflow & Heap Overflow

```java
// StackOverflowError: Stack is finite (default ~1MB per thread)
static void recursion(int n) {
    recursion(n + 1);  // Infinite recursion → stack frame per call
}                      // Stack exhausted → StackOverflowError

// OutOfMemoryError (Java heap space): Heap exhausted
public void oomExample() {
    List<byte[]> list = new ArrayList<>();
    while (true) {
        list.add(new byte[1024 * 1024]);  // 1MB chunks
    }  // Heap eventually full → OutOfMemoryError
}
```

### Real Project Usage

**Memory Leak Pattern (Holding References):**
```java
public class EventListener {
    private List<byte[]> cache = new ArrayList<>();  // Growing list

    public void onEvent(Event e) {
        cache.add(new byte[1024 * 1024]);  // Add to cache
        // Never removed → heap grows indefinitely
        // Even after event processed, cache holds reference
    }
    
    // FIX: Bounded cache with eviction
    private LinkedHashMap<String, byte[]> boundedCache = 
        new LinkedHashMap<String, byte[]>(100) {
            protected boolean removeEldestEntry(Map.Entry eldest) {
                return size() > 100;  // Evict oldest when size > 100
            }
        };
}
```

**Stack Overflow in Recursive Algorithms:**
```java
// Deep recursion can overflow stack
static int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // Each call = new frame on stack
}

factorial(10000);  // StackOverflowError (10K frames on 1MB stack)

// FIX: Iterative approach
static int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;  // No new stack frames
    }
    return result;
}

// Or: Tail recursion (compiler optimization, less common in Java)
```

### Interview Answer

> "Stack and Heap are two distinct memory regions with different lifetimes and purposes. The Stack stores method frames (local variables, parameters, method calls). Each method call pushes a frame onto the stack; when the method returns, the frame is immediately popped and destroyed. This is very efficient and automatic.
>
> The Heap stores all object instances created with `new`. Objects live as long as any reference to them exists. When an object has no references, the garbage collector marks it as eligible for collection and eventually reclaims its memory.
>
> Key differences:
> - **Allocation:** Stack is linear and automatic (LIFO). Heap is fragmented; GC must manage.
> - **Lifetime:** Stack variables destroyed when method returns. Heap objects live until no references.
> - **Size:** Stack is smaller and finite (~1MB per thread). Heap is larger (controlled by -Xmx).
> - **Errors:** StackOverflowError from infinite recursion. OutOfMemoryError from heap exhaustion.
>
> This is why pass-by-value for objects works: the reference lives on the stack (isolated per frame), but it points to the same object on the heap (shared). Mutations of the object are visible, but reassigning the reference is not."
>
> *Likely follow-up: "What causes a memory leak in Java?"* → Holding references to objects that are no longer needed; example: static collections, listeners without unsubscribe.

---

## Q4. String Pool and Memory

### Concept
The String pool is a special memory region where Java caches string literals to save memory and improve performance. Understanding where strings live (stack vs pool vs heap) is crucial for optimizing memory usage.

### Simple Explanation
The String pool is like a library: instead of 1000 books with identical content stored separately, the library keeps one copy and gives everyone a reference to it. Java does this for string literals — "hello" is stored once, and multiple `String s = "hello"` variables reference the same object.

### How It Works

```java
String s1 = "hello";        // Looks in pool; not found → creates "hello" in pool
String s2 = "hello";        // Looks in pool; found → returns ref to same object
System.out.println(s1 == s2);  // true (SAME object!)

String s3 = new String("hello");  // Explicitly creates NEW object on heap
System.out.println(s1 == s3);  // false (different objects)
System.out.println(s1.equals(s3));  // true (same content)

String s4 = s3.intern();     // Manually add s3 to pool or get pool ref
System.out.println(s1 == s4);  // true (s4 now points to pool object)
```

**String Pool Location:**
- **Java 7+:** On the Heap (not Metaspace)
- **Java 6 and earlier:** In Metaspace (separate, limited memory)
- **Implications:** Java 7+, you don't hit "String pool full" errors

### Key Gotchas

#### Gotcha 1: String Concatenation

```java
String a = "hello";
String b = "hello";
String c = a + b;  // Creates NEW string on heap, NOT in pool

System.out.println(a == b);  // true (both in pool)
System.out.println(a == c);  // false (c is new heap object)

// Compiler optimization:
String d = "hello" + "hello";  // Compiler optimizes to "hellohello"
// "hellohello" added to pool → future refs use pool
```

#### Gotcha 2: intern() is Slow in Production

```java
String user = getUserFromDatabase();  // Not in pool (from DB)
String pooled = user.intern();  // Thread-safe but EXPENSIVE
// intern() acquires lock on pool, searches (O(n)), inserts if needed

// Use case: When you have few unique values and memory is critical
// Don't use: When you have high cardinality strings (user IDs, emails)
```

### Interview Answer
> "The String pool is an optimization: identical string literals share a single object reference, saving memory. In Java 7+, the pool lives on the heap (not Metaspace), so you don't hit pool-full errors. The key gotcha is that string concatenation like `a + b` creates a NEW heap object, not a pool object — only compile-time constants are pooled. intern() is thread-safe but expensive, acquiring a lock on the pool for O(n) lookup; use it only for low-cardinality strings where memory is critical. For most code, just use equals() for comparison; don't rely on == for strings."
>
> *Likely follow-up: "Why does string concatenation in a loop create new objects?"*

---

## Q5. Stack vs Heap — Stack Memory and Garbage Collection

### Concept
The JVM is the runtime engine that executes Java bytecode. Understanding its architecture helps explain memory layout, garbage collection, JIT compilation, and class loading.

### Simple Explanation
JVM is a **universal translator**:
1. You write Java code
2. Compiler converts to bytecode (platform-neutral `.class` files)
3. JVM on each machine translates that bytecode to native instructions
4. Result: "Write Once, Run Anywhere"

### JVM Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│ CLASSLOADER SUBSYSTEM                                   │
│ Loading → Linking (verify/prepare/resolve) →            │
│ Initialization                                          │
├─────────────────────────────────────────────────────────┤
│ RUNTIME DATA AREAS                                      │
│ - Method Area (Metaspace): class metadata, bytecode    │
│ - Heap: all object instances                           │
│ - Stack (per thread): method frames                    │
│ - PC Register: current bytecode instruction            │
│ - Native Method Stack: JNI calls                       │
├─────────────────────────────────────────────────────────┤
│ EXECUTION ENGINE                                        │
│ - Interpreter: executes bytecode line-by-line          │
│ - JIT Compiler: compiles hot methods to native code    │
│ - Garbage Collector: reclaims unused memory            │
└─────────────────────────────────────────────────────────┘
```

### ClassLoader Subsystem

**Three Phases:**
1. **Loading:** Finds `.class` file (Bootstrap → Extension → Application classloader)
2. **Linking:**
   - Verify: checks bytecode is valid
   - Prepare: allocates memory for static variables (default values)
   - Resolve: replaces symbolic references with direct references
3. **Initialization:** runs static blocks, assigns static field values

**Classloader Hierarchy (Parent Delegation):**
```
Bootstrap ClassLoader (core Java: java.lang, java.util)
  ↓
Extension ClassLoader (JDK extensions: lib/ext)
  ↓
Application ClassLoader (your classpath)

When Application ClassLoader is asked for a class:
1. Ask parent (Extension)
2. Parent asks ITS parent (Bootstrap)
3. Bootstrap tries to load → if not found, Extension tries
4. If Extension fails, Application tries
5. If Application fails → ClassNotFoundException
```

**Why parent delegation?** Prevents malicious code from overriding `java.lang.String` with a custom version.

### JIT Compilation

```java
// JVM doesn't interpret every bytecode instruction forever

// First execution:
method() called → Interpreter executes (slow, ~10x slower than native)
Counter incremented (method calls tracked)

// After 10K executions (threshold):
Counter > threshold → JIT Compiler kicks in
method() compiled to NATIVE MACHINE CODE → stored in Code Cache
Future calls use native code (100x faster than interpreted)

// Tiered compilation:
C1 Compiler: Fast compilation, moderate optimization (used first)
C2 Compiler: Slower compilation, heavy optimization (used for hot hotspots)
```

**Code Cache:** Native memory where JIT-compiled code is stored. If full → JIT stops compiling → performance degrades.

---

## Key Gotchas & Common Mistakes

### ❌ Gotcha 1: Confusing == with .equals()

```java
// Primitives: == compares VALUE
int a = 5;
int b = 5;
System.out.println(a == b);  // true

// Objects: == compares REFERENCE (heap address)
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);  // false — different objects on heap
System.out.println(s1.equals(s2));  // true — same content

// Integer caching (-128 to 127):
Integer x = 10;
Integer y = 10;
System.out.println(x == y);  // true (cached)

Integer a = 128;
Integer b = 128;
System.out.println(a == b);  // false (not cached)
```

### ❌ Gotcha 2: String Immutability Misconception

```java
String s = "hello";
s = s + " world";  // Does NOT mutate "hello"; creates NEW string "hello world"

// Original "hello" unchanged; s now points to new object
// If "hello" had no other references → eligible for GC immediately

String s2 = "hello";  // May point to pool or new object (compiler dependent)
```

### ❌ Gotcha 3: Defensive Copy Failure

```java
public class User {
    private List<String> tags;

    public List<String> getTags() {
        return tags;  // WRONG: caller can mutate
    }
    // Caller: user.getTags().add("hacker")
}

// FIX:
public List<String> getTags() {
    return Collections.unmodifiableList(tags);
    // Or: return List.copyOf(tags);  (Java 10+)
}
```

### ❌ Gotcha 4: Stack Overflow from Deep Recursion

```java
// Maximum stack depth before StackOverflowError:
// Default stack size ~1MB
// Each frame ~100-200 bytes
// Max recursion depth ~5000-10K calls

static void recursion(int n) {
    System.out.println(n);
    recursion(n + 1);  // Eventually StackOverflowError
}

// Use iteration instead, or increase stack size: -Xss2m
```

### ❌ Gotcha 5: Memory Leak from Static Collections

```java
public class Cache {
    private static Map<String, byte[]> cache = new HashMap<>();  // DANGER

    public static void cache(String key, byte[] data) {
        cache.put(key, data);  // Keeps reference forever
    }
}

// Objects added to cache never get GC'd (static ref is always alive)
// Fix: Use bounded cache with eviction, or WeakHashMap
```

### Interview Answer
> "The stack is a fixed-size region — exceeding it causes StackOverflowError. Stack memory is freed automatically when a method exits. Garbage collection does NOT manage the stack — only the heap. Common stack pitfalls: forgetting that static collections live forever, not realizing defensive copies consume stack memory when passed parameters, deep recursion overflowing the stack. The heap is managed by GC — unreachable objects get collected. A memory leak occurs when you keep references to objects that should be discarded, like a static cache that never evicts, or a listener never unregistered."
>
> *Likely follow-up: "How would you detect a memory leak in a long-running application?"*

---

## Q6. OOP Principles with Real Examples

### The Four Pillars

```java
// ─── 1. ENCAPSULATION ────────────────────────────────────────────────────────
// Concept: Bundle data + behaviour. Hide internal state. Expose only what's needed.
// Real: BankAccount — balance is private; only deposit/withdraw are public.

public class BankAccount {
    private double balance;  // Can't be touched directly from outside

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new InsufficientFundsException();
        this.balance -= amount;
    }

    public double getBalance() { return balance; }
    // No setBalance() — prevents external corruption
}

// ─── 2. INHERITANCE ──────────────────────────────────────────────────────────
// Concept: Child class inherits + extends parent behaviour.
// WARNING: Prefer composition over inheritance for flexibility.

abstract class Notification {
    protected String recipient;
    protected String message;

    abstract void send();  // Each type knows HOW to send

    void log() {
        System.out.println("Sending to: " + recipient);
    }
}

class EmailNotification extends Notification {
    @Override
    void send() {
        log();  // Inherits from parent
        emailService.send(recipient, message);
    }
}

class SMSNotification extends Notification {
    @Override
    void send() {
        smsGateway.send(recipient, message);
    }
}

// ─── 3. POLYMORPHISM ──────────────────────────────────────────────────────────
// Concept: Same interface, different behaviour depending on actual type.
//   - Compile time: method overloading
//   - Runtime: method overriding (dynamic dispatch)

Notification notif;
notif = new EmailNotification();
notif.send();   // Calls EmailNotification.send() — decided at RUNTIME

List<Notification> notifications = List.of(
    new EmailNotification(),
    new SMSNotification(),
    new PushNotification()
);
notifications.forEach(Notification::send);  // Each sends its own way

// ─── 4. ABSTRACTION ──────────────────────────────────────────────────────────
// Concept: Hide complexity behind a clean contract. "What" not "how".

// Interface = pure contract (what it does)
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    void refund(String transactionId);
}

// Implementations hide the complexity
public class StripePaymentProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // 50 lines of Stripe API calls, retries, error handling...
    }
}
// Caller only knows about PaymentProcessor — not Stripe internals
```

### Interview Answer
> "Encapsulation is about protection and control — bank account balance is private, operations go through validated methods. Inheritance avoids code duplication — a base `Notification` class with email, SMS, push children. Polymorphism means you can treat all notification types uniformly through the base reference — the right `send()` is called at runtime. Abstraction hides complexity behind an interface — a payment service is just `process()` and `refund()`, callers don't know if it's Stripe or PayPal underneath.
>
> In practice, I use encapsulation heavily for domain objects, I favour composition over inheritance for flexibility, and I rely on interfaces for abstraction across service layers."
>
> *Likely follow-up: "When would you use composition instead of inheritance?"*

---

## Q7. String Pool Internals — Memory Location and Location Changes Across Java Versions

### Concept
The **String Pool** is a cache of interned Strings at the memory level. Before Java 7, it lived in the PermGen heap. Java 7+ moved it to the regular Heap. This change affects memory management and GC behavior.

### Simple Explanation
String pool is like a postal service directory:
- **Java 6 (PermGen):** Directory is in a special storage room (permanent generation). When you use a string, they check and reuse if it exists. But PermGen never shrinks, so old strings waste space forever.
- **Java 7+ (Heap):** Directory moved to regular mailroom (Heap). Strings are now GC-eligible when no longer needed. More memory efficient.

### The Problem: Java 6 PermGen String Pool

```java
// Java 6: String pool in PermGen (NOT garbage collected)
// PermGen has fixed size (~64MB default), causes OutOfMemoryError if exhausted

String[] strings = new String[1_000_000];
for (int i = 0; i < strings.length; i++) {
    // Each new literal goes to PermGen pool if interned
    strings[i] = ("String_" + i).intern();  // ← Fills PermGen
}
// Error: OutOfMemoryError: PermGen space
// CANNOT be fixed with -Xmx (heap size) — PermGen is separate!
// Must use: -XX:PermSize=128m -XX:MaxPermSize=256m
```

### The Solution: Java 7+ Heap String Pool

```java
// Java 7+: String pool moved to Heap
// GC can now clean up unused interned strings!

// Same code as above now works (mostly)
String[] strings = new String[1_000_000];
for (int i = 0; i < strings.length; i++) {
    strings[i] = ("String_" + i).intern();  // Heap pool, GC-eligible
}
// Still memory-intensive, but now:
// 1. Reusable beyond pool size (GC cleans unused)
// 2. -Xmx controls pool size

// Remove reference → String eligible for GC (Java 7+)
String interned = "hello".intern();
interned = null;  // ← Now GC can collect it (not possible in Java 6)
```

### String Creation Paths and Pool Location

```java
// Path 1: Literal string → goes to pool (Heap in Java 7+)
String s1 = "hello";  // Created in pool
String s2 = "hello";  // Points to SAME pool entry
System.out.println(s1 == s2);  // true (same reference)

// Path 2: new String() → creates on Heap, separate from pool
String s3 = new String("hello");  // New object on Heap
String s4 = new String("hello");  // Different object on Heap
System.out.println(s3 == s4);  // false (different objects but same content)
System.out.println(s3 == s1);  // false (s3 is on heap, s1 in pool)

// Path 3: Concatenation with literals → optimized at compile time
String s5 = "hello" + " " + "world";  // Compiler optimizes to single literal
// Equivalent to: String s5 = "hello world";  // In pool

// Path 4: Concatenation with variables → created on Heap (not in pool)
String prefix = "hello";
String s6 = prefix + " " + "world";  // Runtime concat on Heap, not pool
System.out.println(s6 == "hello world");  // false (s6 not in pool!)

// Path 5: intern() — explicitly add to pool
String s7 = new String("hello");  // On Heap
String s8 = s7.intern();  // Adds to pool if not there, returns pool reference
System.out.println(s8 == s1);  // true (both now point to pool entry)
System.out.println(s7 == s8);  // false (s7 still on Heap, s8 in pool)
System.out.println(s8 == "hello");  // true (s8 in pool)
```

### Performance: Literal vs new String()

```java
// ✅ FAST: Literals (reused from pool)
for (int i = 0; i < 1_000_000; i++) {
    String s = "constant";  // ← Reused from pool
}
// Time: <1ms (pool lookup cost)

// ❌ SLOW: new String() (separate objects each time)
for (int i = 0; i < 1_000_000; i++) {
    String s = new String("constant");  // ← New object each loop
}
// Time: 100ms+ (object creation cost)

// ❌ SLOW: Concatenation with variables
for (int i = 0; i < 1_000_000; i++) {
    String s = "prefix_" + i;  // ← concatenation creates new String each time
}
// Time: 100ms+ (string building cost)

// ✅ CORRECT: StringBuilder for loops
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1_000_000; i++) {
    sb.append("prefix_").append(i).append(" ");
}
String result = sb.toString();  // Single toString() at end
// Time: <10ms (efficient)
```

### String.intern() Use Cases and Risks

```java
// ✓ GOOD: Known set of strings (small, immutable)
public enum Protocol {
    HTTP("http"),
    HTTPS("https"),
    FTP("ftp");
    
    private final String value;  // Pool usage: reuse across JVM
    Protocol(String value) {
        this.value = value.intern();  // Worth pooling
    }
}

// ✓ GOOD: Frequently used strings in multi-threaded system
public class CommonStrings {
    public static final String TRUE_STR = "true".intern();  // Pool once
    public static final String FALSE_STR = "false".intern();
    
    // Code checking: if (result.intern() == TRUE_STR) { ... }
    // Faster than .equals() in tight loops
}

// ❌ DANGEROUS: Untrusted input (can cause pool DOS)
Map<String, Integer> userCache = new HashMap<>();

public void cacheUserData(String userId) {
    // If attacker crafts many unique strings, intern() fills pool!
    String interned = userId.intern();  // ← Could DOS the pool
    userCache.put(interned, 1);
}

// ❌ DANGEROUS: Large strings
String largeString = readFileContents("large.txt").intern();  // Wastes pool memory
// Large strings should NOT be pooled

// ✓ SAFE: Bounded pooling with WeakHashMap
private static final Map<String, String> customPool = new WeakHashMap<>();

public String pool(String str) {
    String existing = customPool.putIfAbsent(str, str);
    return existing != null ? existing : str;
    // Weak references → GC can clean up unused pooled strings
}
```

### Interview Answer: String Pool Evolution

```
"Java 6 had the String pool in PermGen memory, which was never garbage collected. 
This caused OutOfMemoryError: PermGen space when too many strings were interned, 
and you had to separately tune -XX:PermSize and -XX:MaxPermSize.

Java 7 moved the String pool to the regular Heap, making it garbage-collectable. 
Now, interned strings can be collected when no longer referenced, 
and the -Xmx heap size parameter controls it.

The change means:
1. String literals and .intern() use Heap memory
2. GC can reclaim pooled strings (Java 7+ only)
3. No separate PermGen tuning needed (simplified ops)
4. But also: possible pool fragmentation if strings are constantly interned then discarded

In practice, explicit .intern() is rare today — JVM optimizes most cases. 
But you should know the difference when tuning memory or debugging OutOfMemoryError."
```

### Gotcha: String Comparison Best Practice

```java
// ❌ UNRELIABLE: Using == with strings
String user = getUsername();  // From user input, network, database
if (user == "admin") {  // ← WRONG! Might not work
    grantAdminAccess();  // Could be skipped even if user == "admin"
}
// Why? user might not be in pool (was concatenated, built, or from external source)

// ✓ CORRECT: Always use .equals()
if (user.equals("admin")) {
    grantAdminAccess();  // ✓ Works reliably
}

// == works ONLY for:
// 1. String literals (both in pool)
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2);  // true (both in pool)

// 2. Explicitly interned strings
String s3 = new String("hello").intern();
System.out.println(s3 == "hello");  // true (both interned)

// But never rely on == for string value comparison!
```

---

## Related Concepts & Follow-ups

### Call by Value Follow-up
**Q: "Can you make a method return a modified primitive?"**
A: No — reassignment is local. But you can wrap it: `int[] box = {5}; modify(box); // box[0] changed`

### Method Overloading Follow-up
**Q: "Does method overloading affect performance?"**
A: No — resolution is compile-time (static). Zero runtime cost.

### Memory Model Follow-up
**Q: "What causes OutOfMemoryError and how do you fix it?"**
A: Heap exhaustion (too many objects, too large objects, memory leak). Fix: Increase -Xmx, fix memory leaks (remove refs), optimize algorithms to use less memory.

**Q: "What's a memory leak in Java given there's garbage collection?"**
A: Holding references to objects you no longer need. GC can only collect objects with zero references. Examples: static collections, event listeners without unsubscribe, cached objects not evicted.

### String Pool Follow-up
**Q: "When should I use .intern()?"**
A: For small, known sets of strings (enums, protocols). NOT for user input, large strings, or untrusted data (DOS risk). Most cases don't need it.

**Q: "Why did Java move String pool to Heap?"**
A: PermGen was never GC'd, wasting memory. Heap GC makes it automatic. Side effect: now controlled by -Xmx instead of separate PermGen tuning.

---

## Summary: When to Use Each Concept

| Concept | When | Example |
|---------|------|---------|
| Call by value | Method design, debugging mutations | Defensive copy pattern |
| Overloading | API design, flexibility | Builder pattern |
| Stack memory | Understanding lifecycle | Local variables, method params |
| Heap memory | Understanding persistence | Objects, collections |
| String pool | Performance optimization | Use literals, not `new` |
| String.intern() | Known, small string sets | Enums, protocols |
| JIT compilation | Performance tuning | Why Java warms up over time |

