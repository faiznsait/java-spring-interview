# PART A: Core Java — Topic 3: OOP, SOLID Principles & Design Patterns

> **Format:** Concept → Simple Explanation → How It Works → Code → Real Usage → Interview Answer

---

## Part 1: OOP FUNDAMENTALS

## Q1. Polymorphism and Dynamic Method Dispatch

### Concept
The JVM decides which method to invoke at **runtime** based on the actual object type, not the declared reference type. This is the foundation of extensible systems.

### Simple Explanation
A remote control (reference type `Device`) can point at a TV or AC (actual objects). Press "power" → TV does TV stuff, AC does AC stuff. **Same button, different behaviour** decided at runtime based on what's actually in your hand.

### How It Works

```
Compile time: Compiler checks method EXISTS on reference type
Runtime:      JVM vtable lookup — calls ACTUAL object type's version

Method Resolution:
1. Compile time: Does Animal have sound()? YES → compile OK
2. Runtime:      What is actual type? Dog → call Dog.sound()
```

### Code

```java
// Hierarchy
class Animal {
    void sound() { System.out.println("Generic sound"); }
}

class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }
}

class Cat extends Animal {
    @Override
    void sound() { System.out.println("Meow"); }
}

// Usage
Animal ref;

ref = new Dog();
ref.sound();    // Bark — decided at RUNTIME based on actual type Dog

ref = new Cat();
ref.sound();    // Meow — decided at RUNTIME based on actual type Cat

// Pattern: Flexible processing without knowing concrete types
List<Animal> animals = List.of(new Dog(), new Cat(), new Dog());
animals.forEach(Animal::sound);  // Bark, Meow, Bark
```

### What Does NOT Participate in Dynamic Dispatch

```java
// ❌ private — not inherited, not overridable
class Parent {
    private void display() { }  // Not in child vtable
}
class Child extends Parent {
    private void display() { }  // Different method, same name
}

// ❌ static — static binding (reference type decides)
class Parent {
    static void staticMethod() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void staticMethod() { System.out.println("Child"); }
}
Parent ref = new Child();
ref.staticMethod();  // Parent (static binding — decided at compile time)

// ❌ final — cannot be overridden
class Parent {
    final void finalMethod() { }
}
// Child cannot override — no vtable entry to replace
```

### Real Project Usage

```java
// Payment Processing — Core of Strategy pattern
public class PaymentProcessor {
    public PaymentResult processPayment(PaymentMethod method, BigDecimal amount) {
        return method.charge(amount);  // Which charge()? Stripe? PayPal? Decided at runtime
    }
}

// Add 10 payment types without touching PaymentProcessor
interface PaymentMethod {
    PaymentResult charge(BigDecimal amount);
}

class StripePayment implements PaymentMethod { /* ... */ }
class PayPalPayment implements PaymentMethod { /* ... */ }
class AmazonPayPayment implements PaymentMethod { /* ... */ }
```

### Interview Answer

> "Dynamic method dispatch is runtime polymorphism. The JVM decides which method to invoke based on the **actual object type** at runtime, not the **reference type** at compile time. It works via a virtual method table — each class has a vtable where overriding replaces entries. This is the foundation of the Strategy and Template Method patterns.
>
> Private, static, and final methods don't participate — they use static binding (resolved at compile time). But virtual methods are vtable lookups, making them flexible and extensible."
>
> *Likely follow-up: "What's the difference between static binding and dynamic binding?"*

---

## Q2. The Four OOP Pillars

### 1. Encapsulation — Protect and Control State

```java
// Bundle data + behaviour; expose only validated access
public class BankAccount {
    private double balance;  // Private — can't be touched directly

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new InsufficientFundsException();
        this.balance -= amount;
    }

    public double getBalance() { return balance; }
    // No setBalance() — prevents external corruption of invariant
}
```

### 2. Inheritance — Reuse and Specialise

```java
// Good use: IS-A relationship, shared behaviour
abstract class Notification {
    protected String recipient;
    protected String message;

    abstract void send();  // Each subtype implements

    void log() { 
        System.out.println("Sending to: " + recipient); 
    }
}

class EmailNotification extends Notification {
    @Override
    void send() {
        log();  // Inherited
        emailService.send(recipient, message);
    }
}
```

### 3. Polymorphism — Uniform Interface, Varied Behaviour

```java
// Runtime polymorphism via method overriding
Notification n;
n = new EmailNotification(); n.send();  // Email-specific send()
n = new SMSNotification();   n.send();  // SMS-specific send()

// Compile-time polymorphism via method overloading
class Printer {
    void print(String s) { /* */ }
    void print(int i)    { /* */ }  // Same name, different params
}
```

### 4. Abstraction — Hide Complexity Behind a Contract

```java
// Interface = pure contract (WHAT, not HOW)
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    void refund(String transactionId);
}

// Implementation hides 50+ lines of retry, idempotency, error mapping
public class StripePaymentProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // ... complexity hidden
    }
}
// Caller only knows PaymentProcessor — not Stripe internals
```

### Interview Answer
> "Encapsulation is about protection and control — bank account balance is private, operations go through validated methods. Inheritance avoids code duplication — a base `Notification` class with email, SMS, push children. Polymorphism means you can treat all notification types uniformly through the base reference — the right `send()` is called at runtime. Abstraction hides complexity behind an interface — a payment service is just `process()` and `refund()`, callers don't know if it's Stripe or PayPal underneath.
>
> In practice, I use encapsulation heavily for domain objects, I favour composition over inheritance for flexibility, and I rely on interfaces for abstraction across service layers."
>
> *Likely follow-up: "Why is composition preferred over inheritance?"*

---

## Part 2: SOLID PRINCIPLES

## Q3. SOLID: The Five Principles of Good Design

### S: Single Responsibility Principle

**One class, one reason to change**

```java
// ❌ WRONG: Mixed responsibilities
class User {
    void save() { database.insert(this); }       // Data persistence
    void sendWelcomeEmail() { email.send(...); } // Communication
    void logActivity() { logger.info(...); }     // Logging
}
// If email system changes → User class breaks; if DB changes → User breaks (too many reasons to change)

// ✅ RIGHT: Separated concerns
class User {
    private String name;
    public String getName() { return name; }
}

class UserRepository {
    void save(User user) { database.insert(user); }
}

class WelcomeEmailService {
    void sendWelcome(User user) { email.send(...); }
}

class ActivityLogger {
    void logUserCreation(User user) { logger.info(...); }
}
// Each class has ONE reason to change
```

### O: Open/Closed Principle

**Open for extension, closed for modification**

```java
// ❌ WRONG: Every new payment type requires modifying processor
class PaymentProcessor {
    PaymentResult process(String type, BigDecimal amount) {
        if (type.equals("credit_card")) { /* logic */ }
        else if (type.equals("paypal")) { /* logic */ }
        else if (type.equals("crypto")) { /* logic */ }  // MODIFIED
    }
}

// ✅ RIGHT: New types extend, don't modify
interface PaymentMethod {
    PaymentResult charge(BigDecimal amount);
}

class CreditCardPayment implements PaymentMethod { /* */ }
class PayPalPayment implements PaymentMethod { /* */ }
class CryptoPayment implements PaymentMethod { /* */ }  // NEW: no existing code touched

class PaymentProcessor {
    PaymentResult process(PaymentMethod method, BigDecimal amount) {
        return method.charge(amount);  // Works with any new type
    }
}
```

### L: Liskov Substitution Principle

**Subtype must be substitutable for supertype without breaking**

```java
// ❌ WRONG: Rectangle can't be substituted everywhere Square is
class Rectangle {
    void setWidth(double w) { this.width = w; }
    void setHeight(double h) { this.height = h; }
}

class Square extends Rectangle {
    @Override
    void setWidth(double w) { 
        this.width = w; 
        this.height = w;  // Violates contract: Square maintains w==h
    }
}

// Caller: assume all Rectangle work the same
Rectangle shape = new Square();
shape.setWidth(5);
shape.setHeight(3);
System.out.println(shape.area());  // Expected: 15, Got: 9 ❌ Contract violated

// ✅ RIGHT: Use composition
interface Shape {
    double area();
}

class Rectangle implements Shape { /* */ }
class Square implements Shape { /* */ }
// No inheritance; each shape defines its own contracts
```

### I: Interface Segregation Principle

**Many small focused interfaces, not one fat interface**

```java
// ❌ WRONG: Worker interface forces unnecessary methods
interface Worker {
    void work();       // Human workers need this
    void eat();        // Humans need this
    void recharge();   // Robots need this
}

class Human implements Worker {
    void work() { /* */ }
    void eat() { /* */ }
    void recharge() { }  // FORCED: humans don't recharge
}

class Robot implements Worker {
    void work() { /* */ }
    void eat() { }       // FORCED: robots don't eat
    void recharge() { /* */ }
}

// ✅ RIGHT: Focused interfaces
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Rechargeable {
    void recharge();
}

class Human implements Workable, Eatable { /* */ }
class Robot implements Workable, Rechargeable { /* */ }
// Each class implements only what it needs
```

### D: Dependency Inversion Principle

**Depend on abstractions, not concretions**

```java
// ❌ WRONG: Service depends on concrete UserRepository
class UserService {
    private UserRepository repo = new UserRepository();  // Tight coupling
    
    void registerUser(User user) {
        repo.save(user);
    }
}
// If UserRepository changes → UserService breaks; hard to test with mocks

// ✅ RIGHT: Service depends on abstraction
interface UserRepository {
    void save(User user);
}

class UserService {
    private UserRepository repo;  // Abstraction, not concrete
    
    public UserService(UserRepository repo) {  // Injected dependency
        this.repo = repo;
    }
    
    void registerUser(User user) {
        repo.save(user);
    }
}

// Test with mock
UserService service = new UserService(new MockUserRepository());
service.registerUser(user);  // Test runs without touching real DB
```

### Interview Answer
> "SOLID keeps code maintainable as it grows. SRP means each class has one job — my OrderService shouldn't also handle email notifications. OCP means adding features by extension not modification — we register new strategies, we don't add `if/else` branches. LSP means subtypes must be substitutable — a Square breaking a Rectangle's setWidth contract is the classic violation. ISP means don't force implementors into contracts they can't fulfil — split fat interfaces. DIP means depend on abstractions, not concretions — my Service depends on a `UserRepository` interface, not `JpaUserRepository` directly, which enables unit testing with mocks.
>
> For the calculator example: ISP says if it only needs `add()`, define an interface with just `add()`. OCP says register operations via a strategy map — so adding `multiply` later doesn't touch the Calculator class."
>
> *Likely follow-up: "Can you give an example of LSP violation?"*

---

## Part 3: CREATIONAL DESIGN PATTERNS

## Q4. Singleton Pattern

### Problem
Ensure a class has only one instance and provide global access to it.

### When to Use
- Shared resource (database connection, logger, thread pool)
- Configuration manager
- Cache manager

### Implementation (Thread-Safe)

```java
// 1. Eager initialization (simplest, but always creates instance)
public class Logger {
    public static final Logger INSTANCE = new Logger();
    
    private Logger() { }  // Prevent instantiation
}

// 2. Lazy initialization (Bill Pugh Singleton — recommended)
public class DatabaseConnection {
    private DatabaseConnection() { }
    
    static class Holder {
        static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }
    
    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE;  // Classloader handles thread-safety
    }
}

// 3. Enum (best for serialization safety)
public enum LoggerEnum {
    INSTANCE;
    
    public void log(String msg) { System.out.println(msg); }
}

// Usage
LoggerEnum.INSTANCE.log("Application started");
```

### Gotcha: Serialization Breaks Singleton

```java
// ❌ Serialization creates new instance
Logger original = Logger.INSTANCE;
byte[] serialized = serialize(original);
Logger deserialized = deserialize(serialized);
System.out.println(original == deserialized);  // false ❌

// ✅ Fix: implement readResolve()
public class Logger implements Serializable {
    private static final Logger INSTANCE = new Logger();
    
    private Logger() { }
    
    public static Logger getInstance() { return INSTANCE; }
    
    // Prevents deserialization from creating new instance
    private Object readResolve() {
        return INSTANCE;
    }
}
```

### Interview Answer
> "The Singleton pattern ensures one instance and global access. The safest way is the enum pattern — Serializable, reflection-proof, guaranteed single instance. Bill Pugh Singleton (static inner class) is nearly as safe and works pre-Java 5. Double-checked locking is tempting but fragile due to reordering — only use with `volatile`. The real issue is testability — hard-wired singletons make tests tightly coupled. Modern Spring apps rarely need it; we prefer Spring beans for global access and control."
>
> *Likely follow-up: "Can you break a Singleton with reflection?"*

---

## Q5. Factory Pattern

### Problem
Create objects without specifying their exact classes.

### Code

```java
// Simple Factory
class PaymentMethodFactory {
    static PaymentMethod create(String type) {
        return switch (type) {
            case "credit_card" -> new CreditCardPayment();
            case "paypal" -> new PayPalPayment();
            case "crypto" -> new CryptoPayment();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Abstract Factory (for families of related objects)
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

class WindowsUIFactory implements UIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

class MacUIFactory implements UIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Usage
UIFactory factory = isMac() ? new MacUIFactory() : new WindowsUIFactory();
Button btn = factory.createButton();  // Platform-specific button
```

### Interview Answer
> "Factory decouples object creation from usage. Simple Factory is a static method that returns the right type. Abstract Factory is for families of related types — when UI needs to be consistent (all Windows or all Mac widgets). Factory Method is when you have an abstract base and subclasses implement the factory method. In Java, we use `switch` expressions or dependency injection to register implementations. In production I've used factory patterns for payment gateways — caller asks for PaymentGateway, doesn't know if they get Stripe or PayPal until runtime."
>
> *Likely follow-up: "What's the difference between Factory and Builder?"*

---

## Q6. Builder Pattern

### Problem
Construct complex objects step-by-step with validation.

### Code

```java
public class UserBuilder {
    private String name;
    private String email;
    private int age;
    private List<String> roles = new ArrayList<>();
    
    public UserBuilder name(String name) {
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }
        this.name = name;
        return this;
    }
    
    public UserBuilder email(String email) {
        if (!email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        this.email = email;
        return this;
    }
    
    public UserBuilder age(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age");
        }
        this.age = age;
        return this;
    }
    
    public UserBuilder addRole(String role) {
        this.roles.add(role);
        return this;
    }
    
    public User build() {
        if (name == null) throw new IllegalStateException("Name is required");
        return new User(name, email, age, roles);
    }
}

// Fluent API
User user = new UserBuilder()
    .name("Alice")
    .email("alice@example.com")
    .age(30)
    .addRole("admin")
    .addRole("user")
    .build();
```

---

## Part 4: STRUCTURAL DESIGN PATTERNS

## Q7. Proxy Pattern

### Problem
Control access to another object without exposing it directly.

### Use Cases
- Lazy loading (load object only when needed)
- Access control (check permissions before allowing)
- Logging/monitoring
- Caching

### Code

```java
// Real object (expensive to create)
interface UserService {
    User getUser(String id);
}

class RealUserService implements UserService {
    public User getUser(String id) {
        System.out.println("Database query: SELECT * FROM users WHERE id=" + id);
        return new User(id, "Alice");
    }
}

// Proxy: lazy loading + caching
class LazyUserServiceProxy implements UserService {
    private RealUserService realService;  // Lazy
    private Map<String, User> cache = new HashMap<>();
    
    public User getUser(String id) {
        // Check cache first
        if (cache.containsKey(id)) {
            System.out.println("Cache hit for user " + id);
            return cache.get(id);
        }
        
        // Lazy initialization
        if (realService == null) {
            realService = new RealUserService();
        }
        
        User user = realService.getUser(id);
        cache.put(id, user);
        return user;
    }
}

// Usage: caller sees same interface, but lazy loading happens behind scenes
UserService service = new LazyUserServiceProxy();
service.getUser("123");  // Database query
service.getUser("123");  // Cache hit

// = Spring AOP creates proxy automatically
@Transactional  // Spring creates Proxy around this
public void transferMoney(Account from, Account to, BigDecimal amount) { }
```

### Interview Answer
> "Proxy is about controlled access to another object. Examples: lazy loading (don't create the expensive object until needed), access control (check permissions before allowing), logging, caching. Spring AOP uses a proxy to wrap `@Transactional` methods — at runtime Spring creates a proxy that commits/rolls back, so you write blocking service code but the proxy handles transactions. Proxy looks identical to the real object from outside; caller doesn't know they're using a proxy."
>
> *Likely follow-up: "How is Proxy different from Decorator?"*

---

## Q8. Decorator Pattern

### Problem
Add responsibilities to an object dynamically without altering its structure.

### Code

```java
// Component
interface Coffee {
    double cost();
    String description();
}

class SimpleCoffee implements Coffee {
    public double cost() { return 2.0; }
    public String description() { return "Simple Coffee"; }
}

// Decorators
class MilkDecorator implements Coffee {
    private Coffee coffee;
    
    public MilkDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    public double cost() { return coffee.cost() + 0.5; }
    public String description() { return coffee.description() + ", Milk"; }
}

class SugarDecorator implements Coffee {
    private Coffee coffee;
    
    public SugarDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    public double cost() { return coffee.cost() + 0.2; }
    public String description() { return coffee.description() + ", Sugar"; }
}

// Usage: stack decorators
Coffee coffee = new SimpleCoffee();  // $2.0
coffee = new MilkDecorator(coffee);  // $2.5
coffee = new SugarDecorator(coffee);  // $2.7
System.out.println(coffee.cost());  // 2.7
System.out.println(coffee.description());  // Simple Coffee, Milk, Sugar
```

### Interview Answer
> "Decorator adds responsibilities to an object dynamically without subclassing. The classic example is coffee ordering — start with SimpleCoffee, wrap it with MilkDecorator, then SugarDecorator. Each decorator implements the same interface and delegates to the wrapped object, adding its own behaviour. Real use: Spring Security filter chain (each filter is like a decorator), audit logging wrappers around service methods. Key difference from inheritance: you stack decorators at runtime for flexible composition."
>
> *Likely follow-up: "How is Decorator different from Proxy?"*

---

## Q9. Adapter Pattern

### Problem
Convert interface of one class into another clients expect.

### Code

```java
// Old interface (legacy)
interface LegacyPaymentGateway {
    void processPayment(String cardNumber, double amount);
}

// New interface (current)
interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
}

// Adapter
class PaymentGatewayAdapter implements PaymentProcessor {
    private LegacyPaymentGateway legacyGateway;
    
    public PaymentGatewayAdapter(LegacyPaymentGateway legacyGateway) {
        this.legacyGateway = legacyGateway;
    }
    
    public PaymentResult process(PaymentRequest request) {
        String cardNumber = request.getCardNumber();
        double amount = request.getAmount().doubleValue();
        
        try {
            legacyGateway.processPayment(cardNumber, amount);
            return new PaymentResult(true, "Success");
        } catch (Exception ex) {
            return new PaymentResult(false, ex.getMessage());
        }
    }
}

// Usage: new code works with new interface, adapter translates to legacy
PaymentProcessor processor = new PaymentGatewayAdapter(legacyGateway);
processor.process(request);
```

### Interview Answer
> "Adapter translates one interface into another — legacy code meets new code. Old system has `LegacyPaymentGateway.processPayment(cardNumber, amount)`, new system expects `PaymentProcessor.process(PaymentRequest)`. Write an adapter that wraps the legacy gateway and transforms calls. Real use: integrating third-party APIs with incompatible contracts, using old libraries in modern code. It's not about adding new features — it's about interface translation."
>
> *Likely follow-up: "What's the difference between Adapter and Decorator?"*

---

## Q10. Facade Pattern

### Problem
Provide unified, simplified interface to complex subsystem.

### Code

```java
// Complex subsystem: many classes
class DatabaseService { void connect() { } }
class EmailService { void sendEmail(String to, String subject) { } }
class LoggingService { void log(String msg) { } }
class AuthenticationService { void authenticate(String user, String pass) { } }

// Facade: simple interface
public class UserRegistrationFacade {
    private DatabaseService db = new DatabaseService();
    private EmailService email = new EmailService();
    private LoggingService logger = new LoggingService();
    private AuthenticationService auth = new AuthenticationService();
    
    public void registerUser(String username, String password, String emailAddr) {
        logger.log("Starting user registration");
        
        db.connect();
        db.save(new User(username, password, emailAddr));
        
        email.sendEmail(emailAddr, "Welcome!");
        
        logger.log("User registration completed");
    }
}

// Usage: caller doesn't know subsystems exist
UserRegistrationFacade facade = new UserRegistrationFacade();
facade.registerUser("alice", "password123", "alice@example.com");
// Behind scenes: DB, email, logging, all orchestrated
```

### Interview Answer
> "Facade provides a simplified, unified interface to a complex subsystem. UserRegistrationFacade hides the complexity of databases, email service, logging, authentication — one method `registerUser()` orchestrates it all. Caller never touches the subsystem classes directly. Real use: wrapping AWS SDK's verbose APIs into simple methods, orchestrating microservice calls in a backend-for-frontend service. Facade simplifies the API surface, but doesn't restrict access to subsystem classes like Adapter does."
>
> *Likely follow-up: "How do you decide between Facade and Service Layer?"*

---

## Q11. Composite Pattern

### Problem
Compose objects into tree structures to represent hierarchies.

### Code

```java
// Component
interface UIComponent {
    void render();
}

// Leaf
class Button implements UIComponent {
    private String label;
    
    public Button(String label) { this.label = label; }
    
    public void render() { System.out.println("Rendering button: " + label); }
}

// Composite
class Panel implements UIComponent {
    private List<UIComponent> components = new ArrayList<>();
    
    public void add(UIComponent component) {
        components.add(component);
    }
    
    public void render() {
        System.out.println("Rendering panel");
        for (UIComponent comp : components) {
            comp.render();  // Recursion: panel renders all children
        }
    }
}

// Usage: tree structure
Panel mainPanel = new Panel();
mainPanel.add(new Button("Save"));
mainPanel.add(new Button("Cancel"));

Panel subPanel = new Panel();
subPanel.add(new Button("Option 1"));
subPanel.add(new Button("Option 2"));

mainPanel.add(subPanel);
mainPanel.render();  // Renders entire tree recursively
```

---

## Part 5: BEHAVIORAL DESIGN PATTERNS

## Q12. Strategy Pattern

### Problem
Define family of algorithms, encapsulate each one, make them interchangeable.

### Code

```java
interface PaymentStrategy {
    PaymentResult pay(BigDecimal amount);
}

class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    public PaymentResult pay(BigDecimal amount) {
        return processViaStripe(cardNumber, amount);
    }
}

class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    public PaymentResult pay(BigDecimal amount) {
        return processViaPayPal(email, amount);
    }
}

// Context: uses strategy
class ShoppingCart {
    private PaymentStrategy strategy;
    
    public ShoppingCart(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void checkout(BigDecimal total) {
        PaymentResult result = strategy.pay(total);
        System.out.println("Payment: " + result);
    }
}

// Usage: switch strategy at runtime
ShoppingCart cart;

if (userPrefers("credit_card")) {
    cart = new ShoppingCart(new CreditCardPayment("1234-5678-9012"));
} else {
    cart = new ShoppingCart(new PayPalPayment("user@example.com"));
}

cart.checkout(new BigDecimal("99.99"));  // Same interface, different algorithm
```

### Interview Answer
> "Strategy defines a family of algorithms, encapsulates each one, makes them interchangeable. PaymentStrategy interface with CreditCardPayment, PayPalPayment, CryptoPayment implementations. ShoppingCart takes a strategy and uses it — `strategy.pay(amount)`. Which strategy is used depends on user choice at runtime. This is the core pattern you see everywhere — discount rules, sorting algorithms, search filters. It's Strategy when the algorithm varies but the flow stays the same."
>
> *Likely follow-up: "Can you give an example of Strategy without design patterns?"*

---

## Q13. Observer Pattern

### Problem
Define one-to-many dependency where when one object changes state, all dependents are notified automatically.

### Code

```java
// Subject
class Order {
    private String status;
    private List<OrderObserver> observers = new ArrayList<>();
    
    public void attach(OrderObserver observer) {
        observers.add(observer);
    }
    
    public void detach(OrderObserver observer) {
        observers.remove(observer);
    }
    
    public void setStatus(String newStatus) {
        this.status = newStatus;
        notifyObservers();
    }
    
    private void notifyObservers() {
        for (OrderObserver observer : observers) {
            observer.update(this);
        }
    }
}

// Observer interface
interface OrderObserver {
    void update(Order order);
}

// Concrete observers
class EmailNotifier implements OrderObserver {
    public void update(Order order) {
        System.out.println("Sending email: Order status is " + order.getStatus());
    }
}

class InventoryService implements OrderObserver {
    public void update(Order order) {
        System.out.println("Updating inventory for status: " + order.getStatus());
    }
}

// Usage
Order order = new Order();
order.attach(new EmailNotifier());
order.attach(new InventoryService());

order.setStatus("SHIPPED");  // Both observers notified automatically
```

### Interview Answer
> "Observer implements a publish-subscribe mechanism. Subject maintains a list of observers; when state changes, it notifies all observers. Real use: event systems (button click notifies handlers), reactive streams (Observable notifies subscribers), messaging queues. Order status changes to 'SHIPPED' — OrderObserver implementations (EmailNotifier, InventoryService) get notified and react. Key benefit: loose coupling — Subject doesn't know what observers exist, just calls `update()` on each."
>
> *Likely follow-up: "How does Observer differ from Listener?"*

---

## Q14. Template Method Pattern

### Problem
Define skeleton of algorithm, let subclasses override specific steps.

### Code

```java
// Abstract template
abstract class DataProcessor {
    
    // Template method: defines algorithm structure
    public final void process() {
        readData();
        validateData();
        transformData();
        saveData();
        notifyCompletion();
    }
    
    // Steps that vary: subclasses override
    abstract void readData();
    abstract void transformData();
    
    // Steps that are common: concrete in base class
    void validateData() {
        System.out.println("Validating...");
    }
    
    void saveData() {
        System.out.println("Saving to database...");
    }
    
    void notifyCompletion() {
        System.out.println("Process complete");
    }
}

// Concrete implementations
class CSVProcessor extends DataProcessor {
    void readData() { System.out.println("Reading CSV file"); }
    void transformData() { System.out.println("Converting CSV to objects"); }
}

class JSONProcessor extends DataProcessor {
    void readData() { System.out.println("Reading JSON file"); }
    void transformData() { System.out.println("Parsing JSON to objects"); }
}

// Usage: same algorithm flow, different implementations
DataProcessor csv = new CSVProcessor();
csv.process();  // process() calls all steps in order
```

### Interview Answer
> "Template Method defines the skeleton of an algorithm in a base class, lets subclasses override specific steps. DataProcessor.process() defines the flow — readData → validateData → transformData → saveData → notifyCompletion. CSVProcessor and JSONProcessor override only the steps that vary (readData, transformData). CSVProcessor.process() calls all steps in the same order, CSVProcessor's versions run. This prevents duplication of common steps — the template is the contract."
>
> *Likely follow-up: "How does Template Method differ from Strategy?"*

---

## Q15. State Pattern

### Problem
Allow object to alter behaviour when its state changes.

### Real-World: Order Workflow

```java
// States
interface OrderState {
    void next(Order order);
    String getStatus();
}

class PendingState implements OrderState {
    public void next(Order order) {
        order.setState(new ConfirmedState());
    }
    public String getStatus() { return "PENDING"; }
}

class ConfirmedState implements OrderState {
    public void next(Order order) {
        order.setState(new ShippedState());
    }
    public String getStatus() { return "CONFIRMED"; }
}

class ShippedState implements OrderState {
    public void next(Order order) {
        order.setState(new DeliveredState());
    }
    public String getStatus() { return "SHIPPED"; }
}

class DeliveredState implements OrderState {
    public void next(Order order) { /* Final state */ }
    public String getStatus() { return "DELIVERED"; }
}

// Context
class Order {
    private OrderState state = new PendingState();
    
    public void setState(OrderState state) { this.state = state; }
    
    public void processNext() {
        state.next(this);
    }
    
    public String getStatus() {
        return state.getStatus();
    }
}

// Usage
Order order = new Order();
System.out.println(order.getStatus());  // PENDING
order.processNext();
System.out.println(order.getStatus());  // CONFIRMED
order.processNext();
System.out.println(order.getStatus());  // SHIPPED
```

### Interview Answer
> "State allows an object to alter behaviour when its internal state changes. Order starts in PendingState, transitions to ConfirmedState, ShippedState, DeliveredState. Each state has its own implementation of `next()` — PendingState.next() sets the state to ConfirmedState. From outside, you just call `order.processNext()` without caring about states. The order's behaviour changes based on its state. Real use: workflow engines, state machines, payment processing pipelines."
>
> *Likely follow-up: "When would you use State vs a state enum?"*

---

## Q16. Chain of Responsibility Pattern

### Problem
Pass request along chain of handlers; first handler that can process it does so.

### Code

```java
// Handler
abstract class LoggingHandler {
    protected LoggingHandler next;
    
    public void setNext(LoggingHandler next) {
        this.next = next;
    }
    
    public void log(String level, String message) {
        if (canHandle(level)) {
            handle(message);
        } else if (next != null) {
            next.log(level, message);
        }
    }
    
    abstract boolean canHandle(String level);
    abstract void handle(String message);
}

// Concrete handlers
class ConsoleLogger extends LoggingHandler {
    boolean canHandle(String level) { return level.equals("INFO"); }
    void handle(String message) { System.out.println("[CONSOLE] " + message); }
}

class FileLogger extends LoggingHandler {
    boolean canHandle(String level) { return level.equals("DEBUG"); }
    void handle(String message) { System.out.println("[FILE] " + message); }
}

class EmailLogger extends LoggingHandler {
    boolean canHandle(String level) { return level.equals("ERROR"); }
    void handle(String message) { System.out.println("[EMAIL] " + message); }
}

// Setup chain
LoggingHandler console = new ConsoleLogger();
LoggingHandler file = new FileLogger();
LoggingHandler email = new EmailLogger();

console.setNext(file);
file.setNext(email);

console.log("INFO", "Application started");    // ConsoleLogger handles
console.log("DEBUG", "Debug info");             // FileLogger handles
console.log("ERROR", "Critical error");         // EmailLogger handles
```

### Interview Answer
> "Chain of Responsibility passes a request along a chain of handlers until someone handles it. Logging example: ConsoleLogger handles INFO, FileLogger handles DEBUG, EmailLogger handles ERROR. If ConsoleLogger doesn't handle it, it passes to the next handler. Real use: request pipelines, approval workflows (request goes to manager → director → CEO), exception handlers. Each handler has two choices: handle it or pass it on. Clean separation of concerns — each handler only knows its own logic."
>
> *Likely follow-up: "How is Chain of Responsibility different from Strategy?"*

---

## Part 6: ADVANCED OOP CONCEPTS

## Q17. Preventing a Class from Being Extended

### Concept
Three mechanisms to prevent subclassing, each with different semantics and use cases.

### Simple Explanation
Sometimes you want to guarantee "this class will never be subclassed" — either for security (String), performance (final method optimization), or design (sealed hierarchy). Java gives you three increasingly expressive ways.

### Code

```java
// Method 1: final keyword — simplest, most common
public final class ImmutableConfig {
    private final String apiKey;
    private final int    timeout;
    // Compiler prevents any subclass: "class X extends ImmutableConfig" → ERROR
}

// Real use: String, Integer, all wrapper types are final
// Why: Protects invariants that security or behaviour depends on
String s = "immutable";
// If String could be subclassed → someone could override hashCode() → break HashMap contracts

// Method 2: Private constructor — structural prevention
public class MathUtils {
    private MathUtils() {}  // Can't instantiate
    
    public static double square(double x) { return x * x; }
    public static double sqrt(double x)   { return Math.sqrt(x); }
    
    // No subclass possible: subclass constructor must call super()
    // But super() can't access private constructor → compile error
}

class BadSubclass extends MathUtils {
    // ERROR: implicit super() call can't access private constructor
}

// Use for: utility classes, singleton-like patterns

// Method 3: sealed classes (Java 17+) — controlled subclassing
sealed class Shape permits Circle, Rectangle, Triangle {}
final class Circle    extends Shape { /* ... */ }
final class Rectangle extends Shape { /* ... */ }
final class Triangle  extends Shape { /* ... */ }

// ONLY these three can extend Shape
class Pentagon extends Shape { }  // COMPILE ERROR: Pentagon not in permits list

// Other classes CAN extend Shape permittees
non-sealed class ExtendableCircle extends Circle { /* ... */ }
```

### Why sealed classes are powerful

```java
// Sealed enables exhaustive switch WITHOUT default clause
Shape shape = getShape();
double area = switch (shape) {
    case Circle    c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle  t -> 0.5 * t.base() * t.height();
    // No "default" needed — compiler KNOWS all cases covered
};

// Add a 4th shape LATER without updating switch => COMPILE ERROR
// Forces developer to handle new case (unlike open if/else chains)
```

### Real Project Usage

```java
// Sealed domain types — prevent malicious subclassing
sealed interface PaymentMethod permits CreditCard, PayPal, Crypto {}

record CreditCard(String cardNumber, String cvv)   implements PaymentMethod {}
record PayPal(String email)                         implements PaymentMethod {}
record Crypto(String walletAddress, String network) implements PaymentMethod {}

// Now exhaustive processing:
PaymentResult process(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc -> processStripe(cc);
        case PayPal p      -> processPayPal(p);
        case Crypto c      -> processCrypto(c);
        // No default — if a 5th type is added, this won't compile
    };
}
```

### Interview Answer

> "There are three levels of restriction. `final` class = total lockdown — no one can subclass (String, Integer). Private constructor = structural prevention — can't even instantiate, so no subclass possible (utility classes, Singlez). Java 17's `sealed` is the most expressive — you explicitly list which classes CAN subclass via `permits`, and other classes get a compile error. This is powerful because it enables exhaustive switch expressions without a default case — if someone adds a new Shape type without updating the switch, they get a compile error instead of silently hitting default.
>
> I use `final` for immutable value classes, private constructors for utility classes, and `sealed` for domain hierarchies where the set of types is fixed and known."

> *Likely follow-up: "How does sealed relate to records?"*

### Related Gotchas

```java
// ❌ Gotcha: Record is implicitly final
record Point(int x, int y) {}
class Point3D extends Point { }  // COMPILE ERROR: can't extend record

// ✅ OK: Create a new record type
record Point3D(int x, int y, int z) { }  // Different class, not a subclass

// ❌ Gotcha: sealed subclass must be final, sealed, or non-sealed
sealed class Base permits Derived {}
class Derived extends Base { }  // COMPILE ERROR: not explicitly final/sealed/non-sealed

// ✅ Fix:
sealed class Base permits Derived {}
final class Derived extends Base { }  // OK: explicitly final, no further subclassing
```

---

## Q18. Dependency Injection vs Dependency Inversion

### Concept

- **Dependency Inversion Principle (DIP)** — a **design principle**: high-level and low-level modules should both depend on abstractions, not concretions.
- **Dependency Injection (DI)** — an **implementation technique**: provide a class's dependencies from outside rather than having it create them internally.

### Simple Explanation

- **DIP** is the **rule**: "Don't hardwire dependencies together directly."
- **DI** is the **how**: "Let me hand you what you need; you don't create it yourself."

DI is often the tool used to achieve DIP, but DIP can be achieved other ways too (e.g., factories). The key insight: DI makes the dependency boundary explicit, encouraging DIP-compliant design.

### How It Works

```
WITHOUT DIP:
  OrderService
       |
       ↓ (hardwired)
   JpaOrderRepository
       |
       ↓ (hardwired)
   Database Driver

Problem: Can't swap JpaOrderRepository for InMemoryRepository in tests.
Can't switch to a different database without changing OrderService.

WITH DIP:
  OrderService
       |
       ↓ (depends on)
   OrderRepository (interface)
    /              \
   /                \
JpaOrderRepository  MockRepository

Both depend on abstraction. OrderService is immune to implementation changes.
```

### Code

```java
// ═══════════════════════════════════════════════════════════════════════════════
// WITHOUT DI or DIP (tightly coupled — BAD)
// ═══════════════════════════════════════════════════════════════════════════════

@Service
public class OrderService {
    // Creates its own dependency — hardwired to JPA
    private final OrderRepository repository = new JpaOrderRepository();
    
    public void createOrder(Order order) {
        repository.save(order);
    }
}

// Problems:
// 1. Can't swap repositories without changing OrderService
// 2. Can't test with MockOrderRepository
// 3. High coupling — OrderService knows about JPA internals

// ═══════════════════════════════════════════════════════════════════════════════
// WITH DIP (abstraction) + DI (Spring injects)  — GOOD
// ═══════════════════════════════════════════════════════════════════════════════

// Step 1: Define the abstraction (DIP)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findAll();
}

// Step 2: Implementations depend on same abstraction
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        // JPA-specific saving logic
    }
}

@Repository
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    
    @Override
    public void save(Order order) {
        store.put(order.getId(), order);
    }
}

// Step 3: High-level module depends on abstraction, receives DI (constructor)
@Service
public class OrderService {
    private final OrderRepository orderRepository;  // Dependency is INJECTED, not created
    
    // Constructor Injection — Spring provides the implementation
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}

// Usage in production: Spring injects JpaOrderRepository
// Usage in tests: Inject MockOrderRepository without Spring
@Test
void testOrderCreation() {
    OrderRepository mockRepository = new InMemoryOrderRepository();
    OrderService service = new OrderService(mockRepository);
    
    Order order = new Order("Widget", 99.99);
    service.createOrder(order);
    
    assertEquals(1, mockRepository.findAll().size());  // ✓ Passes
}
```

### Three DI Types in Spring

| Type | How | Use When | Testability |
|------|-----|----------|-------------|
| **Constructor** | Pass in `__init__` | ALWAYS — mandatory deps | ⭐⭐⭐ Best |
| **Setter** | `@Autowired` on method | Optional deps | ⭐⭐ Medium |
| **Field** | `@Autowired` on field | Never in production | ⭐ Worst |

#### 1. Constructor Injection (PREFERRED)

```java
@Service
public class OrderService {
    private final UserRepository    userRepository;
    private final InventoryService  inventoryService;
    
    // Constructor — Spring calls this when creating the bean
    public OrderService(UserRepository ur, InventoryService is) {
        this.userRepository = ur;
        this.inventoryService = is;
    }
    
    // Both fields are final — prevents accidental reassignment
    public void createOrder(Long userId, List<Item> items) {
        User user = userRepository.findById(userId).orElseThrow();
        inventoryService.reserve(items);
        // ...
    }
}
```

Why it's preferred:
- ✅ Dependencies explicit in constructor signature
- ✅ Can use `final` → prevents accidental mutation
- ✅ Circular dependencies caught at startup (BeanCurrentlyInCreationException)
- ✅ Works without Spring: `new OrderService(userRepo, inventoryService)`
- ✅ Easy to unit test without Spring context

#### 2. Setter Injection (use for optional deps)

```java
@Service
public class NotificationService {
    private EmailTemplate emailTemplate;    // Optional
    
    @Autowired(required = false)
    public void setEmailTemplate(EmailTemplate template) {
        this.emailTemplate = template;  // Only set if bean exists
    }
    
    public void notify(String message) {
        if (emailTemplate != null) {
            emailTemplate.send(message);
        }
    }
}
```

Use when: dependency is truly optional and service works without it.

#### 3. Field Injection (rarely use in production)

```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;  // Injected by Spring reflection
    
    // Problems:
    // 1. Hidden dependency — not visible in constructor
    // 2. Can't be final → no protection against reassignment
    // 3. Only works with Spring — can't instantiate in unit tests without @InjectMocks + Mockito
    // 4. Harder to debug — dependencies not explicit
}

// In unit tests — requires Mockito too
@Test
void test() {
    ProductService service = new ProductService();  // FAILS — productRepository is null!
    
    // Workaround with Mockito:
    ProductService service = Mockito.spy(new ProductService());  // Needless complexity
}

// ✅ Better: constructor injection
@Service
public class ProductService {
    private final ProductRepository productRepository;
    
    public ProductService(ProductRepository repo) {
        this.productRepository = repo;
    }
}

@Test
void test() {
    ProductService service = new ProductService(mockRepository);  // ✓ Simple
}
```

### Real Project Usage

```java
// Service layer — constructor injection enforces all deps present
@Service
@RequiredArgsConstructor  // Lombok generates constructor from final fields
public class PaymentService {
    private final PaymentGateway        gateway;
    private final NotificationService   notifier;
    private final AuditLogger           auditLogger;
    
    public PaymentResult processPayment(Payment payment) {
        PaymentResult result = gateway.charge(payment.getAmount());
        notifier.notifyUser(payment.getUserId(), result);
        auditLogger.log(payment, result);
        return result;
    }
}

// Test — mock each dependency independently
@Test
void testPaymentSuccess() {
    PaymentGateway gateway = mock(PaymentGateway.class);
    when(gateway.charge(BigDecimal.TEN)).thenReturn(new PaymentResult(true));
    
    NotificationService notifier = mock(NotificationService.class);
    AuditLogger audit = mock(AuditLogger.class);
    
    PaymentService service = new PaymentService(gateway, notifier, audit);
    PaymentResult result = service.processPayment(payment);
    
    assertTrue(result.isSuccess());
    verify(notifier).notifyUser(payment.getUserId(), result);  // Verify side effects
}
```

### Common Pitfall: Circular Dependency

```java
// ❌ OrderService needs UserService; UserService needs OrderRepository (which OrderService creates)
// → Spring fails at startup: BeanCurrentlyInCreationException

@Service
public class OrderService {
    private final UserService userService;  // Circular!
    public OrderService(UserService us) { this.userService = us; }
}

@Service
public class UserService {
    private final OrderService orderService;  // Circular!
    public UserService(OrderService os) { this.orderService = os; }
}

// ✅ Fix 1: Break the cycle by extracting shared logic
public interface OrderValidator { boolean isValid(Order order); }

@Service
public class OrderService {
    private final OrderValidator validator;  // Depends on abstraction
    public OrderService(OrderValidator v) { this.validator = v; }
}

@Service
public class UserService {
    private final OrderValidator validator;  // Same abstraction
    public UserService(OrderValidator v) { this.validator = v; }
}

// ✅ Fix 2: Use @Lazy on one side
@Service
public class OrderService {
    private final UserService userService;
    public OrderService(@Lazy UserService us) { this.userService = us; }  // Lazily initialized
}

// ✅ Fix 3: Use ObjectProvider for late binding
@Service
public class OrderService {
    private final ObjectProvider<UserService> userService;  // Resolved when .getIfAvailable() called
    public OrderService(ObjectProvider<UserService> us) { this.userService = us; }
}
```

### Interview Answer

> "DIP is a principle — depend on abstractions, not concretions. OrderService shouldn't know about JpaOrderRepository; it should depend on an OrderRepository interface. That way you can swap implementations without touching OrderService. DI is the mechanism — Spring's IoC container creates beans and injects them.
>
> I always use constructor injection: it makes dependencies explicit, supports `final` fields, enables testing without Spring, and catches circular dependencies immediately at startup. Field injection is convenient but hides dependencies, breaks unit testing, and violates the principle that dependencies should be explicit.
>
> Setter injection is for optional dependencies — service works without them. In my projects, that's rare."

> *Likely follow-up: "What is IoC (Inversion of Control) and how does Spring implement it?"*

### Related Gotchas

```java
// ❌ Gotcha 1: @Autowired on optional dependency without required=false
@Autowired
private EmailService emailService;  // Fails if bean doesn't exist

// ✅ Fix:
@Autowired(required = false)
private EmailService emailService;  // OK if bean missing

// ❌ Gotcha 2: Self-invocation bypasses @Transactional (Proxy not called)
@Service
public class OrderService {
    @Transactional
    public void createOrder(Order order) { /* ... */ }
    
    public void bulkCreate(List<Order> orders) {
        for (Order o : orders) {
            createOrder(o);  // ❌ Calls REAL method, not proxy — no transaction!
        }
    }
}

// ✅ Fix: Inject self or refactor
@Service
public class OrderService {
    private final OrderService self;  // Inject self
    
    public OrderService(ObjectProvider<OrderService> provider) {
        this.self = provider.getIfAvailable(() -> this);
    }
    
    public void bulkCreate(List<Order> orders) {
        for (Order o : orders) {
            self.createOrder(o);  // Calls proxy — transaction applied ✓
        }
    }
}

// ❌ Gotcha 3: Field injection doesn't work in unit tests
@Service
public class ProductService {
    @Autowired
    private ProductRepository repo;
}

// In unit test — repo is null!
@Test
void test() {
    ProductService service = new ProductService();  // repo = null, NPE at usage
}

// ✅ Constructor injection required for clean testing
@Service
public class ProductService {
    private final ProductRepository repo;
    public ProductService(ProductRepository r) { this.repo = r; }
}

@Test
void test() {
    ProductService service = new ProductService(mockRepo);  // ✓
}
```

---

## Part 7: KEY GOTCHAS & ANTI-PATTERNS

### ❌ Gotcha 1: Singleton Breaks During Reflection

```java
Singleton original = Singleton.getInstance();
Constructor<?> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton reflected = (Singleton) constructor.newInstance();

System.out.println(original == reflected);  // false ❌ — broken singleton
```

### ❌ Gotcha 2: Over-Engineering with Patterns

```java
// Pattern is nice, but if you only have 1 payment method, it's overkill
interface PaymentStrategy { PaymentResult pay(BigDecimal amount); }
// Unless API design demands extensibility, keep it simple
```

### ❌ Gotcha 3: Spring AOP Self-Invocation

```java
@Service
public class OrderService {
    @Transactional  // Proxy around method
    public void createOrder(Order order) { /* */ }
    
    public void processOrder(Order order) {
        createOrder(order);  // ❌ Calls REAL method, not proxy
                             // @Transactional doesn't apply
    }
}

// Fix:
public void processOrder(Order order) {
    orderService.createOrder(order);  // Inject self, call via proxy
}
```

---

## Summary: Pattern Selection Guide

| Pattern | Problem | Example | Complexity |
|---------|---------|---------|-----------|
| **Singleton** | Global unique instance | Logger, Config | Low |
| **Factory** | Object creation | PaymentMethodFactory | Low |
| **Builder** | Complex object construction | UserBuilder | Medium |
| **Proxy** | Lazy loading, access control | Spring AOP, caching | Medium |
| **Decorator** | Add responsibility dynamically | Coffee + Milk | Medium |
| **Adapter** | Interface translation | Legacy code integration | Low |
| **Facade** | Simplified API | UserRegistrationFacade | Low |
| **Composite** | Tree structures | UI components, file systems | Medium |
| **Strategy** | Interchangeable algorithms | PaymentStrategy | Medium |
| **Observer** | Notify multiple dependents | Event systems, reactive | Medium |
| **Template Method** | Algorithm skeleton | DataProcessor | Low |
| **State** | Alter behaviour by state | Order workflow | Medium |
| **Chain of Responsibility** | Pass along chain | Logging, request pipelines | Medium |

