# Topic 09: Spring Configuration, Beans, and Transactions

## Q41. `@Transactional` — How It Works Internally

### Concept
`@Transactional` automatically wraps a method in a database transaction — begin before, commit on success, rollback on exception.

### Simple Explanation
`@Transactional` is like an undo buffer. Everything you do inside the method is buffered. If the whole method succeeds — commit (permanently write). If anything throws an exception — rollback (undo everything).

### How It Works — Spring AOP Proxy
```
You call: orderService.placeOrder(request)

What actually runs:
   ┌─────────────────────────────────────────────────────┐
   │  Spring's CGLIB Proxy wrapping OrderService         │
   │                                                     │
   │  1. TransactionInterceptor.before():                │
   │     - Get connection from pool                      │
   │     - Set autoCommit = false (begin transaction)    │
   │     - Bind connection to current thread (ThreadLocal)│
   │                                                     │
   │  2. Call REAL orderService.placeOrder(request)      │
   │     - Your business logic runs                      │
   │     - All @Repository calls use SAME connection     │
   │       (from ThreadLocal binding)                    │
   │                                                     │
   │  3. TransactionInterceptor.after():                 │
   │     - No exception → connection.commit()            │
   │     - RuntimeException → connection.rollback()      │
   │     - Release connection back to pool               │
   └─────────────────────────────────────────────────────┘
```

### Code
```java
@Service
public class OrderService {

    @Transactional  // Default: propagation=REQUIRED, isolation=DEFAULT, rollbackOn=RuntimeException
    public Order placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));   // INSERT
        inventoryService.reduceStock(request.getProductId());     // UPDATE
        paymentService.charge(request.getPaymentInfo());          // INSERT
        // All three share the same DB connection/transaction
        // If paymentService.charge() throws RuntimeException → ALL three are rolled back
        return order;
    }

    // Custom configuration:
    @Transactional(
        propagation  = Propagation.REQUIRES_NEW,  // Always start a NEW transaction
        isolation    = Isolation.SERIALIZABLE,    // Highest isolation level
        timeout      = 30,                        // Rollback if takes > 30 seconds
        readOnly     = true,                      // Hint to DB: no writes → optimisation
        rollbackFor  = {Exception.class}          // Rollback on checked exceptions too!
    )
    public List<Order> getOrderHistory(Long userId) { ... }
}
```

### Propagation Levels (most important ones)
```java
// REQUIRED (default): Join existing transaction, or create new if none
// Use: most business methods

// REQUIRES_NEW: Always create new transaction; suspend any outer transaction
// Use: audit logging — must commit audit even if outer transaction rolls back

// SUPPORTS: Use existing transaction if present; otherwise run without
// Use: read-only methods that are fine with or without transaction

// NOT_SUPPORTED: Run without transaction; suspend any existing one

// NEVER: Must run without transaction; throw if one exists

// MANDATORY: Must run inside existing transaction; throw if none

// Example — REQUIRES_NEW for audit:
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(AuditEntry entry) {
    auditRepository.save(entry);  // This commits independently — survives outer rollback
}
```

### Common Pitfalls
```java
// ❌ Pitfall 1: Self-invocation — proxy is bypassed
@Service
public class OrderService {
    public void processAll(List<Order> orders) {
        for (Order o : orders) {
            placeOrder(o);  // Called on 'this', NOT on the proxy → @Transactional IGNORED!
        }
    }

    @Transactional
    public void placeOrder(Order order) { ... }
}
// Fix: inject OrderService into itself, or extract to a separate bean

// ❌ Pitfall 2: private methods — Spring proxy can't override them
@Transactional
private void doWork() { ... }  // @Transactional has no effect on private methods!

// ❌ Pitfall 3: Checked exceptions don't trigger rollback by default
@Transactional
public void process() throws IOException {
    // IOException (checked) → NO rollback by default!
}
// Fix: @Transactional(rollbackFor = Exception.class)

// ❌ Pitfall 4: @Transactional on @Repository — redundant when service already has it
// Spring Data JPA methods already have @Transactional — don't add it again unnecessarily
```

### Interview Answer
> "`@Transactional` works through Spring AOP. The container creates a proxy around your bean — when you call the annotated method, the proxy intercepts the call, fetches a connection from the pool, disables autoCommit, binds the connection to the current thread via ThreadLocal, calls your real method, then commits or rolls back. Because all your repository calls pick up the same connection from ThreadLocal, they all participate in one transaction. Default rollback is on `RuntimeException` only — checked exceptions don't trigger rollback unless you add `rollbackFor = Exception.class`. The biggest pitfall is self-invocation — calling a `@Transactional` method from another method in the same class bypasses the proxy."
>
> *Likely follow-up: "What is the difference between Propagation.REQUIRED and REQUIRES_NEW?"*

---

## Q42. Spring Bean Lifecycle — End to End

### All Phases
```
1. BeanDefinitionReader: reads @Component, @Bean, XML → BeanDefinition (blueprint)
2. BeanFactoryPostProcessor: modify BeanDefinitions before any instance is created
   e.g., PropertySourcesPlaceholderConfigurer resolves ${...} placeholders
3. Bean Instantiation: constructor called (or factory method)
4. Dependency Injection: @Autowired fields/setters populated
5. BeanNameAware.setBeanName(): inject bean name if implemented
6. BeanFactoryAware.setBeanFactory(): inject factory if implemented
7. ApplicationContextAware.setApplicationContext(): inject context if needed
8. BeanPostProcessor.postProcessBeforeInitialization(): e.g., @PostConstruct processing
9. Init callbacks run in order: @PostConstruct -> InitializingBean.afterPropertiesSet() -> custom init-method
10. BeanPostProcessor.postProcessAfterInitialization(): AOP proxies created HERE
11. Bean is READY → stored in singleton cache
— Application runs —
12. Container shutdown triggered (JVM hook or context.close())
13. @PreDestroy method (or DisposableBean.destroy())
14. Bean destroyed
```

```java
@Component
@Slf4j
public class DatabaseConnectionPool implements InitializingBean, DisposableBean {

    @Value("${db.pool.size:10}")
    private int poolSize;

    // Phase 3: Constructor
    public DatabaseConnectionPool() {
        log.info("1. Constructor called — fields NOT yet injected, poolSize=0 here");
    }

    // Phase 4: @Autowired injection happens here (automatically)

    // Phase 9: init method — properties ARE injected, safe to use them
    @PostConstruct
    public void init() {
        log.info("2. @PostConstruct — poolSize={}, initialising {} connections", poolSize, poolSize);
        // ✓ Safe to access @Value fields here
        initializePool(poolSize);
    }

    // Alternative to @PostConstruct (from InitializingBean interface):
    @Override
    public void afterPropertiesSet() {
        log.info("2b. afterPropertiesSet called (runs after @PostConstruct)");
    }

    // Phase 13: cleanup before bean is destroyed
    @PreDestroy
    public void cleanup() {
        log.info("3. @PreDestroy — closing all connections");
        closeAllConnections();
    }

    // Alternative to @PreDestroy (from DisposableBean interface):
    @Override
    public void destroy() {
        log.info("3b. DisposableBean.destroy()");
    }
}
```

### BeanPostProcessor — Where AOP Proxies Are Created
```java
// Spring's internal AutoProxyCreator (a BeanPostProcessor) runs in phase 10.
// It checks if any @Aspect advice applies to the bean.
// If yes → wraps the real bean in a CGLIB proxy before putting it in the context.
// This is why @Transactional, @Cacheable, @Async all work — they're proxy-based.

// Custom BeanPostProcessor (rare — for framework-level work):
@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(Audited.class)) {
            return Proxy.newProxyInstance(/* wrap in audit proxy */);
        }
        return bean;  // Return unchanged for non-audited beans
    }
}
```

---

## Q43. DI and IoC — All Injection Types

### Concept
**IoC (Inversion of Control):** Instead of your code creating its dependencies, the framework creates and provides them. Control is inverted — the container is in charge.
**DI (Dependency Injection):** The mechanism used to achieve IoC — dependencies are injected from outside.

### Simple Explanation
Old way: "I need a car, I'll build one myself." IoC/DI: "I need a car — here's the spec, let the factory build it and hand it to me."

### Three Injection Types

```java
// ─── 1. CONSTRUCTOR INJECTION (Preferred) ────────────────────────────────────
@Service
public class OrderService {
    private final UserRepository    userRepository;
    private final InventoryService  inventoryService;

    // @Autowired optional if only one constructor (Spring 4.3+)
    public OrderService(UserRepository userRepository, InventoryService inventoryService) {
        this.userRepository    = userRepository;
        this.inventoryService  = inventoryService;
    }
}
// ✓ Dependencies are final → immutable, no null risk after construction
// ✓ Explicit contract — all deps declared upfront
// ✓ Easy unit testing: new OrderService(mockRepo, mockInventory)
// ✓ Detects circular deps at startup (not at first use)

// ─── 2. SETTER INJECTION ──────────────────────────────────────────────────────
@Service
public class NotificationService {
    private EmailService emailService;  // Optional dependency

    @Autowired(required = false)        // Won't fail if EmailService bean is absent
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
// ✓ Good for OPTIONAL dependencies
// ✓ Allows reconfiguration after construction (rarely needed)
// ✗ Bean can exist in partially-initialised state (null field)
// ✗ Not immutable

// ─── 3. FIELD INJECTION ───────────────────────────────────────────────────────
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
}
// ✓ Concise — no boilerplate
// ✗ Can't use final → mutable
// ✗ Hard to unit test — must use Spring test context or reflection to inject mock
// ✗ Hides dependencies — no explicit constructor contract
// ✗ Spring best-practice guides recommend AGAINST it
```

### `@Autowired` Resolution — How Spring Picks the Right Bean
```java
// Step 1: Match by TYPE
@Autowired
private PaymentGateway gateway;  // Finds all beans implementing PaymentGateway

// Step 2: If multiple matches → match by NAME (bean name = class name camelCase)
// Step 3: If still ambiguous → throw NoUniqueBeanDefinitionException

// Resolve ambiguity:
@Primary  // Preferred bean when multiple candidates
@Component
public class StripeGateway implements PaymentGateway { ... }

@Qualifier("paypalGateway")  // Explicit name
@Autowired
private PaymentGateway gateway;

// Or inject ALL implementations:
@Autowired
private List<PaymentGateway> allGateways;  // Spring injects [StripeGateway, PayPalGateway]
```

---

## Q44. Constructor vs Setter Injection — When to Use Setter

### Constructor Injection — Mandatory Dependencies
```java
// All required deps declared at construction time → object always valid
@Service
public class InvoiceService {
    private final OrderRepository    orderRepo;
    private final TaxCalculator      taxCalc;
    private final InvoicePdfRenderer renderer;

    public InvoiceService(OrderRepository orderRepo,
                          TaxCalculator taxCalc,
                          InvoicePdfRenderer renderer) {
        this.orderRepo = Objects.requireNonNull(orderRepo);
        this.taxCalc   = Objects.requireNonNull(taxCalc);
        this.renderer  = Objects.requireNonNull(renderer);
    }
}
```

### When You MUST Use Setter Injection

**Scenario 1: Optional plugin-style dependency**
```java
@Service
public class ReportService {
    private SlackNotifier slackNotifier;  // Optional — not all deployments have Slack

    @Autowired(required = false)
    public void setSlackNotifier(SlackNotifier slackNotifier) {
        this.slackNotifier = slackNotifier;
    }

    public void generateReport() {
        Report report = buildReport();
        if (slackNotifier != null) {
            slackNotifier.notify("Report ready: " + report.getId());
        }
    }
}
```

**Scenario 2: Circular dependency (last resort)**
```java
// If A needs B and B needs A — constructor injection → StackOverflowError at startup
// Setter injection breaks the cycle: A is created first (half-wired), then B is created
// and wired into A, then A's setter is called to wire B
@Service
public class ServiceA {
    private ServiceB serviceB;

    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
// Better fix: refactor to remove the circular dep entirely
```

---

## Q45. Circular Dependency in Spring — How to Resolve

### The Problem
```java
// Circular: A→B→A
@Service public class ServiceA {
    public ServiceA(ServiceB b) { ... }  // A needs B
}
@Service public class ServiceB {
    public ServiceB(ServiceA a) { ... }  // B needs A
}
// Spring: creating A requires B, creating B requires A → UnsatisfiedDependencyException
```

### Resolutions

```java
// ── Resolution 1: @Lazy — break the cycle with lazy proxy ──────────────────
@Service public class ServiceA {
    public ServiceA(@Lazy ServiceB b) { ... }  // Spring injects a lazy proxy for B
    // B won't actually be created until it's first USED
}

// ── Resolution 2: Setter injection for one side ─────────────────────────────
@Service public class ServiceA {
    private ServiceB serviceB;
    @Autowired public void setServiceB(ServiceB b) { this.serviceB = b; }
}

// ── Resolution 3: ApplicationContext lookup (not recommended) ───────────────
@Service public class ServiceA implements ApplicationContextAware {
    private ApplicationContext ctx;
    @Override public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
    public void doSomething() {
        ServiceB b = ctx.getBean(ServiceB.class);  // Lazy lookup
    }
}

// ── Best Resolution: Refactor to remove the cycle ───────────────────────────
// Extract the shared logic into a third service C that both A and B use
// A → C ← B  (no cycle)
@Service public class SharedService { ... }
@Service public class ServiceA {
    public ServiceA(SharedService shared) { ... }
}
@Service public class ServiceB {
    public ServiceB(SharedService shared) { ... }
}
```

---

## Q46. Singleton vs Prototype Scope

### Concept
- **Singleton (default):** One instance per Spring ApplicationContext, shared by all.
- **Prototype:** New instance created every time the bean is requested from the container.

```java
// Singleton (default) — one shared instance
@Component  // or @Scope("singleton")
public class UserService {
    // All code that injects UserService gets the SAME object
}

// Prototype — new instance per request
@Component
@Scope("prototype")  // or @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ReportBuilder {
    private List<String> lines = new ArrayList<>();  // Safe: each caller gets its own copy

    public void addLine(String line) { lines.add(line); }
    public String build() { return String.join("\n", lines); }
}
// If ReportBuilder were Singleton, concurrent callers would share 'lines' → data corruption!

// Web scopes (web applications only):
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class RequestContext { ... }  // New instance per HTTP request

@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class UserCart { ... }  // New instance per HTTP session
```

---

## Q47. Getting a New Instance Every Time — Prototype in Singleton

### The Problem
```java
// Singleton bean holds a reference to a prototype bean
@Component
public class OrderProcessor {
    @Autowired
    private ReportBuilder reportBuilder;  // Injected ONCE at construction time!
    // Even though ReportBuilder is @Scope("prototype"),
    // Spring injects ONE instance and it's reused forever → broken!
}
```

### Solutions

```java
// ── Solution 1: ApplicationContext.getBean() — pull on demand ───────────────
@Component
public class OrderProcessor {
    @Autowired
    private ApplicationContext ctx;

    public String process(Order order) {
        ReportBuilder rb = ctx.getBean(ReportBuilder.class);  // New instance each call
        rb.addLine("Processing: " + order.getId());
        return rb.build();
    }
}

// ── Solution 2: ObjectProvider (Spring 4.3+) — cleaner, testable ────────────
@Component
public class OrderProcessor {
    @Autowired
    private ObjectProvider<ReportBuilder> reportBuilderProvider;
    // IMPORTANT: ReportBuilder must be @Scope("prototype").
    // ObjectProvider defers lookup; it does not change bean scope by itself.

    public String process(Order order) {
        ReportBuilder rb = reportBuilderProvider.getObject();  // New prototype instance
        rb.addLine("Processing: " + order.getId());
        return rb.build();
    }
}

// ── Solution 3: @Lookup (method injection) — Spring overrides the method ────
@Component
public abstract class OrderProcessor {
    public String process(Order order) {
        ReportBuilder rb = createReportBuilder();  // Spring-overridden method
        rb.addLine("Processing: " + order.getId());
        return rb.build();
    }

    @Lookup  // Spring generates a CGLIB subclass that overrides this method
    protected abstract ReportBuilder createReportBuilder();
    // Each call returns a new prototype instance
}
```

---

## Q48. Multiple Databases in Spring Boot

```java
// application.yml
datasource:
  primary:
    url: jdbc:postgresql://localhost:5432/orders_db
    username: orders_user
    password: secret
  secondary:
    url: jdbc:mysql://localhost:3306/users_db
    username: users_user
    password: secret

// ─── Primary DataSource Config ───────────────────────────────────────────────
@Configuration
@EnableJpaRepositories(
    basePackages = "com.app.orders.repository",   // Repositories using primary DB
    entityManagerFactoryRef = "primaryEntityManager",
    transactionManagerRef   = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig {

    @Primary
    @Bean @ConfigurationProperties("datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean primaryEntityManager(
            EntityManagerFactoryBuilder builder) {
        return builder
            .dataSource(primaryDataSource())
            .packages("com.app.orders.entity")   // Entities for this DB
            .persistenceUnit("primary")
            .build();
    }

    @Primary
    @Bean
    public PlatformTransactionManager primaryTransactionManager() {
        return new JpaTransactionManager(primaryEntityManager(...).getObject());
    }
}

// ─── Secondary DataSource Config (same pattern, no @Primary) ────────────────
@Configuration
@EnableJpaRepositories(
    basePackages = "com.app.users.repository",
    entityManagerFactoryRef = "secondaryEntityManager",
    transactionManagerRef   = "secondaryTransactionManager"
)
public class SecondaryDataSourceConfig {
    @Bean @ConfigurationProperties("datasource.secondary")
    public DataSource secondaryDataSource() { ... }
    // ... same pattern without @Primary
}
```

---

## Q49. Proxy Design Pattern in Spring

### Where Spring Uses Proxies

| Feature | Proxy Type | What it wraps |
|---|---|---|
| `@Transactional` | CGLIB | Service methods |
| `@Cacheable` | CGLIB | Service methods |
| `@Async` | CGLIB | Async methods |
| `@Secured` / `@PreAuthorize` | CGLIB | Security-checked methods |
| Spring AOP `@Aspect` | CGLIB or JDK Dynamic | Any bean |
| `@FeignClient` | JDK Dynamic Proxy | Interface methods → HTTP calls |

```java
// Two proxy types:
// 1. JDK Dynamic Proxy — only for interfaces
//    Class implements interface → proxy wraps the interface
// 2. CGLIB Proxy — for classes (default in Spring Boot)
//    Creates a subclass at runtime → overrides methods to add interception

// Example: @Transactional creates a CGLIB proxy
@Service
public class OrderService {          // Real class
    @Transactional
    public void placeOrder() { ... }
}

// What Spring actually puts in the context (conceptually):
class OrderService$$SpringCGLIB extends OrderService {
    @Override
    public void placeOrder() {
        txManager.begin();       // Added by proxy
        try {
            super.placeOrder();  // Your real code
            txManager.commit();
        } catch (RuntimeException e) {
            txManager.rollback();
            throw e;
        }
    }
}

// Why self-invocation breaks @Transactional:
orderService.placeOrder();  // Goes through CGLIB proxy ✓ — transaction starts
this.placeOrder();          // Calls the REAL object directly, bypasses proxy ✗
```

---

## Q50. Spring Core Container — BeanFactory vs ApplicationContext

### Concept
`BeanFactory` is the basic IoC container. `ApplicationContext` extends it with enterprise features.

| Feature | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Lazy loading | ✅ Beans loaded on first request | ❌ Eager: all singletons loaded at startup |
| Event publishing | ❌ | ✅ `ApplicationEventPublisher` |
| AOP integration | Manual | ✅ Auto-wires `BeanPostProcessor` |
| i18n / MessageSource | ❌ | ✅ |
| Environment/Profiles | ❌ | ✅ |
| `@PostConstruct`, lifecycle | ❌ | ✅ |
| Use case | Embedded/resource-constrained | All Spring Boot applications |

```java
// BeanFactory (low-level — almost never used directly):
BeanFactory factory = new XmlBeanFactory(new FileSystemResource("beans.xml"));
MyBean bean = (MyBean) factory.getBean("myBean");  // Loaded lazily on first getBean()

// ApplicationContext (use this always):
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// All singleton beans are instantiated HERE (at context creation)
MyBean bean = ctx.getBean(MyBean.class);

// In Spring Boot — ApplicationContext is created automatically:
ConfigurableApplicationContext ctx = SpringApplication.run(MyApp.class, args);

// Event publishing through ApplicationContext:
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public Order placeOrder(OrderRequest req) {
        Order order = processOrder(req);
        eventPublisher.publishEvent(new OrderPlacedEvent(order));  // Decoupled notification
        return order;
    }
}

@EventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    emailService.sendConfirmation(event.getOrder());  // Runs in same thread by default
}
```

### Interview Answer (Q49–Q50 combined)
> "The Spring container has two tiers: `BeanFactory` (basic IoC, lazy) and `ApplicationContext` (adds AOP, events, i18n, environment, eager loading). In Spring Boot you always work with `ApplicationContext` — it's what `SpringApplication.run()` creates. The proxy pattern is the mechanism behind Spring's declarative features: `@Transactional`, `@Cacheable`, `@Async` all work by generating a CGLIB subclass at startup that intercepts method calls. The proxy is what's registered in the context — only self-invocation bypasses it. `@FeignClient` uses JDK dynamic proxies because it wraps an interface into HTTP calls."

---

## Quick Revision Card — Section 5

| Q | Core Point |
|---|---|
| **Q38** | Starter Parent = version management + plugin config. Starters = curated transitive deps. Boot = embedded server + auto-config |
| **Q39** | Boot startup: scan → BeanDefinitions → BeanFactoryPostProcessors → instantiate → @PostConstruct → Tomcat. Request: Filter → DispatcherServlet → Handler → Controller → MessageConverter |
| **Q40** | `@Configuration` = manual beans. `@EnableAutoConfiguration` = classpath-driven auto beans gated by `@ConditionalOn*`. Your beans take priority (auto-config backs off) |
| **Q41** | AOP proxy intercepts call → begin tx → run method → commit/rollback. Default: rollback RuntimeException only. Pitfalls: self-invocation, private methods, checked exceptions |
| **Q42** | Constructor → @Autowired → Aware → postProcessBefore → @PostConstruct → postProcessAfter → READY → @PreDestroy |
| **Q43** | Constructor (preferred, final, testable) → Setter (optional deps) → Field (avoid). Use `@Primary` or `@Qualifier` for multiple impls |
| **Q44** | Constructor = mandatory deps. Setter = optional deps OR circular dependency last resort. Best fix for circular = refactor to 3rd class |
| **Q45** | `@Lazy` on injection point, setter injection, or `ObjectProvider`. Best: refactor to eliminate the cycle |
| **Q46** | Singleton = one shared instance. Prototype = new instance per request. Request/Session scopes for web |
| **Q47** | Prototype in singleton: `ObjectProvider<T>.getObject()` or `ctx.getBean()` or `@Lookup` abstract method |
| **Q48** | Two `@Configuration` classes, separate package scans, `@Primary` on primary config, `@EnableJpaRepositories` pointing each to its EntityManager |
| **Q49** | CGLIB proxy for classes, JDK proxy for interfaces. `@Transactional`/`@Cacheable`/`@Async` all work via CGLIB. Self-call bypasses proxy |
| **Q50** | `BeanFactory` = lazy, basic. `ApplicationContext` = eager singletons + AOP + events + profiles. Always use `ApplicationContext` in Boot |

---

**End of Section 5**

---

## Additional Deep-Dive (Q41-Q50)

### Production Troubleshooting Playbook

1. `@Transactional` not working:
- Check method visibility (`public` required for proxy-based interception).
- Check invocation path (self-invocation bypasses proxy).
- Confirm proxy creation (`spring.aop.proxy-target-class=true` when needed).

2. Circular dependency on startup:
- Verify if both beans truly need each other at construction time.
- Extract shared orchestration into a third service.
- Use `@Lazy` only as temporary mitigation, not final design.

3. Multiple bean candidates error:
- Prefer explicit `@Qualifier` over hidden `@Primary` ambiguity in large teams.
- Keep interfaces in domain modules and implementations in infrastructure modules to reduce accidental component scan collisions.

### Senior Interview Edge Points

- Why constructor injection is preferred:
It enforces immutability, prevents partially initialized beans, and makes tests trivial with plain constructors.

- Why field injection is discouraged:
It hides dependencies, blocks immutability (`final`), and encourages framework-coupled tests.

- Why proxy mechanics matter:
Understanding proxy boundaries is the difference between "annotation user" and "framework engineer" in senior interviews.

### Real Project Patterns

- Use `ApplicationEventPublisher` for intra-service decoupling (order placed -> email, audit, cache warmup).
- Keep event listeners idempotent to avoid duplicate side effects when retrying failed flows.
- Split transaction boundaries carefully: main business transaction in one service, audit/logging with `REQUIRES_NEW` when failure isolation is required.

---

## Bean Scopes and Configuration Binding Deep-Dive

### Bean Scopes You Must Know for Interviews

| Scope | Lifecycle | Typical Use |
|---|---|---|
| `singleton` (default) | One bean per Spring container | Services, repositories, stateless components |
| `prototype` | New bean each request to container | Stateful helper objects |
| `request` | One bean per HTTP request | Request context, per-request metadata |
| `session` | One bean per HTTP session | Session user preferences |
| `application` | One bean per ServletContext | Shared web-app state |

```java
@Component
@Scope("prototype")
public class ReportBuilder {
    private final List<String> lines = new ArrayList<>();
    public void addLine(String line) { lines.add(line); }
}

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestAuditContext {
    private String correlationId;
    public String getCorrelationId() { return correlationId; }
    public void setCorrelationId(String correlationId) { this.correlationId = correlationId; }
}
```

### Prototype in Singleton — Common Trap

```java
@Service
public class InvoiceService {
    private final ObjectProvider<ReportBuilder> reportBuilderProvider;

    public InvoiceService(ObjectProvider<ReportBuilder> reportBuilderProvider) {
        this.reportBuilderProvider = reportBuilderProvider;
    }

    public void generateInvoice() {
        ReportBuilder builder = reportBuilderProvider.getObject();
        builder.addLine("invoice-start");
    }
}
```

### `@Value` vs `@ConfigurationProperties`

- Use `@Value` for single keys.
- Use `@ConfigurationProperties` for grouped, validated config.

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {
    @NotBlank
    private String provider;

    @Min(1000)
    private int timeoutMs;

    public String getProvider() { return provider; }
    public void setProvider(String provider) { this.provider = provider; }
    public int getTimeoutMs() { return timeoutMs; }
    public void setTimeoutMs(int timeoutMs) { this.timeoutMs = timeoutMs; }
}
```

### Interview Drill

Question: When do you use `prototype` in real projects?

Strong answer:
- Rarely for business services.
- Mostly for stateful builders/processors that should not share mutable state.
- Access through `ObjectProvider` when injected into singleton beans.

---

## Spring Bean Thread Safety — Shared State in Singletons

### Concept
Spring's default bean scope is **singleton** — one instance per container, shared across all threads. This is efficient for stateless services but dangerous if the bean holds mutable shared state. Multiple requests can invoke the same bean method concurrently, racing to read/write shared fields. Production systems must distinguish between thread-safe singletons (stateless) and thread-unsafe singletons (sharing mutable state), and use synchronization or thread-local storage where needed.

### Simple Explanation
Imagine a bank teller window:
- **Stateless teller** (thread-safe logic): Each customer transaction is independent. Multiple customers can use teller window simultaneously without interfering. Window is reusable.
- **Teller with shared notepad** (thread-unsafe mutable state): Teller jots notes on a shared pad. Two customers' transactions get mixed. Notes overwrite mid-transaction.

Spring singletons are like the shared notepad: reusable but dangerous if state is shared.

### How It Works — Concurrent Access to Singleton Bean

```
REQUEST 1 (Thread-1)          REQUEST 2 (Thread-2)
─────────────────────         ──────────────────────
GET /user/100                 GET /user/200
    ↓                             ↓
Spring injects @Service       Spring injects same @Service instance
UserService bean              UserService bean
    ↓                             ↓
┌─────────────────────────────────────────┐
│ @Service                                │
│ class UserService {                     │
│   private List<User> buffer = [];       │ ← Shared mutable state!
│                                         │
│   public User getUser(Long id) {        │
│     buffer.add(new User(id));  ← TH1   │     buffer.add(new User(id)); ← TH2
│     Thread.sleep(1000);        ← TH1   │     Thread.sleep(1000);  ← TH2
│     User result = buffer.get(0); ← TH1 │     User result = buffer.get(0); ← TH2
│     return result;             ← TH1   │     return result; ← TH2
│   }                                     │
│ }                                       │
└─────────────────────────────────────────┘

RISK: TH1 adds User(100), TH2 adds User(200), both see buffer with 2 users,
both might return User(200)! Data corruption, wrong user returned to customer.
```

### Problem 1: Shared Mutable State in Singleton

```java
@Service
public class OrderService {
    // ❌ WRONG: Shared mutable state
    private List<OrderItem> currentOrder = new ArrayList<>();
    private BigDecimal totalPrice = BigDecimal.ZERO;
    
    public void addItem(OrderItem item) {
        currentOrder.add(item);  // Not thread-safe!
        totalPrice = totalPrice.add(item.getPrice());
    }
    
    public OrderDto submitOrder() {
        OrderDto dto = new OrderDto(currentOrder, totalPrice);
        currentOrder.clear();  // ❌ One thread clears while another adds
        totalPrice = BigDecimal.ZERO;
        return dto;
    }
}

// Scenario:
// Thread 1: addItem(Item-A) → currentOrder = [A]
// Thread 2: addItem(Item-B) → currentOrder = [A, B]       ← Both threads see both items!
// Thread 1: submitOrder() → clears currentOrder → []
// Thread 2: addItem(Item-C) → currentOrder = [C]           ← Lost Item-A, Item-B!
// Thread 2: submitOrder() → returns [C] instead of [A, B, C]

// Result: Orders scrambled, items lost between requests.
```

#### ✅ Solution 1: Keep Singletons Stateless

```java
@Service
public class OrderService {
    private final OrderRepository repository;
    private final PriceCalculator calculator;  // Stateless dependency
    
    public OrderService(OrderRepository repository, PriceCalculator calculator) {
        this.repository = repository;
        this.calculator = calculator;
    }
    
    // ✅ Thread-safe: No shared mutable state
    public OrderDto submitOrder(List<OrderItem> items) {  // State passed as parameter
        BigDecimal total = calculator.calculateTotal(items);  // Stateless computation
        OrderEntity entity = new OrderEntity();
        entity.setItems(items);
        entity.setTotal(total);
        repository.save(entity);
        return new OrderDto(entity);
    }
}

// Each request independent. No shared buffer. Safe for concurrent access.
```

### Problem 2: ThreadLocal Leaks in Singleton Beans

```java
@Service
public class RequestContextService {
    // ❌ DANGEROUS: ThreadLocal in singleton
    private static final ThreadLocal<User> currentUser = ThreadLocal.withInitial(() -> null);
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();
    
    public void setCurrentUser(User user) {
        currentUser.set(user);
    }
    
    public User getCurrentUser() {
        return currentUser.get();  // Correct thread gets correct user
    }
}

// Scenario: Thread pool (e.g., Tomcat with 10 threads)
// Request 1 runs on Thread-1: setCurrentUser(Alice) → ThreadLocal[Thread-1] = Alice
// Request 1 completes, Thread-1 returns to pool
// Request 2 runs on Thread-1: getCurrentUser() → ThreadLocal[Thread-1] still has Alice!
// ❌ Request 2 sees Alice (from previous request). Security breach!

// ThreadLocal was not cleared. Thread reused. Value leaks to next request.
```

#### ✅ Solution 2: Always Clean ThreadLocal

```java
@Service
public class RequestContextService {
    private static final ThreadLocal<User> currentUser = ThreadLocal.withInitial(() -> null);
    
    public void setCurrentUser(User user) {
        currentUser.set(user);
    }
    
    public User getCurrentUser() {
        return currentUser.get();
    }
    
    public void clear() {
        currentUser.remove();  // ✅ CRITICAL: Clean up
    }
}

// In servlet filter or Spring interceptor:
@Component
public class CleanupInterceptor implements HandlerInterceptor {
    @Autowired
    private RequestContextService contextService;
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
            Object handler, Exception ex) throws Exception {
        contextService.clear();  // ✅ Always called, even if exception
    }
}
```

#### ⚠️ Even Better: Use Spring's RequestScope

```java
// Spring provides RequestScope automatically
@Component
@Scope("request")  // New instance PER REQUEST (not per thread)
public class UserContext {
    private User currentUser;
    
    public void setCurrentUser(User user) {
        this.currentUser = user;  // Only visible to this request
    }
    
    public User getCurrentUser() {
        return this.currentUser;
    }
    // No ThreadLocal, Spring manages lifecycle
}

// Use in singleton service:
@Service
public class OrderService {
    private final UserContext userContext;  // Injected request-scoped bean
    
    public OrderService(UserContext userContext) {
        this.userContext = userContext;  // ObjectFactory wraps it
    }
    
    public void placeOrder(OrderDto dto) {
        User user = userContext.getCurrentUser();  // Safe: request-scoped
        // User is never leaked to another request
    }
}
```

### Problem 3: Race Conditions in Lazy Initialization

```java
@Service
public class CacheService {
    private Map<String, String> cache;  // Lazy init
    
    public String get(String key) {
        if (cache == null) {  // ❌ Check
            cache = new HashMap<>();  // ❌ Initialize (slow, happens multiple times)
        }
        return cache.get(key);
    }
}

// Scenario:
// Thread 1: cache == null? YES → start initializing
// Thread 2: cache == null? YES → start initializing (same time!)
// Both threads create separate HashMap instances
// Thread 1 writes to its map, Thread 2 writes to its map
// ❌ They're writing to different maps! Inconsistent state.
```

#### ✅ Solution 3: Synchronize or Use Constructor Injection

```java
// Option A: Synchronize lazy init
@Service
public class CacheService {
    private volatile Map<String, String> cache;  // volatile ensures visibility
    
    public String get(String key) {
        if (cache == null) {
            synchronized (this) {
                if (cache == null) {  // Double-check locking
                    cache = new HashMap<>();
                }
            }
        }
        return cache.get(key);
    }
}

// Option B: Constructor injection (preferred, avoids lazy init race)
@Service
public class CacheService {
    private final Map<String, String> cache;  // Final, initialized in constructor
    
    public CacheService(@Qualifier("cache-bean") Map<String, String> cache) {
        this.cache = cache;  // Spring injects, no lazy init race
    }
    
    public String get(String key) {
        return cache.get(key);
    }
}
@Configuration
class CacheConfig {
    @Bean("cache-bean")
    public Map<String, String> cache() {
        return new HashMap<>();
    }
}

// Option C: Synchronized method (simple but may be slower)
@Service
public class CacheService {
    private Map<String, String> cache = Collections.synchronizedMap(new HashMap<>());
    
    public synchronized String get(String key) {
        return cache.get(key);
    }
}
```

### Real Project Usage

**Scenario: Session tracking in a request handler**
```java
@Service
public class UserService {
    // Injecting request-scoped bean into singleton service
    private final ObjectProvider<RequestContext> requestContextProvider;
    
    public UserService(ObjectProvider<RequestContext> requestContextProvider) {
        this.requestContextProvider = requestContextProvider;
    }
    
    public UserDto getCurrentUserProfile() {
        RequestContext context = requestContextProvider.getObject();  // Current request only
        Long userId = context.getUserId();
        return getUserById(userId);
    }
}

@Component
@Scope("request")
public class RequestContext {
    private Long userId;
    private String ipAddress;
    // ... request-specific data, never leaked
}
```

**Scenario: Health check service with monitoring state**
```java
@Service
public class HealthCheckService {
    private final AtomicInteger requestCount = new AtomicInteger(0);  // Thread-safe counter
    private final ConcurrentHashMap<String, Long> lastCheckTime = new ConcurrentHashMap<>();  // Thread-safe map
    
    public void recordCheck(String service) {
        requestCount.incrementAndGet();  // Atomic, no race
        lastCheckTime.put(service, System.currentTimeMillis());  // Concurrent-safe
    }
    
    public int getTotalChecks() {
        return requestCount.get();
    }
}

// Use thread-safe data structures for shared state in singletons.
```

### Interview Answer

> "Spring singletons are shared across all requests, so they're dangerous if they hold mutable state. The best practice is to keep singletons stateless — all computation is on method parameters or return values, no instance fields.
>
> If you need state, two options: First, use request-scoped or session-scoped beans. Spring creates a new instance per request/session, so no sharing. You inject them into your singleton service using ObjectProvider wrapper. Second, if you must have shared state, use thread-safe data structures like ConcurrentHashMap or AtomicInteger.
>
> ThreadLocal is risky. Thread pools reuse threads. If you don't call remove(), the value leaks to the next request on that thread — security breach. If you use ThreadLocal, always wrap cleanup in a filter or interceptor.
>
> Real example: We had a RequestContextService that stored current user in a static ThreadLocal. A thread from the pool ran request 1, set currentUser=Alice, then returned to pool. Request 2 on the same thread got Alice from ThreadLocal — wrong user. We switched to @Scope(\"request\") and injected RequestContext as ObjectProvider wrapper. Singletons accessed it through ObjectProvider.getObject(), which returns the request-scoped instance. No more leaks.
>
> Bottom line: Singleton = stateless or thread-safe data structures. Request/session-scoped = per-request state. ThreadLocal = only if you manage cleanup."

**Follow-up likely:** "How do you inject a request-scoped bean into a singleton?" → ObjectProvider wrapper delays the lookup until request context is available.

---

## Quick Revision Card — Spring Bean Thread Safety

| Scenario | Risk | Solution |
|----------|------|----------|
| **Mutable state in singleton** | Concurrent modification, data loss | Keep singleton stateless; pass state as parameter |
| **ThreadLocal without cleanup** | Value leaks to next request on same thread | Always call remove() in finally/filter |
| **Lazy init race** | Multiple threads initialize; inconsistent state | Use constructor injection or double-check locking |
| **Request-scoped data** | Shared across threads accidentally | Inject as ObjectProvider<RequestContext> |
| **Shared mutable field** | Read-write race, lost updates | Use thread-safe collections (AtomicInteger, ConcurrentHashMap) |
| **Singleton accessing request context** | Can't access per-request data | Use @Scope(\"request\") + ObjectProvider |
| **Spring session beans** | Serialize/deserialize across instances | Implement Serializable, handle transient fields |

---
