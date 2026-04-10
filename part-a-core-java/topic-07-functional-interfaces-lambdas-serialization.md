# Topic 7 — Functional Interfaces, Lambdas, Serialization & Advanced Patterns

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer

---

## Q1. Functional Interfaces — SAM Contract and @FunctionalInterface

### Concept
A **Functional Interface** has exactly one abstract method (**S**ingle **A**bstract **M**ethod contract).
It's the contract that enables lambda expressions—the compiler can infer which method a lambda implements.

### Simple Explanation
A functional interface is like a vending machine with ONE button. When you press it, only ONE action happens. A lambda is the action you define: "When pressed, dispense coffee." The machine knows which action because there's only one button.

### Code

```java
// ✓ Functional Interface: exactly 1 abstract method
@FunctionalInterface  // Compiler verifies this!
public interface Predicate<T> {
    boolean test(T t);  // One abstract method
    
    default boolean and(Predicate<? super T> other) {  // default is OK
        return t -> test(t) && other.test(t);
    }
    
    static <T> Predicate<T> isEqual(Object target) {  // static is OK
        return t -> target.equals(t);
    }
}

// ✓ Valid: Lambda because interface has exactly 1 abstract method
Predicate<String> isEmpty = str -> str.isEmpty();

// ❌ Invalid Functional Interface: 2 abstract methods
@FunctionalInterface
public interface TwoAbstractMethods {
    void method1();
    void method2();  // ← Second abstract method violates SAM
}
// Compiler error: "Multiple non-overriding abstract methods"

// Important: @FunctionalInterface is OPTIONAL but RECOMMENDED
// Even without annotation, single abstract method = functional interface
public interface ImplicitFunctional {
    void doSomething();  // Functional, even without @FunctionalInterface
}

ImplicitFunctional task = () -> System.out.println("Hello");  // ✓ Works
```

### Rules

```java
// 1. Can inherit abstract methods from Object (doesn't count)
@FunctionalInterface
public interface ObjectMethod {
    void apply();
    
    // These DON'T break SAM (inherited from Object, not new abstract)
    boolean equals(Object o);      // OK: overriding Object.equals()
    String toString();              // OK: overriding Object.toString()
}

// 2. Multiple default methods are OK
@FunctionalInterface
public interface MultipleDefaults {
    void execute();
    
    default void before() {}
    default void after() {}
    default void setup() {}
}

// 3. Multiple static methods are OK
@FunctionalInterface
public interface MultipleStatic {
    void doWork();
    
    static void prepare() {}
    static String format(Object o) { return o.toString(); }
}

// Test:
MultipleDefaults task = () -> System.out.println("work");
task.before();   // Call default
task.execute();  // Call abstract
task.after();    // Call default
```

### Interview Answer

> "A functional interface has exactly one abstract method — the @FunctionalInterface annotation enforces this at compile time. It was introduced in Java 8 to enable lambda expressions. The four cornerstone built-ins are Predicate for testing, Function for transforming, Consumer for side effects, and Supplier for providing values. I use them constantly — Predicate in stream filters, Function in map(), Supplier for lazy initialisation in Optional's orElseGet(), and Consumer in forEach."
>
> *Likely follow-up: "Can an interface with two abstract methods but one inherited from Object be a functional interface?"*

---

## Q2. The 4 Core Functional Interfaces — Predicate, Function, Consumer, Supplier

### Concept
Java provides 4 fundamental functional interfaces in `java.util.function` package.
They cover the 90% of use cases for lambdas.

### The 4 Interfaces

```java
import java.util.function.*;

// 1. Predicate<T> — Test condition, return boolean
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}

// Usage: Filter, validation
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Person> isAdult = p -> p.getAge() >= 18;

if (notEmpty.test("hello")) {  // ✓ true
    System.out.println("Not empty");
}

// 2. Function<T, R> — Transform T into R
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

// Usage: Transformation, mapping
Function<String, Integer> stringLength = s -> s.length();
Function<Integer, String> toString = n -> String.valueOf(n);
Function<Person, String> getName = p -> p.getName();

Integer length = stringLength.apply("hello");  // 5
String numStr = toString.apply(42);            // "42"

// 3. Consumer<T> — Accept T, return void (side-effect)
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

// Usage: Side effects, logging, printing
Consumer<String> printer = System.out::println;
Consumer<Integer> logger = i -> System.out.println("Value: " + i);
Consumer<Person> save = p -> database.insert(p);

printer.accept("hello");
logger.accept(42);
save.accept(new Person("John"));

// 4. Supplier<T> — Return T, no input (factory)
@FunctionalInterface
public interface Supplier<T> {
    T get();
}

// Usage: Lazy initialization, factories, defaults
Supplier<LocalDateTime> now = () -> LocalDateTime.now();
Supplier<Random> randomGen = () -> new Random();
Supplier<List<String>> emptyList = () -> new ArrayList<>();

LocalDateTime current = now.get();
Random rand = randomGen.get();
List<String> list = emptyList.get();

// Lazy vs Eager: Supplier delays computation
LocalDateTime eager = LocalDateTime.now();  // ✗ Computed immediately
Supplier<LocalDateTime> lazy = () -> LocalDateTime.now();  // ✓ Computed on .get()
```

### Real Production Usage

```java
// Predicate: Stream filter
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
Predicate<Integer> isEven = n -> n % 2 == 0;
numbers.stream()
    .filter(isEven)  // Uses Predicate
    .forEach(System.out::println);  // 2, 4

// Function: Stream map
numbers.stream()
    .map(n -> n * 2)  // Function: int → int*2
    .forEach(System.out::println);  // 2, 4, 6, 8, 10

// Consumer: Stream forEach
numbers.stream()
    .forEach(System.out::println);  // Consumer: accept each element

// Supplier: Optional.orElseGet()
Optional<String> value = Optional.empty();
String result = value.orElseGet(() -> "default");  // Supplier only called if empty

// Combining: Stream collect with predicate and function
List<Integer> evens = numbers.stream()
    .filter(isEven)           // Predicate
    .collect(Collectors.toList());  // Supplier inside
```

### Interview Answer

> "Before Java 8, passing behaviour required anonymous class boilerplate. Functional interfaces standardise the contract: one abstract method = one piece of behaviour. The key practical differences: Predicate composes with and/or/negate, Function chains with andThen/compose, Consumer chains side effects with andThen, and Supplier is about laziness — specially useful with orElseGet() because unlike orElse(), the Supplier only executes when the value is actually needed."
>
> *Likely follow-up: "What is the difference between andThen and compose in Function?"*

---

## Q3. Method References — Static, Bound, Unbound, Constructor

### Concept
**Method references** are shorthand syntax for lambdas that simply delegate to an existing method.
Format: `target::methodName`

### The 4 Types

```java
// Type 1: Static Method Reference
// Syntax: ClassName::staticMethod
Function<String, Integer> toInt = Integer::parseInt;  // Instead of: s -> Integer.parseInt(s)
Integer num = toInt.apply("42");  // 42

// Real usage:
Stream<String> strings = Stream.of("1", "2", "3");
Stream<Integer> ints = strings.map(Integer::parseInt);  // Method reference

// Type 2: Bound Instance Method Reference
// Syntax: instance::instanceMethod (bound to specific instance)
String str = "hello";
Supplier<Integer> getLength = str::length;  // Bound to "hello"
Integer length = getLength.get();  // 5

Person john = new Person("John", 30);
Supplier<String> getName = john::getName;  // Bound to john instance
String name = getName.get();  // "John"

// Real usage:
List<Person> people = Arrays.asList(john, new Person("Jane", 25));
people.stream()
    .map(p -> p::getName)  // Each maps to its own getName method
    .forEach(System.out::println);

// Type 3: Unbound Instance Method Reference
// Syntax: ClassName::instanceMethod (NOT bound to instance)
// First parameter becomes the instance
Function<String, Integer> toLength = String::length;  // Unbound to any String
Integer len = toLength.apply("hello");  // 5

BiFunction<String, String, Boolean> equals = String::equals;  // (s1, s2) -> s1.equals(s2)
Boolean result = equals.apply("hello", "hello");  // true

// Key: Unbound passes the instance as first parameter
// Real usage:
Stream<String> words = Stream.of("hello", "world", "java");
words.forEach(System::out.println);  // System.out.println treated as unbound: accept(String)

// Type 4: Constructor Reference
// Syntax: ClassName::new
Supplier<ArrayList> createList = ArrayList::new;  // () -> new ArrayList()
ArrayList list = createList.get();

Function<String, Integer> toInteger = Integer::new;  // String -> new Integer(String)
Integer value = toInteger.apply("42");

// Real usage: Stream.map() with constructor
Stream<String> words = Stream.of("one", "two", "three");
Stream<Integer> lengths = words.map(String::length);

// Collecting with constructor:
List<String> stringList = Arrays.asList("1", "2", "3");
List<Integer> intList = stringList.stream()
    .map(Integer::new)  // String → Integer constructor
    .collect(Collectors.toList());
```

### Comparison Table

```
Type                | Syntax                    | Example
--------------------|---------------------------|----------------------------
Static              | Class::staticMethod       | Integer::parseInt
Bound Instance      | instance::method          | "hello"::length
Unbound Instance    | Class::instanceMethod     | String::length
Constructor         | Class::new                | ArrayList::new

Key Difference:
- Bound: Method applied to known instance
- Unbound: Instance passed as first parameter
- Static: No instance needed
- Constructor: Creates new instance
```

### Interview Answer

> "Method references are syntactic sugar for lambdas that call a single existing method. Four forms: static (Class::method), instance (obj::method), bound (this::method), and constructor (Class::new). They're more readable than lambdas when you're just delegating to another method. The compiler generates invokedynamic bytecode, which can be better optimised than lambda bytecode because the method is already known."
>
> *Likely follow-up: "What's the difference between a method reference and a lambda?"*

---

## Q4. Effectively Final — Capturing Variables in Lambdas

### Concept
Lambdas can capture (use) variables from enclosing scope ONLY if they are **effectively final**:
Not modified after capture.

### Code

```java
// ✓ Effectively Final: Variable never reassigned
int x = 10;
Supplier<Integer> supplier = () -> x;  // Captures x
System.out.println(supplier.get());  // 10

// ❌ NOT effectively final: Variable reassigned
int y = 10;
y = 20;  // ← Reassignment!
Supplier<Integer> supplier2 = () -> y;  // ✗ Compile error: "y is not effectively final"

// If declared final, OK to capture (obviously)
final int z = 10;
Supplier<Integer> supplier3 = () -> z;  // ✓ Explicitly final

// Mutable objects: Assignment restriction doesn't apply to field changes
List<String> list = new ArrayList<>();  // Not final, but can capture
list.add("item");  // Adding to list is OK
Consumer<String> addToList = s -> list.add(s);  // ✓ Captures list reference

// However, the reference itself can't change:
list = new ArrayList<>();  // ✗ Would break effectively final
Consumer<String> addToList2 = s -> list.add(s);  // ✗ Compile error

// Capturing from different scopes
class MyClass {
    int classField = 10;  // Class scope
    
    void method() {
        int localVar = 20;  // Method scope
        
        Supplier<Integer> sum = () -> classField + localVar;  // Captures both!
    }
}
```

### Why Effectively Final?

```java
// Memory: Lambdas capture VALUES, not REFERENCES (for primitives)
// So compiler ensures value doesn't change after capture

// Internals: Captured variables copied to lambda synthetic class fields
// for value types (int, boolean), captured at compile time
// Compiler needs to ensure the value doesn't change

// Example: Compiler transforms this...
int x = 10;
Functional<Integer> f = () -> x;

// ...into something like this internally...
class SyntheticLambda implements Functional<Integer> {
    private final int x;  // Captured value stored as final field
    SyntheticLambda(int x) { this.x = x; }
    public Integer apply() { return x; }
}
Functional<Integer> f = new SyntheticLambda(10);
```

### Interview Answer

> "Captured variables must be effectively final — not reassigned after capture. This is because lambdas are translated to synthetic methods that receive captured variables as parameters, so reassigning them would cause confusion about which value to use. For mutable objects in scope, all mutable fields accessed must be Serializable if you ever serialize the lambda (e.g., to cache or send to another JVM)."
>
> *Likely follow-up: "Why can't a lambda access a mutable local variable?"*

---

## Q5. Functional Interfaces in Action — Streams, Optional, and method()

### Concept
Functional interfaces are the backbone of functional programming in Java.
Streams, Optional, Comparators all use them.

### Streams + Functional Interfaces

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6);

// filter() : Predicate<T>
Stream<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0);  // Predicate: test condition

// map() : Function<T, R>
Stream<String> strings = numbers.stream()
    .map(n -> "Number: " + n);  // Function: T→R transformation

// forEach() : Consumer<T>
numbers.stream()
    .forEach(System.out::println);  // Consumer: side effect

// collect() uses Supplier internally
List<Integer> collected = numbers.stream()
    .collect(Collectors.toList());  // Supplier creates new List
```

### Optional + Functional Interfaces

```java
Optional<String> value = Optional.of("Hello");

// ifPresent() : Consumer<T>
value.ifPresent(System.out::println);  // Accept and print

// map() : Function<T, R> (transforms value)
Optional<Integer> length = value.map(String::length);  // "Hello" → 5

// filter() : Predicate<T> (keeps if true)
Optional<String> nonEmpty = value.filter(s -> !s.isEmpty());

// orElseGet() : Supplier<T> (lazy default)
String result = Optional.empty()
    .orElseGet(() -> "default");  // Supplier only called if empty
```

### Comparators with Functional Interfaces

```java
class Person {
    String name;
    int age;
}

List<Person> people = Arrays.asList(
    new Person("Alice", 30),
    new Person("Bob", 25),
    new Person("Charlie", 35)
);

// Traditional: Anonymous class
Comparator<Person> byName = new Comparator<Person>() {
    @Override
    public int compare(Person a, Person b) {
        return a.name.compareTo(b.name);
    }
};

// Modern: Lambda (Comparator is functional!)
Comparator<Person> byAgeLambda = (a, b) -> Integer.compare(a.age, b.age);

// Cleanest: Method reference + chaining
people.sort(Comparator
    .comparing(Person::getName)        // Sort by name (ascending)
    .thenComparing(Person::getAge)     // Then by age (ascending)
    .reversed());                       // Reverse order

// Output: Charlie (35), Alice (30), Bob (25)
```

### Interview Answer

> "Before Java 8, passing behaviour required anonymous class boilerplate. Functional interfaces standardise the contract: one abstract method = one piece of behaviour. The key practical differences: Predicate composes with and/or/negate, Function chains with andThen/compose, Consumer chains side effects with andThen, and Supplier is about laziness — specially useful with orElseGet() because unlike orElse(), the Supplier only executes when the value is actually needed."
>
> *Likely follow-up: "What is the difference between andThen and compose in Function?"*

---

## Q6. lambda Internals — invokedynamic, Serialization, and Gotchas

### Concept
Lambdas compile differently than anonymous classes. They use **invokedynamic** bytecode instruction,
making them lightweight and serializable challenges.

### Bytecode: invokedynamic vs Anonymous Class

```java
// anonymous class
button.setOnClick(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Clicked");
    }
});
// Bytecode: Creates anonymous class file (extra .class file), slower loading

// Lambda
button.setOnClick(e -> System.out.println("Clicked"));
// Bytecode: Uses invokedynamic instruction (dynamic dispatch at runtime)
//           No extra class file created, faster loading
```

### Serialization Problem: Lambdas Can't Be Serialized Directly

```java
// ❌ BROKEN: Lambda is NOT Serializable
public class MyClass {
    public void test() throws IOException {
        Supplier<String> lambda = () -> "hello";
        
        // Serialize lambda
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(lambda);  // ✗ NotSerializableException!
        oos.close();
    }
}

// ✓ WORKS: Lambda with all captured variables must be Serializable
// This requires implementing a serializable functional interface directly
@FunctionalInterface
public interface SerializableSupplier<T> extends Supplier<T>, Serializable {
}

public class MyClass {
    public void test() throws IOException {
        SerializableSupplier<String> lambda = (SerializableSupplier<String>) () -> "hello";
        
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(lambda);  // ✓ Now works!
        oos.close();
    }
}

// ✓ BETTER: Just make your functional interface extend Serializable
public interface MyFunction<T, R> extends Function<T, R>, Serializable {
    // Lambdas assigned to this interface can be serialized
}

MyFunction<String, Integer> fn = (MyFunction<String, Integer>) s -> s.length();
// Potentially serializable (if all captured vars are Serializable)
```

### Captured Variables Must Be Serializable

```java
// ❌ BROKEN: Captured variable not Serializable
class NonSerializable {}

NonSerializable obj = new NonSerializable();
SerializableSupplier<NonSerializable> supplier = () -> obj;  // ← Captures non-serializable

try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("lambda"))) {
    oos.writeObject(supplier);  // NotSerializableException! (obj is not serializable)
}

// ✓ WORKS: Captured variable is Serializable
String str = "hello";  // String is Serializable
SerializableSupplier<String> supplier2 = () -> str;
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("lambda"))) {
    oos.writeObject(supplier2);  // ✓ Works
}
```

### Common Lambda Gotchas

```java
// Gotcha 1: this reference in lambda
class MyClass {
    int x = 10;
    
    void method() {
        Supplier<Integer> supplier = () -> x;  // ← Captures this.x (implicit this)
        // If 'this' becomes unreferenced while lambda still alive, it stays alive!
        // Memory leak risk in long-lived lambdas
    }
}

// Gotcha 2: Variable shadowing
int x = 10;
Supplier<Integer> supplier = () -> {
    int x = 20;  // Creates local x (shadows outer x)
    return x;  // Returns 20, not 10
};

// Gotcha 3: Loop variable capture (common bug)
List<Supplier<Integer>> suppliers = new ArrayList<>();
for (int i = 0; i < 3; i++) {
    suppliers.add(() -> i);  // ← Captures i, not value "i"
}

for (Supplier<Integer> s : suppliers) {
    System.out.println(s.get());  // All print 3! (i = 3 after loop)
}
// Fix: Use local final variable
for (int i = 0; i < 3; i++) {
    final int value = i;  // i. is not effectively final (reassigned each iteration)
    suppliers.add(() -> value);  // Captures value, not i
}

// Gotcha 4: Lambdas don't have their own scope for this/super
class Outer {
    int x = 10;
    
    class Inner {
        int x = 20;
        
        void method() {
            Supplier<Integer> s = () -> x;  // x refers to Inner.x (20), not Outer.x!
            System.out.println(s.get());  // 20
            
            Supplier<Integer> this_x = () -> this.x;  // Explicitly Inner.x
            System.out.println(this_x.get());  // 20
            
            Supplier<Integer> outer_x = () -> Outer.this.x;  // Explicitly Outer.x
            System.out.println(outer_x.get());  // 10
        }
    }
}
```

### Interview Answer

> "Captured variables must be effectively final — not reassigned after capture. This is because lambdas are translated to synthetic methods that receive captured variables as parameters, so reassigning them would cause confusion about which value to use. For mutable objects in scope, all mutable fields accessed must be Serializable if you ever serialize the lambda (e.g., to cache or send to another JVM)."
>
> *Likely follow-up: "Why can't a lambda access a mutable local variable?"*

---

## Q7. Serialization Concepts — Serializable, Externalizable, Custom Control

### Concept
**Serialization** converts object to byte stream for storage/network. Java provides default + custom serialization mechanisms.

### Serializable vs Externalizable

```java
// SERIALIZABLE: Simple, default mechanism
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;  // Version ID for compatibility
    
    private String name;
    private int age;
    transient private Date lastModified;  // transient field not serialized
    
    // No special methods needed; Java handles serialization automatically
}

// Serialization
Person p = new Person("John", 30);
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(p);
oos.close();

// Deserialization
ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
ObjectInputStream ois = new ObjectInputStream(bais);
Person p2 = (Person) ois.readObject();
ois.close();

// EXTERNALIZABLE: Full custom control, must implement readExternal/writeExternal
public class CustomPerson implements Externalizable {
    private String name;
    private int age;
    
    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);         // Write name with custom format
        out.writeInt(age);          // Write age
        out.writeObject(getMetadata());  // Custom metadata
    }
    
    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        name = in.readUTF();        // Read name
        age = in.readInt();         // Read age
        // Custom deserialization logic
    }
}

// Key Difference:
// Serializable: Java controls, mostly automatic (can override readObject/writeObject)
// Externalizable: You control entirely, must implement both methods

// Table:
// Feature                    | Serializable | Externalizable
// Automatic serialization    | ✓            | ✗ (must code)
// Field by field default     | ✓            | ✗ (you decide)
// Override writeObject       | ✓ (optional) | ✗ (not applicable)
// Override readObject        | ✓ (optional) | ✗ (not applicable)
// Override writeExternal     | ✗            | ✓ (required)
// Override readExternal      | ✗            | ✓ (required)
// Backward compatibility     | ✓ (serialVersionUID) | ✗
// Performance                | Slower       | Faster (custom optimization)
```

### Interview Answer

> "Serialization converts an object to bytes for storage or network transmission. The interface is a marker (Serializable), and Java's built-in serialisation handles most work. Custom serialisation (writeObject/readObject) lets you control the format — critical for security or backwards compatibility. Externalizable is for cases where you want complete control and don't want Java's default mechanism. The biggest gotcha: serialVersionUID must stay constant across versions to read old data, and readResolve() is essential for Singleton pattern during deserialisation."
>
> *Likely follow-up: "What is serialVersionUID and why must it be constant?"*

---

## Q8. Advanced Serialization Patterns — readResolve, Singleton Serialization

### Concept
**readResolve()** and **writeReplace()** allow custom object deserialization control.
Critical for Singleton patterns!

### Singleton Serialization Problem

```java
// Naive singleton (breaks with serialization!)
public class Singleton {
    public static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {}  // Private constructor, prevent instantiation
}

// Problem: Serialization creates SECOND instance!
Singleton s1 = Singleton.INSTANCE;

// Serialize
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(s1);
oos.close();

// Deserialize
ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
ObjectInputStream ois = new ObjectInputStream(bais);
Singleton s2 = (Singleton) ois.readObject();  // ← NEW INSTANCE!
ois.close();

System.out.println(s1 == s2);  // false (different objects!)
// Singleton pattern BROKEN!

// Solution: readResolve()
public class SingletonFixed implements Serializable {
    public static final SingletonFixed INSTANCE = new SingletonFixed();
    
    private SingletonFixed() {}
    
    // Called after deserialization
    private Object readResolve() throws ObjectStreamException {
        return INSTANCE;  // Always return same instance
    }
}

SingletonFixed s1 = SingletonFixed.INSTANCE;
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(s1);
oos.close();

ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
ObjectInputStream ois = new ObjectInputStream(bais);
SingletonFixed s2 = (SingletonFixed) ois.readObject();
ois.close();

System.out.println(s1 == s2);  // true (same instance!)

// Caveat: readResolve protects deserialization path only.
// Reflection can still break singleton unless constructor is guarded.
Constructor<SingletonFixed> c = SingletonFixed.class.getDeclaredConstructor();
c.setAccessible(true);
// SingletonFixed hacked = c.newInstance();  // would create another instance if constructor allows it
```

### readResolve() and writeReplace()

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String name;
    private transient String password;
    
    // Called BEFORE serialization, can replace object with proxy
    private Object writeReplace() throws ObjectStreamException {
        // Example: Replace complex object with simpler representation
        PersonProxy proxy = new PersonProxy();
        proxy.name = this.name;
        // Don't serialize password!
        return proxy;
    }
    
    // Called AFTER deserialization
    private Object readResolve() throws ObjectStreamException {
        // Can validate or reconstruct
        if (name == null || name.isEmpty()) {
            throw new RuntimeException("Invalid person: name required");
        }
        return this;
    }
}

// Proxy object (simpler, safer to serialize)
class PersonProxy implements Serializable {
    String name;
}
```

### serialVersionUID — Backward Compatibility

```java
public class VersionedClass implements Serializable {
    private static final long serialVersionUID = 1L;  // Version 1
    
    private String name;
    // v1: fields...
}

// Later, add a compatible field (UID can remain the same)
public class VersionedClass implements Serializable {
    private static final long serialVersionUID = 1L;  // Keep same UID for compatible evolution
    
    private String name;
    private int age;  // NEW field
    
    // readObject handles format differences during deserialization
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();  // Read default fields
        // age field not in old serialized version
        // Handle missing field: age = 0 (default)
    }
    
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();  // Write all fields
        // age is written for new versions
    }
}

// Only bump serialVersionUID for intentionally incompatible changes,
// so old serialized bytes are rejected with InvalidClassException.
```

### Interview Answer

> "Serialization converts an object to bytes for storage or network transmission. The interface is a marker (Serializable), and Java's built-in serialisation handles most work. Custom serialisation (writeObject/readObject) lets you control the format — critical for security or backwards compatibility. Externalizable is for cases where you want complete control and don't want Java's default mechanism. The biggest gotcha: keep serialVersionUID stable for compatible class evolution, and change it only when you intentionally break compatibility. Also, readResolve() protects singleton deserialization, but reflection needs a constructor guard."
>
> *Likely follow-up: "What is serialVersionUID and why must it be constant?"*

---

## Q9. Comparable vs Comparator — Sorting Strategies

### Concept
- **Comparable**: Object defines its natural ordering (implements Comparator-like logic)
- **Comparator**: External sorting strategy without modifying class

### Code

```java
// Comparable: Natural ordering, object defines itself
public class Person implements Comparable<Person> {
    private String name;
    private int age;
    
    @Override
    public int compareTo(Person other) {
        return this.age - other.age;  // Natural order: ascending age
    }
}

List<Person> people = Arrays.asList(
    new Person("Alice", 30),
    new Person("Bob", 25)
);

Collections.sort(people);  // Uses Person.compareTo()
// Bob (25) comes before Alice (30)

// Comparator: External sorting, no class modification needed
Comparator<Person> byName = (a, b) -> a.getName().compareTo(b.getName());
Collections.sort(people, byName);  // Alice before Bob (alphabetical)

// Can create multiple comparators for different sorts
Comparator<Person> byAge = Comparator.comparingInt(Person::getAge);
Comparator<Person> byAgeThenName = byAge.thenComparing(Person::getName);

// Comparator is functional (single abstract method: compare)
Comparator<Integer> ascending = (a, b) -> a - b;
Comparator<Integer> descending = (b, a) -> a - b;  // Reverse

List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5);
numbers.sort(ascending);    // [1, 1, 3, 4, 5]
numbers.sort(descending);   // [5, 4, 3, 1, 1]
```

### Comparison Table

| Feature | Comparable | Comparator |
|---------|-----------|-----------|
| **Type** | Interface on class | External strategy |
| **Modification** | Must modify class | No class change |
| **Multiple sorts** | Only one natural order | Unlimited |
| **Default** | Built-in assumption | Flexible choice |
| **Use** | String, Integer, Date | Custom sorting |

---

## Quick Revision Card

| Concept | Key Points |
|---------|-----------|
| **@FunctionalInterface** | ✓ Exactly 1 abstract method ✓ Can have multiple defaults/static ✗ 2+ abstract = error |
| **4 Core Interfaces** | Predicate (test), Function (apply), Consumer (accept), Supplier (get) |
| **Method References** | Static::method, instance::method, Class::instanceMethod, Class::new |
| **Effectively Final** | Lambda can capture variables only if not reassigned after capture |
| **Serialization** | Serializable (automatic), Externalizable (custom), readResolve (singleton fix) |
| **Lambda vs Anonymous** | Lambda: invokedynamic bytecode, faster; Anonymous: creates .class file |
| **Captured Variables** | Must be effectively final, all mutable objects in scope must be Serializable to serialize lambda |
| **this in Lambda** | Refers to enclosing class, not lambda itself (no lambda scope) |

---

## Real Project Usage — Topic 7 (Functional Interfaces, Lambdas & Serialization)

### Scenario 1: Method Reference Readability
> "Order processing pipeline used `.map(order -> OrderUtils.toDTO(order))` but was repetitive. Switched to method reference: `.map(OrderUtils::toDTO)`. Much clearer intent — transform order to DTO. Bonus: method references work better with JIT optimisation."

### Scenario 2: Serialization Gone Wrong
> "Caching service tried to serialize a lambda that captured a non-serializable `HttpClient`. Runtime exception: `NotSerializableException`. Lessons: (1) Capture only serializable objects. (2) Use method references (static methods) instead of lambdas for distributed caches. (3) Or mark lambda `transient` and reconstruct on deserialization."

### Scenario 3: Stream Exception Handling
> "Stream of transactions had some bad records. `.map(Parser::parse)` threw exceptions and broke the pipeline. Fixed: wrapper function that caught exceptions and returned empty Optional: `.flatMap(tx -> tryParse(tx))` where tryParse returns `Optional<Transaction>`. Bad records silently skipped, good ones processed."

### Scenario 4: Custom Predicate Chain
> "Filtering orders by location and status was done with nested if-statements. Refactored to predicate composition: `Predicate<Order> inRegion = o -> regions.contains(o.region()); Predicate<Order> isPending = o -> o.status() == PENDING; orders.stream().filter(inRegion.and(isPending)).collect(...)`. Much more testable and composable."

---

## Summary: Core Concepts Quick Reference

| Concept | Definition | Use Case |
|---------|-----------|----------|
| **@FunctionalInterface** | Interface with exactly 1 abstract method | Enables lambda assignment |
| **Predicate** | `boolean test(T t)` | Filtering (`if`, `filter`) |
| **Function** | `R apply(T t)` | Transformation (`map`) |
| **Consumer** | `void accept(T t)` | Side-effects (`forEach`) |
| **Supplier** | `T get()` | Factory/lazy eval (`orElseGet`) |
| **Method Reference** | `Class::method` syntax | Cleaner lambda, better JIT |
| **Effectively Final** | Variable not reassigned after capture | Lambda requirement |
| **Serializable Lambda** | Must capture only serializable objects | For distributed caches/RPC |

---

## Advanced Serialization — Production Patterns & Security

### Concept
Java serialization is powerful but dangerous in production. The `readObject()` method executes arbitrary bytecode — attackers can embed exploit chains (gadget chains) in serialized payloads. Additionally, serialization version mismatches break deserialization across service updates. Production systems must use `serialVersionUID` strategically, implement `readResolve()` for singletons, validate deserialized objects, and plan migration paths.

### Simple Explanation
**Serialization** = Converting an object to bytes so you can send it over network or store to disk.  
**Deserialization** = Reading those bytes back into an object.  

The danger: Deserialization calls `readObject()`, which can execute malicious code hidden in specially crafted bytes. It's like opening a ZIP file — if the ZIP unpacker has a bug, the ZIP attacker can exploit it. Production code must validate what it deserializes.

### Problem 1: Gadget Chains (Security breach via deserialization)

```java
// ❌ DANGEROUS: Blindly deserializing untrusted data
public class PaymentProcessor {
    public PaymentRequest deserializeFromNetwork(byte[] payload) {
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(payload));
        return (PaymentRequest) ois.readObject();  // VULNERABILITY!
        // If payload contains exploit chain (e.g., commons-collections library gadget),
        // readObject() executes that chain, giving attacker code execution.
    }
}

// Exploit arrives across network:
// attackers craft a byte stream that, when deserialized, triggers system command execution.
// This is how remote code execution (RCE) works in Java.
```

#### ✅ Solution 1: Whitelist Deserialization Classes

```java
public class SafePaymentProcessor {
    private static final ObjectInputFilter FILTER = ObjectInputFilter.Config.createFilter(
        "java.lang.*;java.util.*;com.payment.*;!*"  // Allow only these packages
    );

    public PaymentRequest deserializeFromNetwork(byte[] payload) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(payload));
        ois.setObjectInputFilter(FILTER);  // Enforce whitelist
        try {
            return (PaymentRequest) ois.readObject();
        } catch (InvalidClassException e) {
            throw new SecurityException("Rejected forbidden class: " + e.getMessage());
        }
    }
}

// Java 9+ ObjectInputFilter prevents deserialization of dangerous classes.
// If payload tries to deserialize a gadget chain class, it's rejected before readObject() runs.
```

### Problem 2: Version Mismatch Breaks Deserialization

```java
public class OrderEntity implements Serializable {
    private static final long serialVersionUID = 1L;  // Version 1
    private Long id;
    private String status;        // v1 fields
}

// Bytes serialized with v1 saved to cache/disk.
// Service updates:

public class OrderEntity implements Serializable {
    private static final long serialVersionUID = 1L;  // ❌ Didn't change!
    private Long id;
    private String status;
    private BigDecimal totalPrice;  // NEW field v2
}

// ❌ Deserialization breaks:
// old v1 bytes don't have totalPrice.
// InvalidClassException: local class incompatible with stream data
```

#### ✅ Solution 2: Increment serialVersionUID on Breaking Changes

```java
public class OrderEntity implements Serializable {
    private static final long serialVersionUID = 2L;  // ✅ Incremented
    private Long id;
    private String status;
    private BigDecimal totalPrice = BigDecimal.ZERO;  // Default for old deserialized objects
    
    // Custom deserialization to handle version migration
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        // Handle v1->v2 migration: set totalPrice default if null
        if (this.totalPrice == null) {
            this.totalPrice = BigDecimal.ZERO;
        }
    }
}

// Strategy:
// - Change serialVersionUID when adding/removing/renaming fields
// - Increment version = "all old serialized objects are incompatible, re-serialize"
// - Use readObject() to handle migration or set defaults
```

### Problem 3: Singleton Pattern Breaks Under Deserialization

```java
public class ConfigSingleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static ConfigSingleton instance = new ConfigSingleton();
    
    private ConfigSingleton() { }  // Private constructor
    
    public static ConfigSingleton getInstance() {
        return instance;
    }
}

// Problem:
byte[] serialized = serializeToBytes(ConfigSingleton.getInstance());
ConfigSingleton deserialized = deserializeFromBytes(serialized);

if (deserialized == ConfigSingleton.getInstance()) {
    System.out.println("Same instance");
} else {
    System.out.println("❌ Different instance! Singleton broken."); // This happens!
}

// Deserialization bypasses private constructor, creates a new object.
// Now you have 2 instances of the "singleton". Config inconsistencies follow.
```

#### ✅ Solution 3: Implement readResolve()

```java
public class ConfigSingleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static ConfigSingleton instance = new ConfigSingleton();
    private String configValue = "default";
    
    private ConfigSingleton() { }
    
    public static ConfigSingleton getInstance() {
        return instance;
    }
    
    // Called after deserialization
    private Object readResolve() throws ObjectStreamException {
        return getInstance();  // Return the singleton instance, discard deserialized copy
    }
}

// Now:
byte[] serialized = serializeToBytes(ConfigSingleton.getInstance());
ConfigSingleton deserialized = deserializeFromBytes(serialized);
assert deserialized == ConfigSingleton.getInstance();  // ✅ Same instance
```

### Problem 4: Enum Serialization Misconception (Interview Trap)

```java
public enum PaymentStatus {
    PENDING,
    APPROVED,
    REJECTED
}
```

Interview-safe clarification:
- Java enum serialization is handled specially by the JVM.
- You do not normally use `readResolve()` to make enums safe.
- Enums are already identity-safe singletons per constant.
- The practical guidance is: prefer enums for fixed-state values because serialization semantics are robust by default.

### Problem 5: Backward Compatibility After Refactoring

```java
// Old version v1
public class UserEntity implements Serializable {
    private static final long serialVersionUID = 1L;
    private String fullName;  // Single field
}

// New version v2: refactor fullName → firstName + lastName
public class UserEntity implements Serializable {
    private static final long serialVersionUID = 2L;
    private String firstName;
    private String lastName;
    
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        // Default deserialization
        ois.defaultReadObject();
        
        // Handle v1 compatibility: reconstruct from cached fullName if present
        // (This requires special handling or a migration field)
    }
}

// Better approach: use a migration pattern
public class UserEntity implements Serializable {
    private static final long serialVersionUID = 2L;
    private String firstName;
    private String lastName;
    // Keep transient field for migration data
    private transient String fullName;  // Won't serialize, but can be used in readObject
}
```

### Real Project Usage

**Scenario: Distributed cache with Serialization**
```java
public class CachedUser implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String email;
    // ... Redis serializes/deserializes this across instances
}

// Update: Add "preferences" field
public class CachedUser implements Serializable {
    private static final long serialVersionUID = 2L;  // ✅ Increment
    private Long id;
    private String email;
    private Map<String, String> preferences = new HashMap<>();  // Default for old entries
}

// Production migration:
// 1. Deploy code v2 with serialVersionUID=2
// 2. Old cached entries (serialVersionUID=1) fail deserialization
// 3. Catch InvalidClassException and re-fetch user from DB
// 4. Gradually cache expires, gets repopulated with v2 format
// 5. No downtime, graceful migration
```

**Scenario: Serialization in Message Queues (Kafka)**
```java
public class OrderEvent implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long orderId;
    private String status;
}

// Service publishes events to Kafka
// If you change OrderEvent structure, old events on queue still need to deserialize
// Solution: Increment serialVersionUID, implement readObject() to handle old format, or migrate manually
```

### Interview Answer

> "Serialization in Java is a security minefield. First, never deserialize untrusted data without validation. ObjectInputFilter in Java 9+ lets you whitelist classes — only allow your own domain classes and standard library, reject everything else. This blocks gadget chain exploits.
>
> Second, serialVersionUID is not optional — it's your versioning contract. When you add or remove fields, increment it. Old serialized objects become incompatible. In production, I implement readObject() to handle migrations: set default values for new fields, or map old field names to new ones. This makes deployments non-breaking.
>
> Third, singletons and serialization don't mix without readResolve(). Deserialization bypasses constructors and creates new instances. readResolve() intercepts deserialization and hands back the original singleton, keeping identity intact. That's critical for configs and factories.
>
> Fourth point: avoid serializing across major version boundaries. If you must, use a migration field, version it explicitly, and test thoroughly. Better yet, use JSON serialization for APIs (not Java serialization) — it's text, self-describing, and less vulnerable to gadget chains.
>
> Real example: We had a payment event queue. Service v1 published OrderEvent with a `total` field. Service v2 added a `currency` field. We incremented serialVersionUID, set a default currency in readObject(). Old events deserialized fine, service v2 just used default 'USD' for those. Smooth migration."

**Follow-up likely:** "What's the difference between serialVersionUID mismatch and a gadget chain attack?" → First is versioning (structural incompatibility). Second is security (malicious code execution during deserialization).

---

## Quick Revision Card — Serialization

| Concept | Action | Why |
|---------|--------|-----|
| **Untrusted Data** | Use ObjectInputFilter + whitelist | Prevent gadget chain RCE |
| **Adding Field** | Increment serialVersionUID, set default | Handle old deserialized objects |
| **Singleton** | Implement readResolve() | Return original instance, not deserialized copy |
| **Major Refactor** | Use migration field; test v1↔v2 | Ensure backward compatibility |
| **Enum Serialization** | Rely on built-in enum serialization | JVM preserves enum constant identity |
| **Caching Layer** | Catch InvalidClassException + refetch | Graceful migration when serialVersionUID changes |
| **Messaging (Kafka)** | Version events separately from code | Producers/consumers may be different versions |
| **Prefer JSON** | Use for APIs; Java serialization for cache | JSON is safer, human-readable, language-neutral |

---

**End of Topic 7 — Functional Interfaces, Lambdas & Serialization**

