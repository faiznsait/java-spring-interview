# Topic 11: Spring Runtime, Observability, and Testing
    ↓
┌───────────────────────────────────────────────────────────┐
│  1. Servlet Container (Tomcat/Jetty/Undertow)            │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  2. Filter Chain (@WebFilter or FilterRegistrationBean)  │
│     - Logging Filter                                      │
│     - Security Filter (Spring Security)                   │
│     - CORS Filter                                         │
│     - Custom Filters                                      │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  3. DispatcherServlet (Front Controller)                 │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  4. HandlerMapping                                        │
│     Maps request URL → Controller method                  │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  5. Interceptor Chain (preHandle)                        │
│     - Authentication Interceptor                          │
│     - Logging Interceptor                                 │
│     - Rate Limit Interceptor                              │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  6. HandlerAdapter                                        │
│     Invokes @Controller method                            │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  7. Controller (@GetMapping / @PostMapping)              │
│     Business logic / Service layer calls                  │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  8. Service Layer (@Service)                             │
│     Business logic / Repository calls                     │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  9. Repository Layer (@Repository / JPA)                 │
│     Database interaction                                  │
└───────────────────────────────────────────────────────────┘
    ↓ (Response flows back)
┌───────────────────────────────────────────────────────────┐
│  10. @ControllerAdvice (if exception thrown)             │
│      Global exception handler                             │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  11. Interceptor Chain (postHandle, afterCompletion)     │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│  12. View Resolver / Message Converter                   │
│      JSON serialization (@RestController)                 │
└───────────────────────────────────────────────────────────┘
    ↓
Client receives Response
```

### Code Example — Complete Flow

#### 1. Filter
```java
@Component
@Order(1)  // Executed first
public class RequestLoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        log.info("Filter: Request {} {}", req.getMethod(), req.getRequestURI());
        
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);  // Continue to next filter/servlet
        
        long duration = System.currentTimeMillis() - start;
        log.info("Filter: Request completed in {}ms", duration);
    }
}
```

#### 2. Interceptor
```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        log.info("Interceptor: preHandle");
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;  // Stop here — don't proceed to controller
        }
        
        // Validate JWT, store user in context
        return true;  // Proceed to controller
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                          Object handler, ModelAndView modelAndView) {
        log.info("Interceptor: postHandle (after controller, before view)");
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                               Object handler, Exception ex) {
        log.info("Interceptor: afterCompletion (after everything, even exceptions)");
        // Cleanup: close resources, log metrics
    }
}

// Register interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private AuthenticationInterceptor authInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")       // Apply to these paths
                .excludePathPatterns("/api/auth/login");  // Skip for login
    }
}
```

#### 3. Controller
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable Long id) {
        log.info("Controller: getOrder called for id={}", id);
        OrderDto order = orderService.getOrder(id);
        return ResponseEntity.ok(order);
    }
    
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        log.info("Controller: createOrder called");
        OrderDto order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
}
```

#### 4. Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        log.error("Entity not found: {}", ex.getMessage());
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_ERROR", "An error occurred"));
    }
}
```

### Execution Order Summary
```
1. Filter.doFilter() — BEFORE
2. Interceptor.preHandle()
3. Controller method
4. Interceptor.postHandle()
5. View resolution / JSON serialization
6. Interceptor.afterCompletion()
7. Filter.doFilter() — AFTER (continuation)
```

### Interview Answer
> "Request enters through the servlet container, goes through Filter chain (logging, security), reaches DispatcherServlet. HandlerMapping finds the matching controller method. Interceptor preHandle runs (authentication check). Controller method executes, calls service layer. Response flows back through interceptor postHandle and afterCompletion, then serialized to JSON by message converters. If an exception occurs anywhere, @ControllerAdvice catches it globally. Filters run outside Spring MVC, interceptors inside — that's why filters see raw request/response, interceptors see Spring types."

---

## Q98. Filter vs Interceptor

### Side-by-Side Comparison

| | Filter | Interceptor |
|---|---|---|
| **Part of** | Java EE (Servlet spec) | Spring MVC |
| **When** | Before DispatcherServlet | After DispatcherServlet, before Controller |
| **Access to** | Raw `ServletRequest`/`ServletResponse` | Spring types: `HttpServletRequest`, handler method |
| **Can access** | Request/Response only | Request, Response, Controller method, ModelAndView |
| **URL patterns** | Regex / URL patterns | Ant-style paths with exclude support |
| **Skip controller?** | No (can't see Spring context) | Yes (return `false` from `preHandle`) |
| **Typical use** | Encoding, CORS, compression, security | Auth, logging, rate limiting, transaction management |
| **Order** | `@Order` or `FilterRegistrationBean` | `InterceptorRegistry` order |

### Filter — Best For
```java
// 1. Request/Response modification at HTTP level
@Component
public class CompressionFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // Wrap response to enable GZIP compression
        response = new GZIPResponseWrapper((HttpServletResponse) response);
        chain.doFilter(request, response);
    }
}

// 2. CORS headers (before Spring Security)
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorsFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        chain.doFilter(req, res);
    }
}

// 3. Character encoding
@Component
public class EncodingFilter extends CharacterEncodingFilter {
    public EncodingFilter() {
        setEncoding("UTF-8");
        setForceEncoding(true);
    }
}
```

### Interceptor — Best For
```java
// 1. Authentication/Authorization (Spring-aware)
@Component
public class RoleInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // handler = actual controller method (Spring knows it)
        if (handler instanceof HandlerMethod) {
            HandlerMethod method = (HandlerMethod) handler;
            RequiresRole annotation = method.getMethodAnnotation(RequiresRole.class);
            
            if (annotation != null && !hasRole(annotation.value())) {
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                return false;  // Stop execution
            }
        }
        return true;
    }
}

// 2. Performance logging per controller
@Component
public class PerformanceInterceptor implements HandlerInterceptor {
    private static final ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        startTime.set(System.currentTimeMillis());
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                               Object handler, Exception ex) {
        long duration = System.currentTimeMillis() - startTime.get();
        if (handler instanceof HandlerMethod) {
            HandlerMethod method = (HandlerMethod) handler;
            log.info("Controller: {}.{} took {}ms",
                method.getBeanType().getSimpleName(),
                method.getMethod().getName(),
                duration);
        }
        startTime.remove();
    }
}

// 3. Inject data into model before controller
@Component
public class UserContextInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String userId = extractUserFromToken(request.getHeader("Authorization"));
        request.setAttribute("userId", userId);  // Available in controller
        return true;
    }
}
```

### Why Two Mechanisms Exist
```
Filter = servlet-level, generic HTTP processing
  → Works even if you don't use Spring MVC
  → Security frameworks (Spring Security) use Filters

Interceptor = Spring MVC-specific, controller-aware
  → Has access to handler method details
  → Can manipulate ModelAndView
  → Better for business-level concerns
```

### Interview Answer
> "Filters are servlet-level — run before DispatcherServlet, work with raw ServletRequest/Response. Use for HTTP-level concerns: encoding, CORS, compression. Spring Security uses filters because it needs to work outside Spring MVC.
>
> Interceptors are Spring MVC-level — run after DispatcherServlet, before controller. They have access to the handler method and ModelAndView. Use for business concerns: authentication (checking roles on specific methods), performance logging, adding user context. Interceptors can skip controller execution by returning false from preHandle.
>
> In my project, I use a filter for request/response logging with correlation IDs (applies to all servlets). I use interceptors for role-based access control (Spring-aware, can inspect @RequiresRole annotations on controller methods)."

---

## Q99. When `@Component` Becomes a Bean

### Component Scanning Process

```
1. SpringApplication.run() starts
2. @SpringBootApplication has @ComponentScan
3. Component Scanner scans base package + sub-packages
4. Finds classes with stereotype annotations:
   @Component, @Service, @Repository, @Controller
5. Creates BeanDefinition for each
6. BeanFactory instantiates beans
7. Dependency injection (@Autowired)
8. BeanPostProcessor hooks (e.g., @PostConstruct)
9. Bean ready for use
```

### Code Example
```java
// Default: scans package of @SpringBootApplication and sub-packages
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}

// Custom scan paths
@SpringBootApplication
@ComponentScan(basePackages = {"com.app.service", "com.app.repository"})
public class MyApp { }
```

```java
package com.app.service;

@Component  // ← Detected during component scan
public class EmailService {
    
    @PostConstruct  // ← Called after bean creation
    public void init() {
        log.info("EmailService bean initialized");
    }
    
    public void sendEmail(String to, String message) {
        // business logic
    }
}
```

### Bean Lifecycle Timeline
```
Application Start
    ↓
Component Scan (finds @Component classes)
    ↓
BeanDefinition Registration
    ↓
Bean Instantiation (constructor called)
    ↓
Dependency Injection (@Autowired fields/setters populated)
    ↓
Aware Interfaces (BeanNameAware, ApplicationContextAware)
    ↓
@PostConstruct / InitializingBean.afterPropertiesSet()
    ↓
Bean Ready (available in ApplicationContext)
    ↓
Application Running...
    ↓
Application Shutdown
    ↓
@PreDestroy / DisposableBean.destroy()
    ↓
Bean Destroyed
```

### When Bean Creation Happens

**Singleton (default):**
```java
@Component  // Singleton by default
public class MyService {
    public MyService() {
        log.info("MyService constructor called");  // ← Called ONCE at startup
    }
}

// Bean created at application startup (eager initialization)
// Same instance reused for all requests
```

**Prototype:**
```java
@Component
@Scope("prototype")
public class TaskProcessor {
    public TaskProcessor() {
        log.info("TaskProcessor constructor called");  // ← Called EVERY time
    }
}

// New instance created EVERY TIME you inject or getBean()
```

**Lazy Initialization:**
```java
@Component
@Lazy
public class HeavyService {
    public HeavyService() {
        log.info("HeavyService constructor");  // ← Called only when FIRST USED
    }
}

// Bean NOT created at startup
// Created when first injected or accessed
```

### Custom Condition
```java
@Component
@ConditionalOnProperty(name = "feature.email.enabled", havingValue = "true")
public class EmailService {
    // Only becomes a bean if application.properties has:
    // feature.email.enabled=true
}

@Component
@Profile("prod")
public class ProductionDataSource {
    // Only becomes a bean if active profile is "prod"
    // java -jar app.jar --spring.profiles.active=prod
}
```

### Interview Answer
> "When SpringApplication.run() executes, component scanning kicks in — scans the package of @SpringBootApplication and sub-packages for classes with @Component, @Service, @Repository, @Controller. For each found class, Spring creates a BeanDefinition, instantiates it (calls constructor), performs dependency injection (@Autowired), calls @PostConstruct, and registers it in ApplicationContext.
>
> Singleton beans (default) are created eagerly at startup. Prototype beans are created on-demand every time you request them. @Lazy beans are created only when first accessed. @Conditional annotations let you conditionally create beans based on profiles, properties, or custom conditions.
>
> **Follow-up:** *What if two beans of the same type exist?*
> — Use @Primary on one bean to mark it as default, or use @Qualifier at injection point to specify which one. Without either, you get NoUniqueBeanDefinitionException."

---

## Q100. RestTemplate vs WebClient

### Blocking vs Non-Blocking

| | RestTemplate | WebClient |
|---|---|---|
| **Introduced** | Spring 3.0 | Spring 5.0 (WebFlux) |
| **Blocking?** | ✅ Yes — thread waits for response | ❌ No — async/reactive |
| **Thread model** | One thread per request | Event loop — few threads handle many requests |
| **Status** | Maintenance mode (not recommended) | Recommended for all new projects |
| **Use for** | Legacy projects only | All HTTP clients |
| **Reactive support** | ❌ No | ✅ Yes (returns Mono/Flux) |
| **Can use in non-reactive app?** | ✅ Yes | ✅ Yes (with `.block()`) |

### RestTemplate (Blocking)
```java
@Service
public class UserService {
    private final RestTemplate restTemplate = new RestTemplate();
    
    public User getUser(Long id) {
        String url = "https://api.example.com/users/" + id;
        
        // Thread BLOCKS here until response received
        User user = restTemplate.getForObject(url, User.class);
        
        return user;  // continues after response arrives
    }
    
    // Parallel calls — manual thread management
    public Dashboard getDashboard(Long userId) {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        Future<User> userF = executor.submit(() -> restTemplate.getForObject("/users/" + userId, User.class));
        Future<List<Order>> ordersF = executor.submit(() -> restTemplate.exchange("/orders?userId=" + userId, HttpMethod.GET, null, new ParameterizedTypeReference<List<Order>>() {}).getBody());
        Future<PaymentSummary> paymentF = executor.submit(() -> restTemplate.getForObject("/payments/summary/" + userId, PaymentSummary.class));
        
        try {
            return new Dashboard(userF.get(), ordersF.get(), paymentF.get());
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            executor.shutdown();
        }
    }
}
```

### WebClient (Non-Blocking)
```java
@Service
public class UserService {
    private final WebClient webClient = WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    
    // Reactive style — returns Mono
    public Mono<User> getUser(Long id) {
        return webClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .bodyToMono(User.class);
        // No blocking — subscribes asynchronously
    }
    
    // Block if you need synchronous behavior (in non-reactive app)
    public User getUserSync(Long id) {
        return webClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .bodyToMono(User.class)
                .block();  // ← Blocks current thread until response
    }
    
    // Parallel calls — declaratively
    public Mono<Dashboard> getDashboard(Long userId) {
        Mono<User> userMono = webClient.get().uri("/users/{id}", userId)
                .retrieve().bodyToMono(User.class);
        
        Mono<List<Order>> ordersMono = webClient.get().uri("/orders?userId={id}", userId)
                .retrieve().bodyToFlux(Order.class).collectList();
        
        Mono<PaymentSummary> paymentMono = webClient.get().uri("/payments/summary/{id}", userId)
                .retrieve().bodyToMono(PaymentSummary.class);
        
        // Zip = execute all in parallel, combine results
        return Mono.zip(userMono, ordersMono, paymentMono)
                .map(tuple -> new Dashboard(tuple.getT1(), tuple.getT2(), tuple.getT3()));
    }
    
    // Error handling
    public Mono<User> getUserWithFallback(Long id) {
        return webClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .onStatus(HttpStatus::is4xxClientError, response -> 
                    Mono.error(new UserNotFoundException("User not found")))
                .onStatus(HttpStatus::is5xxServerError, response -> 
                    Mono.error(new ServiceUnavailableException("Service down")))
                .bodyToMono(User.class)
                .doOnError(ex -> log.error("Error fetching user", ex))
                .onErrorReturn(new User(id, "Unknown", "unknown@example.com"));  // fallback
    }
}
```

### Performance Comparison
```
Scenario: 1000 concurrent requests, each waits 1s for response

RestTemplate (Blocking):
  - 1000 threads needed (one per request)
  - High memory usage (each thread = ~1MB stack)
  - Context switching overhead
  - Total: ~1000MB memory

WebClient (Non-Blocking):
  - ~10 threads (event loop)
  - Threads not blocked — handle multiple requests each
  - Total: ~10MB memory
  - Result: 100x better resource utilization
```

### When to Block in WebClient (Acceptable Use)
```java
// Scenario: Scheduled job (not serving user requests)
@Scheduled(cron = "0 0 * * * *")
public void hourlySyncJob() {
    List<User> users = webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(User.class)
            .collectList()
            .block(Duration.ofSeconds(30));  // ✅ OK — not in request thread
    
    users.forEach(this::processUser);
}
```

### Configuration Best Practices
```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
                .baseUrl("https://api.example.com")
                .defaultHeader(HttpHeaders.USER_AGENT, "MyApp/1.0")
                .clientConnector(new ReactorClientHttpConnector(
                    HttpClient.create()
                        .responseTimeout(Duration.ofSeconds(5))   // timeout per request
                        .connectionProvider(ConnectionProvider.builder("custom")
                            .maxConnections(100)                   // connection pool size
                            .pendingAcquireTimeout(Duration.ofSeconds(30))
                            .build())
                ))
                .filter(logRequest())   // Log all requests
                .filter(addAuthHeader())  // Add auth header
                .build();
    }
    
    private ExchangeFilterFunction logRequest() {
        return (request, next) -> {
            log.info("Request: {} {}", request.method(), request.url());
            return next.exchange(request);
        };
    }
    
    private ExchangeFilterFunction addAuthHeader() {
        return (request, next) -> {
            ClientRequest filtered = ClientRequest.from(request)
                    .header("Authorization", "Bearer " + getToken())
                    .build();
            return next.exchange(filtered);
        };
    }
}
```

### Interview Answer
> "RestTemplate is blocking — thread waits for response, one thread per request. It's in maintenance mode since Spring 5. WebClient is non-blocking/reactive — few threads handle many concurrent requests via event loop, 100x better resource utilization.
>
> For new projects, always use WebClient. In non-reactive apps, you can still use it and call `.block()` if you need synchronous behavior. For parallel calls, WebClient's `Mono.zip()` is declarative and efficient — RestTemplate requires manual ExecutorService management.
>
> I configure WebClient with connection pooling (100 connections), timeouts (5s per request), and filters for logging and auth headers. In production, non-blocking WebClient lets us handle 10,000 concurrent requests with 10 threads instead of 10,000 threads.
>
> **Follow-up:** *When would you still use RestTemplate?*
> — Only in legacy projects already using it. Migration cost may not be worth it if the app is simple and traffic is low. All new code should use WebClient."

---

## Q101. Exception Handling Best Practices

### Global Exception Handler (`@RestControllerAdvice`)

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // 1. Domain-specific exceptions
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex, WebRequest request) {
        log.error("Entity not found: {}", ex.getMessage());
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.NOT_FOUND.value())
                .error("Not Found")
                .message(ex.getMessage())
                .path(((ServletWebRequest) request).getRequest().getRequestURI())
                .build();
    }
    
    // 2. Validation errors (@Valid failed)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex, WebRequest request) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Validation Failed")
                .message("Invalid request payload")
                .fieldErrors(fieldErrors)
                .path(((ServletWebRequest) request).getRequest().getRequestURI())
                .build();
    }
    
    // 3. Business logic exceptions
    @ExceptionHandler(InsufficientFundsException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ErrorResponse handleInsufficientFunds(InsufficientFundsException ex, WebRequest request) {
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.UNPROCESSABLE_ENTITY.value())
                .error("Business Rule Violation")
                .message(ex.getMessage())
                .path(((ServletWebRequest) request).getRequest().getRequestURI())
                .build();
    }
    
    // 4. Security/Auth exceptions
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex, WebRequest request) {
        log.warn("Access denied: {}", ex.getMessage());
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.FORBIDDEN.value())
                .error("Forbidden")
                .message("You do not have permission to access this resource")
                .path(((ServletWebRequest) request).getRequest().getRequestURI())
                .build();
    }
    
    // 5. Generic fallback (catch-all)
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex, WebRequest request) {
        log.error("Unexpected error", ex);  // Full stack trace in logs
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .error("Internal Server Error")
                .message("An unexpected error occurred")  // Don't expose details to client
                .path(((ServletWebRequest) request).getRequest().getRequestURI())
                .build();
    }
}
```

### Error Response Model (RFC 7807 Problem Details)
```java
@Data
@Builder
public class ErrorResponse {
    private Instant timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private Map<String, String> fieldErrors;  // For validation errors
    
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String traceId;  // For distributed tracing
}

// Example JSON response:
/*
{
  "timestamp": "2026-03-13T10:30:00Z",
  "status": 400,
  "error": "Validation Failed",
  "message": "Invalid request payload",
  "path": "/api/orders",
  "fieldErrors": {
    "email": "must be a valid email",
    "quantity": "must be greater than 0"
  },
  "traceId": "abc-123-def-456"
}
*/
```

### HTTP Status Code Guide

| Status | Use Case | Example |
|---|---|---|
| **200 OK** | Successful GET/PUT | User fetched, profile updated |
| **201 Created** | Resource created | Order created, user registered |
| **204 No Content** | Successful DELETE | Order cancelled, user deleted |
| **400 Bad Request** | Validation failure, malformed JSON | Email format invalid |
| **401 Unauthorized** | Missing/invalid auth | No JWT token, expired token |
| **403 Forbidden** | Auth valid but no permission | Regular user accessing admin endpoint |
| **404 Not Found** | Resource doesn't exist | Order ID 999 not found |
| **409 Conflict** | Business rule conflict | Email already registered |
| **422 Unprocessable Entity** | Business logic failure | Insufficient funds, stock unavailable |
| **429 Too Many Requests** | Rate limit exceeded | User made 1000 requests in 1 minute |
| **500 Internal Server Error** | Unexpected error, DB down | Null pointer, DB connection failed |
| **503 Service Unavailable** | Service temporarily down | Maintenance mode, circuit breaker open |

### Validation with `@Valid`
```java
@PostMapping
public ResponseEntity<OrderDto> createOrder(@Valid @RequestBody CreateOrderRequest request) {
    // If validation fails, MethodArgumentNotValidException thrown automatically
    // Caught by @RestControllerAdvice
    OrderDto order = orderService.createOrder(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(order);
}

@Data
public class CreateOrderRequest {
    @NotNull(message = "Product ID is required")
    private Long productId;
    
    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity;
    
    @Email(message = "Email must be valid")
    @NotBlank(message = "Email is required")
    private String email;
    
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    private BigDecimal amount;
}
```

### Custom Exception Hierarchy
```java
// Base exception
public abstract class BusinessException extends RuntimeException {
    protected BusinessException(String message) {
        super(message);
    }
}

// Domain exceptions
public class EntityNotFoundException extends BusinessException {
    public EntityNotFoundException(String entity, Long id) {
        super(String.format("%s with id %d not found", entity, id));
    }
}

public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(BigDecimal available, BigDecimal required) {
        super(String.format("Insufficient funds: available=%s, required=%s", available, required));
    }
}

public class DuplicateEmailException extends BusinessException {
    public DuplicateEmailException(String email) {
        super(String.format("Email %s is already registered", email));
    }
}
```

### Interview Answer
> "I use @RestControllerAdvice for global exception handling — centralizes all error responses in one place. Each exception type gets its own handler with appropriate HTTP status: 404 for EntityNotFoundException, 400 for validation errors, 422 for business logic failures like InsufficientFunds.
>
> Error response model follows RFC 7807 Problem Details — includes timestamp, status, error type, message, path, and traceId for distributed tracing. For validation errors, I include a fieldErrors map showing which fields failed and why.
>
> Never expose internal details (stack traces, SQL errors) in production responses — log them server-side with full context, return generic message to client. Always use appropriate status codes — 401 for auth failure, 403 for permission denial, 422 for business rule violations."

---


## Q103. Logging and Monitoring Best Practices

### Structured Logging with Logback/SLF4J

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- Console for local dev -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [traceId=%X{traceId}] - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- JSON for production (ELK-friendly) -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.json</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.json.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="JSON" />
    </root>
    
    <!-- Custom levels -->
    <logger name="com.app" level="DEBUG" />
    <logger name="org.springframework.web" level="WARN" />
    <logger name="org.hibernate.SQL" level="DEBUG" />
</configuration>
```

### Application Logging Patterns
```java
@RestController
@Slf4j
public class OrderController {
    
    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        // 1. Log entry with context
        log.info("Fetching order: id={}", id);
        
        try {
            OrderDto order = orderService.getOrder(id);
            
            // 2. Log success with key details
            log.info("Order fetched successfully: id={}, status={}, total={}",
                    order.getId(), order.getStatus(), order.getTotal());
            
            return order;
            
        } catch (EntityNotFoundException ex) {
            // 3. Log business exceptions at WARN (expected)
            log.warn("Order not found: id={}", id);
            throw ex;
            
        } catch (Exception ex) {
            // 4. Log unexpected exceptions at ERROR with full context
            log.error("Failed to fetch order: id={}", id, ex);
            throw ex;
        }
    }
}

@Service
@Slf4j
public class PaymentService {
    
    public Payment processPayment(PaymentRequest req) {
        // Structured logging with MDC (Mapped Diagnostic Context)
        try (MDC.MDCCloseable userId = MDC.putCloseable("userId", req.getUserId().toString());
             MDC.MDCCloseable orderId = MDC.putCloseable("orderId", req.getOrderId().toString())) {
            
            log.info("Processing payment: amount={}, method={}",
                    req.getAmount(), req.getMethod());
            
            Payment payment = stripeClient.charge(req);
            
            log.info("Payment successful: paymentId={}, transactionId={}",
                    payment.getId(), payment.getTransactionId());
            
            return payment;
        }
        // MDC automatically cleared when try block exits
    }
}
```

### What to Log (and What Not to)

```java
✅ LOG:
- Request entry/exit with key params
- Business events (order placed, payment processed)
- External API calls (start, success, failure, duration)
- Exception context (not just stack trace)
- Performance metrics (slow queries, long operations)
- Security events (login attempts, access denied)

❌ DON'T LOG:
- Passwords, credit cards, PII (unless encrypted/masked)
- Full request/response bodies in production (too verbose, may contain secrets)
- Inside tight loops (log spam)
- At DEBUG level in production for high-traffic paths

// Masking sensitive data
log.info("User logged in: email={}, password=****", email);
log.info("Payment processed: cardNumber=****{}", last4Digits);
```

### Monitoring with Spring Boot Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized  # Hide details from public
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

```java
// Custom health indicator
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("validConnection", true)
                    .build();
        } catch (SQLException ex) {
            return Health.down()
                    .withDetail("error", ex.getMessage())
                    .build();
        }
    }
}

// Custom metrics
@Service
public class OrderService {
    private final MeterRegistry meterRegistry;
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCreatedCounter = Counter.builder("orders.created")
                .description("Total orders created")
                .tag("service", "order-service")
                .register(meterRegistry);
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
                .description("Order processing duration")
                .register(meterRegistry);
    }
    
    public Order createOrder(CreateOrderRequest req) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(req);
            orderCreatedCounter.increment();
            return order;
        });
    }
}
```

### Prometheus + Grafana Dashboards

**Key Metrics to Monitor:**
```
JVM:
  - jvm.memory.used / jvm.memory.max
  - jvm.gc.pause (GC pause duration)
  - jvm.threads.live

HTTP:
  - http.server.requests (rate, latency, errors)
  - Filter by: uri, status, method

Database:
  - hikaricp.connections.active
  - hikaricp.connections.pending

Custom:
  - orders.created.total
  - payments.failed.total
  - circuit.breaker.state (OPEN/CLOSED)
```

**Sample Grafana Dashboard Panels:**
- Request rate (requests/sec)
- p50, p95, p99 latency
- Error rate (%)
- Active threads
- Heap usage (%)
- DB connection pool usage

### Distributed Tracing Integration
```java
// Micrometer Tracing auto-injects traceId + spanId into logs
// No code changes needed — just dependency

// Manual span for critical operations
@Autowired Tracer tracer;

public void processOrder(Order order) {
    Span span = tracer.nextSpan().name("process-order").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        span.tag("order.id", order.getId().toString());
        span.tag("order.total", order.getTotal().toString());
        
        // Business logic
        
    } finally {
        span.end();
    }
}
```

### Interview Answer
> "I use structured logging with Logback — JSON format in production for ELK ingestion. Every log line includes traceId from Micrometer Tracing — allows end-to-end request tracking across services. Log entry/exit for controllers, business events like 'order placed', external API calls with duration, and exceptions with full context. Never log passwords, credit cards, or PII. Use MDC to enrich logs with userId, orderId contextually.
>
> For monitoring, Spring Boot Actuator exposes `/actuator/health` for K8s probes and `/actuator/prometheus` for Prometheus scraping. Custom metrics track business KPIs: orders created, payment failures, circuit breaker states. Grafana dashboards visualize request rate, p95 latency, error rate, heap usage, connection pool usage.
>
> In production, alerts fire on: error rate > 1%, p95 latency > 1s, heap usage > 85%, circuit breaker OPEN state. All logs centralized in ELK, searchable by traceId — when user reports an issue, I search their traceId and see the complete request flow across all services in seconds."

---

## Q104. Mockito Basics for Unit Testing

### Service Layer Testing

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepo;
    
    @Mock
    private PaymentService paymentService;
    
    @Mock
    private InventoryService inventoryService;
    
    @InjectMocks  // Injects mocks into OrderService constructor
    private OrderService orderService;
    
    @Test
    void shouldCreateOrderSuccessfully() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(1L, 2, "user@example.com");
        Order order = new Order(1L, "PENDING", BigDecimal.valueOf(100));
        
        when(inventoryService.isAvailable(1L, 2)).thenReturn(true);
        when(orderRepo.save(any(Order.class))).thenReturn(order);
        
        // When
        OrderDto result = orderService.createOrder(request);
        
        // Then
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getStatus()).isEqualTo("PENDING");
        
        // Verify interactions
        verify(inventoryService).isAvailable(1L, 2);
        verify(orderRepo).save(any(Order.class));
        verify(paymentService, never()).processPayment(any());  // Not called at this stage
    }
    
    @Test
    void shouldThrowExceptionWhenInventoryUnavailable() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(1L, 2, "user@example.com");
        when(inventoryService.isAvailable(1L, 2)).thenReturn(false);
        
        // When / Then
        assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(OutOfStockException.class)
                .hasMessage("Product 1 is out of stock");
        
        verify(inventoryService).isAvailable(1L, 2);
        verify(orderRepo, never()).save(any());  // Should not save if validation fails
    }
    
    @Test
    void shouldHandlePaymentFailure() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(1L, 2, "user@example.com");
        when(inventoryService.isAvailable(1L, 2)).thenReturn(true);
        when(paymentService.charge(any()))
                .thenThrow(new PaymentFailedException("Card declined"));
        
        // When / Then
        assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(PaymentFailedException.class);
        
        verify(paymentService).charge(any());
    }
}
```

### Controller Layer Testing (MockMvc)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean  // MockBean = Spring Boot's @Mock for integration with context
    private OrderService orderService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void shouldGetOrderSuccessfully() throws Exception {
        // Given
        OrderDto order = new OrderDto(1L, "COMPLETED", BigDecimal.valueOf(100));
        when(orderService.getOrder(1L)).thenReturn(order);
        
        // When / Then
        mockMvc.perform(get("/api/orders/1")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.status").value("COMPLETED"))
                .andExpect(jsonPath("$.total").value(100));
        
        verify(orderService).getOrder(1L);
    }
    
    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        // Given
        when(orderService.getOrder(999L))
                .thenThrow(new EntityNotFoundException("Order", 999L));
        
        // When / Then
        mockMvc.perform(get("/api/orders/999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.error").value("Not Found"))
                .andExpect(jsonPath("$.message").value("Order with id 999 not found"));
    }
    
    @Test
    void shouldCreateOrderWithValidRequest() throws Exception {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(1L, 2, "user@example.com");
        OrderDto created = new OrderDto(1L, "PENDING", BigDecimal.valueOf(100));
        
        when(orderService.createOrder(any(CreateOrderRequest.class))).thenReturn(created);
        
        // When / Then
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.status").value("PENDING"));
    }
    
    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        // Given
        CreateOrderRequest invalid = new CreateOrderRequest(null, 0, "invalid-email");
        
        // When / Then
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(invalid)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.fieldErrors.productId").exists())
                .andExpect(jsonPath("$.fieldErrors.quantity").exists())
                .andExpect(jsonPath("$.fieldErrors.email").exists());
    }
}
```

### Common Mockito Patterns

```java
// 1. Argument matching
when(userRepo.findById(1L)).thenReturn(Optional.of(user));
when(userRepo.findById(anyLong())).thenReturn(Optional.of(user));
when(userRepo.findByEmail(eq("test@example.com"))).thenReturn(user);
when(orderRepo.save(argThat(order -> order.getTotal().compareTo(BigDecimal.ZERO) > 0)))
        .thenReturn(savedOrder);

// 2. Return different values on successive calls
when(service.getValue())
        .thenReturn(1)      // first call
        .thenReturn(2)      // second call
        .thenThrow(new RuntimeException());  // third call

// 3. Verify call count
verify(userRepo).save(any());           // called once
verify(userRepo, times(2)).findById(anyLong());
verify(userRepo, never()).deleteById(anyLong());
verify(userRepo, atLeastOnce()).save(any());

// 4. Capture arguments
ArgumentCaptor<Order> orderCaptor = ArgumentCaptor.forClass(Order.class);
verify(orderRepo).save(orderCaptor.capture());
Order captured Order = orderCaptor.getValue();
assertThat(capturedOrder.getStatus()).isEqualTo("PENDING");

// 5. Void method stubbing
doThrow(new RuntimeException("Error")).when(service).voidMethod();
doNothing().when(service).voidMethod();

// 6. Spy (partial mock — real object with some methods stubbed)
OrderService spyService = spy(new OrderService(orderRepo, paymentService));
doReturn(order).when(spyService).expensiveMethod();  // stub this method
spyService.otherMethod();  // calls real method
```

### Interview Answer
> "I use Mockito for unit testing service layers — mock all dependencies with @Mock, inject them with @InjectMocks. Stub return values with `when().thenReturn()`, verify interactions with `verify()`. For controllers, @WebMvcTest with MockMvc — test HTTP layer independently from services by mocking service layer with @MockBean.
>
> Common patterns: argument matchers (`any()`, `eq()`), verify call count (`times()`, `never()`), argument captors to assert on saved entities, and `doThrow()` for void methods. Test both happy path and error scenarios — ensure exceptions are caught and handled correctly.
>
> Unit tests should be fast (no DB, no network) — mock all external dependencies. Integration tests with @SpringBootTest for end-to-end flows."

---

## Q105. API Optimization Strategies

### 1. Caching
```java
@Service
public class ProductService {
    
    // Application-level cache
    @Cacheable(value = "products", key = "#id")
    public ProductDto getProduct(Long id) {
        // Only executed if not in cache
        return productRepo.findById(id).map(this::toDto).orElseThrow();
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void updateProduct(Long id, UpdateProductRequest req) {
        // Evict from cache on update
    }
    
    @CachePut(value = "products", key = "#result.id")
    public ProductDto createProduct(CreateProductRequest req) {
        // Save and add to cache
        return toDto(productRepo.save(toEntity(req)));
    }
}

// Redis configuration for distributed cache
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
}
```

### 2. Pagination (Avoid Loading Entire Result Sets)
```java
@GetMapping("/api/orders")
public Page<OrderDto> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String sortDir) {
    
    Sort sort = Sort.by(Sort.Direction.fromString(sortDir), sortBy);
    Pageable pageable = PageRequest.of(page, size, sort);
    
    return orderRepo.findAll(pageable).map(this::toDto);
}

// Response includes pagination metadata
/*
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 1000,
  "totalPages": 50
}
*/
```

### 3. N+1 Query Problem Mitigation
```java
// ❌ N+1 Problem
@GetMapping("/orders")
public List<OrderDto> getOrders() {
    List<Order> orders = orderRepo.findAll();  // 1 query
    return orders.stream()
            .map(order -> {
                User user = userRepo.findById(order.getUserId()).get();  // N queries!
                return new OrderDto(order, user.getName());
            })
            .collect(Collectors.toList());
}

// ✅ Solution 1: Join Fetch
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") String status);

// ✅ Solution 2: DTO Projection
@Query("SELECT new com.app.dto.OrderDto(o.id, o.total, u.name) " +
       "FROM Order o JOIN o.user u WHERE o.status = :status")
List<OrderDto> findOrderDtosByStatus(@Param("status") String status);

// ✅ Solution 3: EntityGraph
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findAll();
```

### 4. Async Processing (Non-Blocking)
```java
@Service
public class NotificationService {
    
    @Async  // Runs in separate thread pool
    public void sendOrderConfirmation(Long orderId) {
        // Email sending doesn't block controller thread
        emailService.send(orderId);
    }
}

// Enable async
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### 5. Response Compression
```yaml
# application.yml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
    min-response-size: 1024  # Only compress responses > 1KB
```

### 6. Connection Pooling
```yaml
# HikariCP (default in Spring Boot)
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # Max connections
      minimum-idle: 5                # Min idle connections
      connection-timeout: 30000      # 30s timeout
      idle-timeout: 600000           # 10min idle before closing
      max-lifetime: 1800000          # 30min max connection lifetime
```

### 7. Database Indexing
```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_user_id", columnList = "user_id"),
    @Index(name = "idx_status_created", columnList = "status,created_at")
})
public class Order {
    // Common queries:
    // WHERE user_id = ?
    // WHERE status = ? ORDER BY created_at DESC
}
```

### 8. Filtering/Projection to Avoid Overfetching
```java
// ❌ Overfetching — returns all fields
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepo.findById(id).orElseThrow();
    // Returns: id, name, email, address, phone, passwordHash, createdAt, updatedAt...
}

// ✅ Projection — return only needed fields
@GetMapping("/users/{id}")
public UserSummaryDto getUser(@PathVariable Long id) {
    return userRepo.findUserSummaryById(id);  // DTO projection
    // Returns: id, name, email only
}

interface UserSummaryDto {
    Long getId();
    String getName();
    String getEmail();
}
```

### 9. Rate Limiting
```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    private final RateLimiter rateLimiter = RateLimiter.create(100.0);  // 100 req/sec
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler) {
        if (!rateLimiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            return false;
        }
        return true;
    }
}
```

### 10. Observability (Metrics + Alerts)
```java
@Service
public class OrderService {
    private final Timer orderProcessingTimer;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.orderProcessingTimer = Timer.builder("order.processing.time")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(meterRegistry);
    }
    
    public Order createOrder(CreateOrderRequest req) {
        return orderProcessingTimer.record(() -> doCreateOrder(req));
    }
}
```

### Interview Answer
> "API optimization is multifaceted. **Caching** — Redis for frequently accessed data (product catalog), with TTL based on update frequency. **Pagination** — never load entire result sets, default 20 items per page. **N+1 queries** — use JOIN FETCH or DTO projections, never lazy-load in loops.
>
> **Async processing** — email sends, notifications run in separate thread pools via @Async, don't block request threads. **Response compression** — GZIP for responses > 1KB reduces bandwidth 70%. **Connection pooling** — HikariCP with 20 max connections prevents connection exhaustion.
>
> **Database indexing** — index all foreign keys and common WHERE/ORDER BY columns. **Filtering/projection** — DTO projections return only needed fields, avoid overfetching. **Rate limiting** — 100 req/sec per user prevents abuse. **Observability** — p95 latency, error rate, cache hit rate tracked in Prometheus, alerts fire on degradation."

---

## Q106. Rate Limiting vs Throttling

### Definitions

**Rate Limiting:** Hard limit — reject requests exceeding threshold. Returns 429 Too Many Requests.

**Throttling:** Soft limit — slow down (delay) requests exceeding threshold. Still processes all.

### Rate Limiting Implementation

```java
// Guava RateLimiter (in-memory, per-instance)
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    private final LoadingCache<String, RateLimiter> limiters;
    
    public RateLimitInterceptor() {
        this.limiters = CacheBuilder.newBuilder()
                .expireAfterAccess(1, TimeUnit.HOURS)
                .build(new CacheLoader<>() {
                    @Override
                    public RateLimiter load(String key) {
                        return RateLimiter.create(10.0);  // 10 requests/sec per user
                    }
                });
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler) throws Exception {
        String userId = extractUserId(request);  // From JWT
        RateLimiter limiter = limiters.get(userId);
        
        if (!limiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("X-RateLimit-Limit", "10");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setHeader("Retry-After", "1");  // seconds
            return false;
        }
        return true;
    }
}
```

### Distributed Rate Limiting (Redis)
```java
@Component
public class RedisRateLimiter {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean isAllowed(String userId, int maxRequests, Duration window) {
        String key = "rate_limit:" + userId;
        Long current = redisTemplate.opsForValue().increment(key);
        
        if (current == 1) {
            redisTemplate.expire(key, window);  // Set TTL on first request
        }
        
        return current <= maxRequests;
    }
}

// Usage
if (!rateLimiter.isAllowed(userId, 100, Duration.ofMinutes(1))) {
    throw new RateLimitExceededException("100 requests per minute exceeded");
}
```

### Sliding Window Rate Limit (More Accurate)
```java
public boolean isAllowedSlidingWindow(String userId, int maxRequests, Duration window) {
    String key = "rate_limit:" + userId;
    long now = System.currentTimeMillis();
    long windowStart = now - window.toMillis();
    
    // Remove old entries
    redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);
    
    // Count requests in current window
    Long count = redisTemplate.opsForZSet().zCard(key);
    
    if (count < maxRequests) {
        redisTemplate.opsForZSet().add(key, UUID.randomUUID().toString(), now);
        redisTemplate.expire(key, window);
        return true;
    }
    
    return false;
}
```

### API Gateway Rate Limiting (Spring Cloud Gateway)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter:
                  replenishRate: 10      # tokens added per second
                  burstCapacity: 20      # max tokens in bucket
                key-resolver: "#{@userKeyResolver}"  # Extract user ID
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id")
    );
}
```

### Where to Implement

| Layer | Use Case | Tool |
|---|---|---|
| **API Gateway** | Global limits (all services) | Spring Cloud Gateway, Kong, Nginx |
| **Application** | Per-endpoint limits | Guava RateLimiter, Bucket4j, @RateLimiter |
| **Database** | Query rate limits | DB-level throttling (rare) |

### Interview Answer
> "Rate limiting hard-rejects requests exceeding threshold — returns 429. Throttling delays but still processes. I implement rate limiting at API Gateway level for global limits (1000 req/min per IP) using Spring Cloud Gateway with Redis. At application level, I use Bucket4j or Guava RateLimiter for per-user limits (100 req/min per user).
>
> Distributed systems need Redis-based rate limiting — in-memory limiters only work per-instance. Sliding window approach is more accurate than fixed window — prevents burst at window boundaries. Always include headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` to inform clients."

---

## Q107. Overfetching vs Underfetching

### Definitions

**Overfetching:** API returns more data than client needs. Wastes bandwidth, slows response.

**Underfetching:** API returns less data than client needs. Client must make multiple requests.

### Examples

```java
// ❌ Overfetching
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepo.findById(id).orElseThrow();
}
// Returns: id, name, email, passwordHash, address, phone, profilePicture (base64), 
//          preferences (JSON), createdAt, updatedAt, lastLoginAt...
// Mobile app only needs: id, name, profilePicture URL
// Wastes 90% of bandwidth

// ❌ Underfetching
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    return new OrderDto(order.getId(), order.getTotal());
    // Returns: id, total
}
// Client also needs: user name, product names, shipping address
// Client must call: GET /users/{userId}, GET /products/{productId}...
// 5 requests instead of 1 — high latency
```

### Solutions

#### 1. DTO Projections
```java
// Different DTOs for different use cases
public record UserSummaryDto(Long id, String name) {}
public record UserDetailDto(Long id, String name, String email, String phone, AddressDto address) {}

@GetMapping("/users/{id}/summary")
public UserSummaryDto getUserSummary(@PathVariable Long id) {
    return userRepo.findUserSummaryById(id);  // Only fetches id, name
}

@GetMapping("/users/{id}/detail")
public UserDetailDto getUserDetail(@PathVariable Long id) {
    return userRepo.findUserDetailById(id);  // Fetches all needed fields
}
```

#### 2. Filtering with `@JsonView`
```java
public class Views {
    public static class Summary {}
    public static class Detail extends Summary {}
}

@Entity
public class User {
    @JsonView(Views.Summary.class)
    private Long id;
    
    @JsonView(Views.Summary.class)
    private String name;
    
    @JsonView(Views.Detail.class)
    private String email;
    
    @JsonView(Views.Detail.class)
    private String phone;
}

@GetMapping("/users/{id}")
@JsonView(Views.Summary.class)  // Only includes fields marked with Summary
public User getUser(@PathVariable Long id) {
    return userRepo.findById(id).orElseThrow();
}
```

#### 3. Query Parameter Filtering (Field Selection)
```java
@GetMapping("/users/{id}")
public Map<String, Object> getUser(
        @PathVariable Long id,
        @RequestParam(required = false) List<String> fields) {
    
    User user = userRepo.findById(id).orElseThrow();
    
    if (fields == null || fields.isEmpty()) {
        return objectMapper.convertValue(user, Map.class);  // All fields
    }
    
    // Return only requested fields
    Map<String, Object> result = new HashMap<>();
    if (fields.contains("id")) result.put("id", user.getId());
    if (fields.contains("name")) result.put("name", user.getName());
    if (fields.contains("email")) result.put("email", user.getEmail());
    
    return result;
}

// Usage: GET /users/1?fields=id,name,email
```

#### 4. GraphQL (Ultimate Solution)
```graphql
# Client specifies exactly what it needs
query {
  user(id: 1) {
    id
    name
    orders {
      id
      total
      items {
        product {
          name
        }
      }
    }
  }
}

# Server returns ONLY what was requested
```

```java
// Spring Boot + GraphQL
@Controller
public class UserGraphQLController {
    @QueryMapping
    public User user(@Argument Long id) {
        return userRepo.findById(id).orElseThrow();
    }
    
    @SchemaMapping(typeName = "User", field = "orders")
    public List<Order> orders(User user) {
        return orderRepo.findByUserId(user.getId());
    }
}
```

### Interview Answer
> "Overfetching returns more data than needed — wastes bandwidth, slows mobile apps. Underfetching returns too little — client makes multiple round-trips, high latency. Solutions: DTO projections with separate summary/detail endpoints, `@JsonView` for field-level filtering, query parameter field selection (`?fields=id,name`), or GraphQL where client specifies exact requirements.
>
> In my project, REST endpoints have separate DTOs: `/users/{id}/summary` for lists (id, name only), `/users/{id}` for details (all fields). For complex aggregations, I use GraphQL — client fetches user+ orders + order items + products in one request with exactly the fields needed. Reduces API calls from 20 to 1 for dashboard loads."

---

## Q108. Connection Pooling

### Why It Matters

```
WITHOUT Connection Pooling:
  Request A → create DB connection (50ms) → execute query (10ms) → close connection
  Request B → create DB connection (50ms) → execute query (10ms) → close connection
  100 requests = 100 × (50ms connection + 10ms query) = 6 seconds total

WITH Connection Pooling:
  Startup → create 10 connections, keep open
  Request A → borrow connection (1ms) → execute query (10ms) → return to pool
  Request B → borrow connection (1ms) → execute query (10ms) → return to pool
  100 requests = 100 × (1ms borrow + 10ms query) = 1.1 seconds total
  5.4x faster!
```

### HikariCP (Default in Spring Boot)

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
    hikari:
      maximum-pool-size: 20        # Max total connections
      minimum-idle: 5               # Min idle connections kept alive
      connection-timeout: 30000     # Max wait time for connection (30s)
      idle-timeout: 600000          # Max idle time before closing (10min)
      max-lifetime: 1800000         # Max connection lifetime (30min)
      
      # Leakage detection
      leak-detection-threshold: 60000  # Warn if connection held > 60s
      
      # Connection validation
      connection-test-query: SELECT 1
      validation-timeout: 5000
```

### How It Works
```
┌─────────────────────────────────────────────────┐
│          HikariCP Connection Pool              │
│                                                 │
│  Available:  [conn1] [conn2] [conn3] ...       │
│  In-Use:     [conn4] [conn5]                    │
│  Idle:       [conn6] [conn7]                    │
│                                                 │
│  Stats:                                         │
│    Total: 10    Active: 2    Idle: 8          │
│    Pending: 0   Max: 20                        │
└─────────────────────────────────────────────────┘

Request arrives:
1. Check if idle connection available
2. If yes → return immediately
3. If no + total < max → create new connection
4. If no + total == max → wait (up to connection-timeout)
5. After query → return connection to pool (don't close)
```

### Monitoring Connection Pool

```java
@Component
public class HikariMetrics {
    @Autowired
    private HikariDataSource hikariDataSource;
    
    @Scheduled(fixedRate = 10000)  // Every 10s
    public void logPoolStats() {
        HikariPoolMXBean pool = hikariDataSource.getHikariPoolMXBean();
        log.info("Pool stats: total={}, active={}, idle={}, waiting={}",
                pool.getTotalConnections(),
                pool.getActiveConnections(),
                pool.getIdleConnections(),
                pool.getThreadsAwaitingConnection());
    }
}
```

```yaml
# Prometheus metrics (via Actuator)
management:
  metrics:
    enable:
      hikaricp: true

# Metrics exposed:
# hikaricp.connections.active
# hikaricp.connections.idle
# hikaricp.connections.pending
# hikaricp.connections.timeout.total
```

### Connection Pool Sizing Formula

```
Formula:
  connections = (core_count × 2) + effective_spindle_count

For typical web app (no heavy disk I/O):
  connections = (CPU_cores × 2)
  
  4-core machine → 8 connections
  8-core machine → 16 connections

Cloud database (AWS RDS):
  connections = min(20 × instances, RDS max_connections / 2)
  
  3 instances × 20 = 60 connections total
  RDS max_connections = 200 → use 100 max
```

### Common Issues

```java
// ❌ Connection leak — never returned to pool
public User getUser(Long id) {
    Connection conn = dataSource.getConnection();
    // ... use connection
    // FORGOT: conn.close()
    // Connection stuck in "active" state forever
}

// ✅ Try-with-resources — auto-closes
public User getUser(Long id) {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
        stmt.setLong(1, id);
        ResultSet rs = stmt.executeQuery();
        // ...
    }  // Connection automatically returned to pool
}

// ✅ Spring JdbcTemplate/JPA — handles pooling automatically
@Autowired
private JdbcTemplate jdbcTemplate;

public User getUser(Long id) {
    return jdbcTemplate.queryForObject(
        "SELECT * FROM users WHERE id = ?",
        new Object[]{id},
        new BeanPropertyRowMapper<>(User.class)
    );
}  // Connection management handled by Spring
```

### Interview Answer
> "Connection pooling reuses DB connections instead of creating new ones per request — creating a connection takes 50ms, borrowing from pool takes 1ms. HikariCP is default in Spring Boot and fastest available.
>
> I configure maximum-pool-size based on CPU cores (typically 2× cores). For 4-core machine: 8 connections. For cloud DBs, ensure all app instances combined don't exceed DB max_connections limit. Set connection-timeout to 30s — fail fast if pool exhausted. Enable leak-detection-threshold (60s) to catch connections never returned.
>
> Monitor with Actuator metrics: `hikaricp.connections.active`, `pending`, `timeout.total`. If `pending` stays > 0 or timeouts occur, either increase pool size or optimize slow queries. Connection leaks show as `active` connections that never decrease — find with leak detection warnings in logs."

---

## Quick Revision Card — Section 11

| Topic | Key Point |
|---|---|
| **Request Flow** | Client → Filter → DispatcherServlet → Interceptor → Controller → Service → Repository |
| **Filter** | Servlet-level, runs before Spring. Use for: encoding, CORS, logging. |
| **Interceptor** | Spring MVC-level, controller-aware. Use for: auth, rate limit, perf logging. |
| **@Component → Bean** | Component scan at startup → BeanDefinition → instantiate → DI → @PostConstruct → ready. |
| **RestTemplate vs WebClient** | RestTemplate = blocking, maintenance mode. WebClient = non-blocking, 100x better resource usage. |
| **Exception Handling** | @RestControllerAdvice + domain exceptions + RFC 7807 error model + appropriate status codes. |
| **REST Naming** | Nouns (not verbs), plural, lowercase, hyphens. Path params for IDs, query params for filters. |
| **Logging** | Structured JSON, traceId in every line, MDC for context. Never log passwords/PII. |
| **Monitoring** | Actuator + Prometheus + Grafana. Metrics: p95 latency, error rate, heap, connections. |
| **Mockito** | @Mock dependencies, @InjectMocks service. when().thenReturn() + verify(). @WebMvcTest for controllers. |
| **API Optimization** | Cache (Redis), pagination, N+1 fix (JOIN FETCH), async (@Async), compression, connection pooling, indexing. |
| **Rate Limiting** | Hard reject > threshold. Implement at Gateway (global) or app (per-user). Redis for distributed. |
| **Overfetching** | API returns too much. Fix: DTO projections, @JsonView, field selection, GraphQL. |
| **Connection Pooling** | HikariCP. Pool size = 2× CPU cores. Monitor: active, idle, pending, timeouts. Prevents connection exhaustion. |

---

## Additional Deep-Dive (Q97-Q108)

### Runtime Diagnostics Cheat Sheet

1. Request latency spikes:
- Check `http.server.requests` percentiles first (`p95`, `p99`).
- Correlate with DB metrics (`hikaricp.connections.pending`) and external API latency.
- Verify whether bottleneck is filter/interceptor overhead, serialization, or downstream IO.

2. Exception volume increase:
- Group by exception type and endpoint, not just total count.
- Confirm if failures are client-side (`4xx`) or server-side (`5xx`) before patching.
- Ensure global exception handler preserves root-cause context for internal logs while returning safe client responses.

3. Pool exhaustion:
- Inspect slow query logs and connection leak warnings before increasing pool size.
- If average query is fast but pending is high, scale horizontally or add read replicas.

### Senior Interview Follow-ups You Should Be Ready For

- Why `WebClient` can still be used in non-reactive apps: it gives better IO handling for remote calls even if your controller layer is MVC.
- Why filters and interceptors both exist: servlet-wide concerns belong in filters; handler-aware concerns belong in interceptors.
- Why observability must be designed, not added later: logs + metrics + traces together are required for real incident diagnosis.

### Real Project Usage

- Teams with stable SLAs usually define an SLO budget (for example, 99.9% under 300ms) and track burn rate from Prometheus alerts.
- For critical APIs, dashboards pair latency/error metrics with deployment markers so regressions are tied quickly to releases.

---

## Thread Dump Analysis & Deadlock Diagnosis — Reading Stack Traces Under Load

### Concept
When a production system hangs, hangs partially, or shows high CPU, thread dumps reveal exactly what each thread is doing: blocked on a lock, waiting for I/O, spinning, or runnable. A thread dump is a snapshot of all threads' call stacks at one moment. Experts read thread dumps to diagnose deadlocks, identify stuck threads, spot lock contention, and find infinite loops.

### Simple Explanation
Imagine investigating a traffic jam:
- **Call stack** = Following one car's path backward to see how it got stuck (came from highway → lane merged → stopped behind truck)
- **Thread dump** = Photo of all cars at once, showing exactly where each is and what it's waiting for
- **Deadlock** = Car A waiting for Car B to move, Car B waiting for Car A to move. Forever.

### How to Capture a Thread Dump

```bash
# 1. Find process ID
jps | grep app.jar
# Output: 12345 app.jar

# 2. Capture thread dump
jstack 12345 > dump.txt
# or multiple dumps to see if threads are stuck or moving
jstack 12345 > dump1.txt
sleep 5
jstack 12345 > dump2.txt

# 3. In production, often routed through actuator
curl http://localhost:8080/actuator/threaddump
# Returns JSON with all thread info
```

### Reading a Thread Dump — Normal Thread States

```
"http-nio-8080-exec-1" tid=0x00007f8f23 nid=0x4d68 runnable
java.lang.Thread.State: RUNNABLE
    at com.example.OrderService.calculateTotal(OrderService.java:125)
    at com.example.OrderController.createOrder(OrderController.java:45)
    - locked <0x00007f8f140a00f0> (a java.lang.Object)
    at java.lang.Thread.run(Thread.java:829)

Thread State Legend:
- RUNNABLE: Thread is executing code (good)
- WAITING: Thread called wait() in synchronized block, waiting for notify()
- BLOCKED: Thread waiting to acquire a lock (potential contention)
- TIMED_WAITING: wait(timeout) or Thread.sleep(), will wake automatically
- NEW: Thread created but not started
- TERMINATED: Thread exited
```

### Problem 1: Deadlock — Two Threads Waiting on Each Other

```
Thread 1: "OrderProcessor" (tid=100)
  java.lang.State: BLOCKED
  waiting to lock <0x00007f8f000c5640> (a OrderEntity)
  - locked <0x00007f8f000c5700> (a PaymentEntity)  ← Holds payment lock
  at com.example.OrderService.finalizeOrder(OrderService.java:200)
  at com.example.OrderProcessor.run(OrderProcessor.java:50)

Thread 2: "PaymentProcessor" (tid=101)
  java.lang.State: BLOCKED
  waiting to lock <0x00007f8f000c5700> (a PaymentEntity)
  - locked <0x00007f8f000c5640> (a OrderEntity)  ← Holds order lock
  at com.example.PaymentService.recordPayment(PaymentService.java:300)
  at com.example.PaymentProcessor.run(PaymentProcessor.java:75)

Deadlock Detected!
  Thread 1 holds Order, waits for Payment
  Thread 2 holds Payment, waits for Order
  ⇒ Neither releases, both frozen forever
```

#### ✅ Fix Deadlock: Lock Ordering

```java
// ❌ WRONG: Inconsistent lock order (can deadlock)
public class OrderService {
    public void finalizeOrder(Order order, Payment payment) {
        synchronized (order) {  // Lock ORDER first
            synchronized (payment) {  // Then PAYMENT
                // ...
            }
        }
    }
}

public class PaymentService {
    public void recordPayment(Payment payment, Order order) {
        synchronized (payment) {  // Lock PAYMENT first ← Different order!
            synchronized (order) {  // Then ORDER
                // ...
            }
        }
    }
}

// Thread 1: finalizeOrder locks ORDER, tries PAYMENT
// Thread 2: recordPayment locks PAYMENT, tries ORDER
// ⇒ Deadlock

// ✅ FIX: Always lock in same order
public class OrderService {
    public void finalizeOrder(Order order, Payment payment) {
        synchronized (order) {      // Always ORDER first
            synchronized (payment) {  // Always PAYMENT second
                // ...
            }
        }
    }
}

public class PaymentService {
    public void recordPayment(Payment payment, Order order) {
        synchronized (order) {      // ✅ Honor order: ORDER first
            synchronized (payment) {  // PAYMENT second
                // ...
            }
        }
    }
}
```

### Problem 2: Lock Contention — Many Threads Blocked Waiting for One Lock

```
Thread 1: "http-nio-8080-exec-1" (tid=1000)
  java.lang.State: BLOCKED
  waiting to lock <0x00007f8f000c5640> (a OrderCache)
  at com.example.OrderCache.getOrder(OrderCache.java:120)
  at com.example.OrderService.getOrderFromCache(OrderService.java:45)

Thread 2: "http-nio-8080-exec-2" (tid=1001)
  java.lang.State: BLOCKED
  waiting to lock <0x00007f8f000c5640> (a OrderCache)  ← Same lock
  at com.example.OrderCache.getOrder(OrderCache.java:120)

Thread 3: "http-nio-8080-exec-3" (tid=1002)
  java.lang.State: BLOCKED
  waiting to lock <0x00007f8f000c5640> (a OrderCache)  ← Same lock
  at com.example.OrderCache.getOrder(OrderCache.java:120)

... (100 more threads blocked on same lock)

Thread 98: "http-nio-8080-exec-98" (tid=1097)
  java.lang.State: RUNNABLE (Holds lock)
  - locked <0x00007f8f000c5640> (a OrderCache)
  at com.example.OrderCache.getOrder(OrderCache.java:121)  ← Computing
  Reading from slow database... Thread.sleep(2000)         ← SLOW!

Cause:
One thread holds OrderCache lock and is doing slow work (DB query).
All 100 other threads wait for the same lock.
Throughput: limited by speed of single thread, not by parallel capacity.
```

#### ✅ Fix: Lock-Free Collections or Fine-Grained Locks

```java
// ❌ WRONG: One big lock on entire cache
@Service
public class OrderCache {
    private Map<Long, Order> cache = new HashMap<>();
    
    public synchronized Order getOrder(Long id) {  // All IDs compete for same lock
        Order cached = cache.get(id);
        if (cached == null) {
            cached = loadFromDB(id);  // SLOW operation while holding lock
            cache.put(id, cached);
        }
        return cached;
    }
}

// ✅ FIX: ConcurrentHashMap + release lock for slow operations
@Service
public class OrderCache {
    private final ConcurrentHashMap<Long, Order> cache = new ConcurrentHashMap<>();
    
    public Order getOrder(Long id) {  // No global lock
        return cache.computeIfAbsent(id, key -> loadFromDB(key));  // Thread-safe compute
    }
    
    private Order loadFromDB(Long id) {
        // Slow operation, not holding cache lock
        return repository.findById(id).orElse(null);
    }
}

// Result:
// All 100 threads can compute in parallel
// Each thread computes its own order independently
// Throughput improved 50-100x
```

### Problem 3: Infinite Loop or Spinning Thread

```
Thread 1: "ProcessorThread-01" (tid=500)
  java.lang.State: RUNNABLE (High CPU!)
  at com.example.BatchProcessor.process(BatchProcessor.java:200)
  at com.example.BatchProcessor.process(BatchProcessor.java:200)
  at com.example.BatchProcessor.process(BatchProcessor.java:200)  ← Repeated!
  at com.example.BatchProcessor.process(BatchProcessor.java:198)
  at java.lang.Thread.run(Thread.java:829)

Analysis:
Same line 200 appears multiple times in stack (recursive call loop).
Thread keeps calling same method over and over.
CPU usage: 100% (one core burning).

Dump at T0: Line 200 from Thread.java
Dump at T0+1s: Line 200 from Thread.java (same location)
⇒ Thread hasn't moved in 1 second = stuck in loop or infinite recursion
```

#### ✅ Fix: Add Timeout or Exit Condition

```java
// ❌ WRONG: Infinite loop
private void process() {
    while (true) {  // Infinite loop!
        Item item = queue.poll();
        if (item == null) {
            continue;  // Spin-wait, no break!
        }
        processItem(item);
    }
}

// ✅ CORRECT: Add timeout
private void process() {
    while (true) {
        Item item = queue.poll(100, TimeUnit.MILLISECONDS);  // Poll with timeout
        if (item == null) {
            continue;  // Brief sleep, then retry
        }
        processItem(item);
    }
}

// Or break on condition:
private void process() throws InterruptedException {
    while (!Thread.currentThread().isInterrupted()) {
        Item item = queue.poll(100, TimeUnit.MILLISECONDS);
        if (item == null) {
            continue;
        }
        processItem(item);
    }
}
```

### Problem 4: Thread Waiting on I/O, Never Returns

```
Thread 1: "RemoteServiceCaller-01" (tid=600)
  java.lang.State: WAITING
  at java.net.SocketInputStream.read(SocketInputStream.java:-1)  ← Blocked on socket read
  at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
  at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
  at ... (HTTP client waiting for remote service response)
  at com.example.ExternalApiClient.call(ExternalApiClient.java:150)
  at com.example.OrderService.fetchPaymentStatus(OrderService.java:75)
  at com.example.OrderProcessor.run(OrderProcessor.java:40)

Analysis:
Thread blocked reading from remote socket (waiting for response).
If remote service is down or slow, thread waits indefinitely.
No timeout set.

With 100 request threads:
- Remote service responds in 5 second
- Thread blocked 5 seconds
- All 100 threads eventually block
- Application becomes unresponsive (all threads stuck)
```

#### ✅ Fix: Add Socket Timeout

```java
// ❌ WRONG: No timeout
@Service
public class ExternalApiClient {
    private RestTemplate restTemplate;
    
    public PaymentStatus checkPayment(String id) {
        return restTemplate.getForObject(
            "http://payment-service:8080/status/" + id,
            PaymentStatus.class
            // No timeout!
        );
    }
}

// ✅ CORRECT: Add timeout
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);    // 5s connect
        factory.setReadTimeout(5000);       // 5s read
        return new RestTemplate(factory);
    }
}

@Service
public class ExternalApiClient {
    @Autowired
    private RestTemplate restTemplate;
    
    public PaymentStatus checkPayment(String id) {
        try {
            return restTemplate.getForObject(
                "http://payment-service:8080/status/" + id,
                PaymentStatus.class
                // Now has 5s timeout
            );
        } catch (ResourceAccessException e) {
            // Handle timeout
            return PaymentStatus.UNKNOWN;
        }
    }
}
```

### Production Diagnostic Workflow

```bash
# Step 1: Capture baseline (system working well)
jstack [PID] > baseline.txt

# Step 2: Wait 30 seconds
sleep 30

# Step 3: Capture under load/hang
jstack [PID] > problem.txt

# Step 4: Compare
diff baseline.txt problem.txt

# Look for:
# - NEW threads in problem.txt (hung requests not releasing)
# - BLOCKED count increased
# - Same thread in BLOCKED state in both dumps (stuck)

# Step 5: Search for "BLOCKED" in problem.txt
grep -A 10 'java.lang.Thread.State: BLOCKED' problem.txt

# Step 6: Find what lock they're blocked on
# Each BLOCKED entry shows: "waiting to lock <0xaddress>"
# Search for threads that HOLD that lock: "-locked <0xaddress>"

# Step 7: Look at that thread's call stack
# Is it doing slow work? DB query? Remote call?
```

### Interview Answer

> "Thread dumps are X-rays of production. They show every thread's exact location in code and what it's waiting for.
>
> When production hangs or is slow, I capture multiple thread dumps (T0, T0+5s, T0+10s). If a thread is in the same place in all 3, it's stuck. If threads are in different places, they're moving (not stuck).
>
> Deadlock detection: Look for threads in BLOCKED state waiting on locks. Trace which locks they hold and which they wait for. Circular wait = deadlock. Fix: ensure all code acquires locks in the same order.
>
> Lock contention: Many threads BLOCKED on same lock. One thread RUNNABLE holding the lock and doing slow work. Fix: split the lock (ConcurrentHashMap), or move slow work outside sync block.
>
> Infinite loops: Thread spinning CPU at 100%, call stack shows same line repeatedly in multiple dumps. Fix: add exit condition, timeout, or yield.
>
> Hung I/O: Thread WAITING in SocketInputStream.read(), stuck because remote service is down. Fix: add socket timeout on HTTP client.
>
> Real scenario: High-memory order service, latency spike to 30 seconds. Thread dumps showed 200 threads BLOCKED on OrderCache lock. One thread holding lock, doing slow DB query (reading from full table scan). Root cause: missing index. Added index, lock held < 100ms, throughput returned to normal. Solved with thread dump analysis."

**Follow-up likely:** "How do you find deadlocks programmatically?" → ThreadMXBean.findMonitorDeadlockedThreads() in code, or use jvisualvm GUI tool.

---

## Quick Revision Card — Thread Dump Analysis

| State | Meaning | Action |
|-------|---------|--------|
| **RUNNABLE** | Thread executing | Normal; if 100% CPU check for spin loops |
| **BLOCKED** | Waiting for lock | Check what lock; find holder; why is holder slow? |
| **WAITING** | Called wait(), waiting for notify() | Check if notify() is being called |
| **TIMED_WAITING** | wait(timeout) or sleep() | Normal; thread will auto-wake |
| **Deadlock signature** | Circular lock wait | Fix: enforce consistent lock ordering |
| **Lock contention** | Many BLOCKED on same lock | One RUNNABLE holds lock and is slow |
| **Infinite loop** | Same line in multiple dumps | Add timeout, break condition, yield |
| **Socket wait** | SocketInputStream.read() | Add timeout on HTTP/database client |

---

## Jackson Deep-Dive - Serialization, Deserialization, and API Safety

### Concept
Jackson is the default JSON engine in Spring Boot. It converts request JSON to Java objects (deserialization) and Java objects to response JSON (serialization). In interviews, the real depth is not basic mapping, but controlling payload shape, version tolerance, date/time handling, and preventing security or compatibility bugs.

### Simple Explanation
Think of Jackson as an airport translator:
- Incoming passenger language (JSON) is translated to your internal language (Java DTO/entity).
- Outgoing announcements (Java objects) are translated back to JSON for clients.
- If translation rules are sloppy, the wrong traveler gets on the wrong flight: fields disappear, dates break, unknown fields crash requests, or sensitive fields leak.

### How It Works in Spring Boot

1. `MappingJackson2HttpMessageConverter` is auto-registered by Spring Boot.
2. Incoming body in controller (`@RequestBody`) is deserialized by `ObjectMapper`.
3. Outgoing return object (`@RestController`) is serialized by same mapper.
4. Customization points:
- Global: `spring.jackson.*` properties or `Jackson2ObjectMapperBuilderCustomizer`.
- Per-field/per-class: annotations like `@JsonProperty`, `@JsonIgnore`, `@JsonInclude`, `@JsonFormat`, `@JsonAlias`.
- Advanced: custom serializers/deserializers, modules (`JavaTimeModule`), mixins.

### Core Annotations You Should Know

| Annotation | Purpose | Typical Interview Use |
|---|---|---|
| `@JsonProperty` | Rename field in JSON | Backward-compatible field rename |
| `@JsonAlias` | Accept old field names on input | API version migration |
| `@JsonIgnore` | Exclude field from JSON | Hide internal/sensitive fields |
| `@JsonInclude(NON_NULL)` | Skip nulls | Smaller responses |
| `@JsonFormat` | Date/time format control | Consistent API date output |
| `@JsonView` | Field-level views | Summary vs detail payloads |
| `@JsonCreator` | Constructor-based mapping | Immutable DTOs/records |
| `@JsonIgnoreProperties(ignoreUnknown = true)` | Tolerate extra fields | Consumer resilience |

### Code - Production Scenarios

#### 1) Input compatibility with renamed fields

```java
public record CustomerRequest(
    @JsonProperty("customerId")
    @JsonAlias({"clientId", "userId"})
    String customerId,

    @JsonProperty("fullName")
    String fullName
) {}
```

Why this matters:
- Old clients may still send `clientId`.
- New clients send `customerId`.
- Service accepts both during migration without breaking.

#### 2) Prevent accidental data leakage

```java
public class UserResponse {
    private String id;
    private String email;

    @JsonIgnore
    private String passwordHash;

    @JsonIgnore
    private String internalRoleMapping;
}
```

Interview point:
- `@JsonIgnore` is not authorization.
- It only controls serialization shape.
- Security still needs access control and endpoint-level checks.

#### 3) Stable date/time handling with Java 21

```java
@Configuration
public class JacksonConfig {
    @Bean
    Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
        return builder -> builder
            .modules(new JavaTimeModule())
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .featuresToDisable(DeserializationFeature.ADJUST_DATES_TO_CONTEXT_TIME_ZONE);
    }
}

public record OrderResponse(
    String orderId,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ssXXX")
    OffsetDateTime createdAt
) {}
```

Why this matters:
- Avoid numeric timestamps unless explicitly required.
- Emit ISO-8601 consistently.
- Time-zone drift is a common production bug.

#### 4) Unknown fields strategy (strict vs tolerant)

```java
// Strict DTO for internal admin APIs
public record AdminConfigRequest(
    String feature,
    boolean enabled
) {}

// Tolerant DTO for public client APIs
@JsonIgnoreProperties(ignoreUnknown = true)
public record PublicWebhookEvent(
    String eventId,
    String type,
    JsonNode payload
) {}
```

Recommended approach:
- Internal trusted clients: fail fast on unknown fields.
- External/public clients: tolerate unknown fields to reduce breaking changes.

#### 5) Custom serializer/deserializer for domain values

```java
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toPlainString());
    }
}

public class MoneyDeserializer extends JsonDeserializer<BigDecimal> {
    @Override
    public BigDecimal deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        return new BigDecimal(p.getValueAsString()).setScale(2, RoundingMode.HALF_UP);
    }
}

public record PaymentDto(
    String paymentId,
    @JsonSerialize(using = MoneySerializer.class)
    @JsonDeserialize(using = MoneyDeserializer.class)
    BigDecimal amount
) {}
```

Use when:
- Currency format must be exact.
- Client contracts require string decimal formatting.

### ObjectMapper Tuning Checklist (Production)

```yaml
spring:
  jackson:
    default-property-inclusion: non_null
    deserialization:
      fail-on-unknown-properties: false
    serialization:
      write-dates-as-timestamps: false
    time-zone: UTC
```

Practical defaults:
- `non_null` to reduce noisy payloads.
- `fail-on-unknown-properties=false` for external API tolerance.
- `UTC` for consistent cross-region behavior.

### Common Jackson Pitfalls

1. Lazy-loaded JPA entities serialized directly
- Causes N+1 or `LazyInitializationException`.
- Fix: map entities to DTOs in service layer.

2. Bidirectional relationships (`User -> Orders -> User`)
- Can recurse infinitely.
- Fix: DTO mapping preferred; alternatively `@JsonManagedReference/@JsonBackReference`.

3. Multiple ObjectMapper instances with different config
- Leads to inconsistent API behavior.
- Fix: centralize via Boot customization; avoid `new ObjectMapper()` in business code.

4. Swallowing parse errors
- Hides client contract bugs.
- Fix: return structured `400 Bad Request` with field-level details.

### Real Project Usage

Scenario: Mobile app API version migration.
- v1 field: `clientId`
- v2 field: `customerId`
- Implementation used `@JsonAlias({"clientId"})` on the v2 DTO.
- Service accepted both fields for 60 days, metrics tracked remaining v1 traffic.
- After adoption, alias was removed in a scheduled cleanup release.

Scenario: Payment timestamps were off by timezone.
- Some services emitted epoch millis, others ISO strings in local zones.
- Standardized all APIs to `OffsetDateTime` + ISO-8601 UTC output.
- Incident rate for reporting mismatches dropped significantly.

### Interview Answer

> "Jackson in Spring Boot is not just JSON conversion, it's contract control. I start with safe defaults: ISO date/time, UTC, null exclusion where useful, and unknown-field strategy based on API type. For public APIs I usually tolerate unknown fields to support forward compatibility. For internal admin APIs I keep strict validation.
>
> I avoid serializing JPA entities directly because lazy proxies and bidirectional relationships create runtime issues and recursion. DTO mapping gives stable contracts and avoids leaking internal model details.
>
> For versioning, I use `@JsonAlias` and `@JsonProperty` to migrate field names without breaking old clients, then remove aliases after telemetry confirms adoption. For domain precision like money, I add custom serializers/deserializers so rounding and formatting are consistent across services.
>
> In production, the biggest anti-pattern is creating ad-hoc `new ObjectMapper()` instances in random classes. That causes inconsistent behavior between controllers, jobs, and tests. I centralize mapper config using Boot customization so every JSON path follows the same rules.
>
> If asked about safety: Jackson annotations control shape, not authorization. `@JsonIgnore` prevents output but does not enforce security policy. Access control still belongs in security and service logic."

Follow-up likely: How do you handle backward compatibility for request/response changes without breaking clients?

---

## Quick Revision Card - Jackson

| Topic | Key Point |
|---|---|
| `ObjectMapper` | Keep one centralized config; avoid local ad-hoc mappers |
| `@JsonAlias` | Accept old field names during migration |
| `@JsonProperty` | Stable external naming independent of Java field names |
| `@JsonIgnore` | Hide fields in JSON, but not a security boundary |
| Date/Time | Use `JavaTimeModule` + ISO-8601 + UTC |
| Unknown fields | Public APIs tolerant; internal APIs strict |
| JPA entities | Prefer DTO mapping to avoid lazy/recursion issues |
| Money fields | Custom serializer/deserializer for deterministic format |

---

**End of Section 11 (with Thread Dump Analysis)**
